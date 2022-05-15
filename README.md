# Architecture of my home DNS

I use this setup starting from 2020 to now. I will keep updating it if I find something new. Please let me know in  [Discussions](https://github.com/dannypv05261/home-dns-architecture-tutorial/discussions/1) if you give any suggestion on my solution / improvement on config setting.

This repository is to provide the idea and the major config of my home DNS architecture with following objectives:
* Build a solution with security level higher than sole DNS over TLS but weaker than routing every traffic data to VPN Servers
* Do public DNS query by getting rid of local ISP, and choose to trust VPN provider.
  * i.e. I turn to trust my VPN provider to deliever our DNS request without descrpying the DNS content for monitoring and without keeping my browsing history
* Enable DNS blacklist for whitelist devices within local network
  * Only add my own devices to the list and you don't do support for your family memeber lol
* Resolve internal Kubernetes services with internal domain name
* Make every components replacable, standalone, robust and stateless under Mircoservice architecture without mounting physical volume
  * Let say you replace from Pi-hole to AdGuard Home, chaning another VPN provider, removing CoreDNS because you don't run K8S. We just need to change the upstream DNS servers.
* All the stuff run in a Kubernete cluster
  * You may also apply the same idea on docker-compose
  
![Architecture](https://dannypv.ddns.net/share/2b9cbf9eb2ef70fdffb3c27b2e006ae17cf5d4707bcd7165c90504aa)

Cluster IP address  | Service
--- | ---
10.e.e.e  | etcd
10.w.w.w  | CoreDNS
10.x.x.x  | Unbound 1
10.y.y.y  | Unbound 2

## 1. DNS Filter + non-Kubernetes Local DNS Resolver
##### Stateless Pi-Hole
ConfigMap: Query upstream DNS servers by sequence so that we can ask Pi-Hole to query CoreDNS first
```
  02-dnsmasq.conf: |
    # dns in reversed order only read the dns servers from resolv-file
    strict-order
    # don't cache negative result
    no-negcache
```

- - - -

ConfigMap: Load CNAME
```
  05-pihole-custom-cname.conf: |
    cname=a.lan,alias.lan
```

- - - -

As Pi-hole 4 uses SQLlite to store config files, we need to import config data from file by some tricks.
Here are some examples
ConfigMap:
* adlists.list: Adlist group management
* custom.list: Local DNS Records
* client.list: Client group management (We only do DNS filtering on specific clients)
* whitelist.txt: Whitelist management
* whitelist-wildcard.txt: Whitelist management
```
  adlists.list: |
    https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
  custom.list: |
    192.168.xxx.xxx alias.lan
  client.list: |
    192.168.xxx.xxx
    XX:XX:XX:XX:XX:XX
  whitelist.txt: |
    owncloud.com
  whitelist-wildcard.txt: |
    xxx.edu.hk
```

- - - -

Deployment: import ConfigMap files  into database by different tricks of adding args
1. Create a enabling DNS filtering user group
2. Import whitelist
3. Import Adlist and make them all assigned to newly create group
4. Import Local DNS Records
5. Import clients which required to enable DNS filtering
```
      containers:
        name: pihole
        # pi-hole
        - image: pihole/pihole:2022.04.3
          command: ["bin/sh", "-c"]
          args: ["sqlite3 /etc/pihole/gravity.db \"INSERT OR REPLACE INTO \\`group\\` (id, enabled, name, description) VALUES (1, 1, 'Ad. Filter', 'From group.list');\" && \
                cat /etc/pihole/whitelist.txt | while read line; do pihole -w \"$line\" ; done && \
                cat /etc/pihole/whitelist-wildcard.txt | while read -r line; do pihole --white-wild \"$line\" ; done && \
                sqlite3 /etc/pihole/gravity.db \"UPDATE domainlist_by_group SET group_id=1 WHERE group_id=0;\" && \
                cat /etc/pihole/adlists.list | while read line; do sqlite3 /etc/pihole/gravity.db \"INSERT OR REPLACE INTO adlist (address, enabled) VALUES (trim('$line'), 1);\"; done && \
                sqlite3 /etc/pihole/gravity.db \"UPDATE adlist_by_group SET group_id=1 WHERE group_id=0;\" && \
                cat /etc/pihole/client.list | while read line; do sqlite3 /etc/pihole/gravity.db \"INSERT OR REPLACE INTO client (ip, comment) VALUES (trim('$line'), 'From client.list');\"; done && \
                sqlite3 /etc/pihole/gravity.db \"UPDATE client_by_group SET group_id=1 WHERE group_id=0;\" && \
                /s6-init"]

          env:
            - name: PIHOLE_DNS_
              value: "10.w.w.w#53;10.x.x.x#53;10.y.y.y#53"
          
          ......

          volumeMounts:
            - mountPath: /etc/dnsmasq.d/02-dnsmasq.conf
              name: pihole-configmap-etc-dnsmasq-d
              subPath: 02-dnsmasq.conf
            - mountPath: /etc/dnsmasq.d/05-pihole-custom-cname.conf
              name: pihole-configmap-etc-dnsmasq-d
              subPath: 05-pihole-custom-cname.conf
            - mountPath: /etc/dnsservers.conf
              name: pihole-configmap-etc-dnsmasq-d
              subPath: dnsservers.conf
              
      volumes:
        - name: pihole-configmap-etc-dnsmasq-d
          configMap:
            defaultMode: 0644
            name: pihole-configmap-etc-dnsmasq-d
        - name: pihole-configmap-etc-pihole
          configMap:
            defaultMode: 0644
            name: pihole-configmap-etc-pihole
            
```

- - - -

##### Limitation
1. The import data trick will be broken when Pi-hole officals update the database structure
2. To update config, we need to update ConfigMap and restart Pi-hole containers

## 2. Kubernetes Ingress DNS Resolver
##### CoreDNS + external-dns + etcd: auto update Kubernetes Ingress DNS records by using etcd as a bridge between CoreDNS and external-dns

external-dns's deployment: detect ingress change and register domain names to etcd automatically
* Set DNS source to ingress
* Set provider to CoreDNS
* Add the internal network domain names to domain-filte
* Specify the etcd instance you used to store ingress domain name e.g. gitlab.home.internal
```
    spec:
      serviceAccountName: external-dns
      imagePullSecrets:
        - name: gcr-json-key
      containers:
        - name: external-dns
          image: k8s.gcr.io/external-dns/external-dns:v0.11.0
          args:
            - --source=ingress
            - --provider=coredns
            - --log-level=info
            - --domain-filter=home.internal
          env:
            - name: ETCD_URLS
              value: http://etcd.default:2379
...... 
```

- - - -

CoreDNS's Corefile ConfigMap: read Ingress DNS from etcd for each query
* Set port 53 to main DNS with unbound upstream and set port 5353 to fail-over DNS with cloudflare DNS with HTTP over TLS
* When domain name match home.internal e.g. gitlab.home.internal, use DNS from etcd http://10.e.e.e:2379
* When the record doesn't match home.internal, do DNS query from unbound ClusterIP and fail-over local DNS 10.x.x.x:53 10.y.y.y:53 127.0.0.1:5053 in sequence
```
 Corefile: |
    .:5053 {
      ready
      health
      # bind 127.0.0.1
      forward . tls://1.1.1.1 tls://1.0.0.1 {
        policy round_robin
        tls_servername cloudflare-dns.com
      }        
      cache 3600 {
        denial 0
      }
      reload
    }
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        hosts /etc/coredns/NodeHosts {
          fallthrough
        }

        etcd home.internal {
            stubzones
            path /skydns
            # cannot resolve by domain name (cannot reference to itself)
            endpoint http://10.e.e.e:2379
            fallthrough
        }

        # forward to localhost in case upstream network down
        forward . 10.x.x.x:53 10.y.y.y:53 127.0.0.1:5053 {
          policy sequential
        }

        cache 60 {
          denial 0
        }

        loop
        reload
        loadbalance
    }
```

- - - -

##### Limitation
The fail-over mechanism only works when the upstream instance is down, any response that is not a network error (REFUSED, NOTIMPL, SERVFAIL, etc) is taken as a healthy upstream. (From https://coredns.io/plugins/forward/)
```
        # forward to localhost in case upstream network down
        forward . 10.x.x.x:53 10.y.y.y:53 127.0.0.1:5053 {
          policy sequential
        }
```
## 3. Public DNS Resolver
##### VPN client + unbound
Deployment: VPN Sharing: with VPN client as a side car container, unbound container can also share the VPN interface by volume mount. You can use other VPN service. I use surfshark as an example personally.
```
      initContainers:
        - name: install
          image: busybox
          command:
            - mkdir
            - /dev/net
            - mknod
            - /dev/net/tun
            - c
            - "10"
            - "200"
            
      containers:
        - name: surfshark
          image: ilteoood/docker-surfshark
          imagePullPolicy: IfNotPresent
          securityContext:
            capabilities:
              add:
                - NET_ADMIN
          volumeMounts:
            - mountPath: /dev/net/tun
              name: dev-tun
          env:
            - name: SURFSHARK_COUNTRY
              value: jp
            - name: SURFSHARK_CITY
              value: tok
            - name: CONNECTION_TYPE
              value: udp
            - name: OPENVPN_OPTS
              value: "--pull-filter ignore block-outside-dns"
        ......
        - name: unbound          
          image: klutchell/unbound:1.15.0
        ......
      volumes:
        - name: dev-tun
          hostPath:
            path: /dev/net/tun
            type: CharDevice
        ......
```

- - - -

ConfigMap: set unbound config file
* set unbound query upstream to public DNS with DNS over TLS
* must set allow TCP when I used with surfshark. I personally recommend UDP only if your VPN client supports it
* allow downstream to query unbound
* if your VPN service doesn't route all the traffic to VPN tunnel, you need to set it manually with 'outgoing-interface' property
```
data:
  unbound.conf: |
    server:
      # The port number, default 53, on which the server responds to queries.
      port: 53
      
      tls-cert-bundle: /etc/ssl/certs/ca-certificates.crt
      forward-zone:
      name: "."
      forward-tls-upstream: yes
    
      # cloudflare
      forward-addr: 1.1.1.1@853
      forward-addr: 1.0.0.1@853
      #forward-addr: 2606:4700:4700::1111@853
      #forward-addr: 2606:4700:4700::1001@853
    
      # Listen on UDP but still issues queries upstream over UDP
      do-ip4: yes
      do-udp: yes
      do-tcp: yes
      do-ip6: no
      #  enable udp upstream even if do-udp is no
      udp-upstream-without-downstream: yes

      # listen on all interfaces
      interface: 0.0.0.0@53
      interface: ::0@53

      # answer queries from below subnets
      access-control: 192.168.x.0/24 allow
      access-control: 10.0.0.0/8 allow
      access-control: fd00::/8 allow
      access-control: fe80::/10 allow

      private-address: 192.168.x.0/24
      private-address: 10.0.0.0/8
      private-address: fd00::/8
      private-address: fe80::/10

      # query via VPN
      #outgoing-interface: 10.8.8.0/24
      #outgoing-interface: fe80:0000:0000:0000:0000:0000:0000:0000/16
      ......
```

- - - -

Deployment: set unbound container
* set dnsConfig to public DNS because we need to download config file from a domain name. We need to break the chicken and egg problem
* must set allow TCP when I used with surfshark. I personally recommend UDP only if your VPN client supports it
* add volumn mount to the ConfigMap of unbound above
```
    spec:
      dnsConfig:
        # need to connect to VPN with domain names
        nameservers:
          - 1.1.1.1
          - 1.0.0.1
    .....
        - name: unbound
          image: klutchell/unbound:1.15.0
          ports:
            - name: svc-53-udp
              containerPort: 53
              protocol: UDP
            - name: svc-53-tcp
              containerPort: 53
              protocol: TCP

          volumeMounts:
            - mountPath: /opt/unbound/etc/unbound/unbound.conf
              name: unbound-configmap-etc
              subPath: unbound.conf
            - mountPath: "/etc/unbound/ca-certificates.crt"
              name: tls-ca
              readOnly: true  
    ......
      volumes:
        - name: unbound-configmap-etc
          configMap:
            defaultMode: 0775
            name: unbound-configmap-etc
        - name: tls-ca
          hostPath:
            path: /etc/ssl/certs/ca-certificates.crt
    ......
```

- - - -

Let do DNS leak test in one of the client.
As our unbound connected to JP VPN Server and we use Cloudflare as public DNS. So, it is fine to leak to Cloudflare under recusive queries
![DNS leak](https://dannypv.ddns.net/share/b53e7f3d2819e8069ff10abc41c36431ab4b2548998380c03bb67ebc)

We can also make sure we have enabled DNS over TLS (DoT).
![DNS DOT](https://dannypv.ddns.net/share/c8f5e77d547f9b1b99c8a7351408eddc7e84ecbc28bbde4a84c94c73)
