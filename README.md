# home-dns-architecture

Project Objectives:
* Build a solution with security level higher than sole DNS over TLS but weaker than routing every traffic data to VPN Servers
* Do public DNS query by getting rid of local ISP DNS tracking, and choose to trust VPN Provider out of a country instead
* Enable DNS blacklist for whitelist devices within local network
* Resolve internal Kubernetes services with internal domain name
* Make every components replacable, robust and stateless under Mircoservice architecture without mounting physical volume
* All the stuff run in a Kubernete cluster

![Architecture](https://dannypv.ddns.net/share/40dc6f01deb5c5c61f4515e8c0337b4d34f3bbe3ca4f7e376ae22aba)

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
ConfigMap: Load CNAME
```
  05-pihole-custom-cname.conf: |
    cname=a.lan,alias.lan
```
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
Deployment: import ConfigMap files  into database by different tricks of adding args
1. Create a enabling DNS filtering user group
2. Import whitelist
3. Import Adlist and make them all assigned to newly create group
4. Import Local DNS Records
5. Import clients which required to enable DNS filtering
```
          args: ["sqlite3 /etc/pihole/gravity.db \"INSERT OR REPLACE INTO \\`group\\` (id, enabled, name, description) VALUES (1, 1, 'Ad. Filter', 'From group.list');\" && \
                cat /etc/pihole/whitelist.txt | while read line; do pihole -w \"$line\" ; done && \
                cat /etc/pihole/whitelist-wildcard.txt | while read -r line; do pihole --white-wild \"$line\" ; done && \
                sqlite3 /etc/pihole/gravity.db \"UPDATE domainlist_by_group SET group_id=1 WHERE group_id=0;\" && \
                cat /etc/pihole/adlists.list | while read line; do sqlite3 /etc/pihole/gravity.db \"INSERT OR REPLACE INTO adlist (address, enabled) VALUES (trim('$line'), 1);\"; done && \
                sqlite3 /etc/pihole/gravity.db \"UPDATE adlist_by_group SET group_id=1 WHERE group_id=0;\" && \
                cat /etc/pihole/client.list | while read line; do sqlite3 /etc/pihole/gravity.db \"INSERT OR REPLACE INTO client (ip, comment) VALUES (trim('$line'), 'From client.list');\"; done && \
                sqlite3 /etc/pihole/gravity.db \"UPDATE client_by_group SET group_id=1 WHERE group_id=0;\" && \
                /s6-init"]
                
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
            - name: TZ
              value: Asia/Hong_kong
            - name: ETCD_URLS
              value: http://etcd.default:2379
```

CoreDNS's Corefile ConfigMap: read Ingress DNS from etcd for each query
* Set port 53 to main DNS with unbound upstream and set port 5353 to fail-over DNS with cloudflare DNS with HTTP over TLS
* When domain name match home.internal e.g. gitlab.home.internal, use DNS from etcd http://10.z.z.z:2379
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
            endpoint http://10.z.z.z:2379
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

##### Limitation
