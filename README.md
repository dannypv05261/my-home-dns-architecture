# home-dns-architecture

Project Objectives:
* Build a solution with security level higher than sole DNS over TLS but weaker than routing every traffic data to VPN Servers
* Do public DNS query by getting rid of local ISP DNS tracking, and choose to trust VPN Provider out of a country instead
* Enable DNS blacklist for specific devices within local network
* Resolve internal Kubernetes services with internal domain name
* Make every components robust and stateless under Mircoservice architecture

![Architecture](https://dannypv.ddns.net/share/40dc6f01deb5c5c61f4515e8c0337b4d34f3bbe3ca4f7e376ae22aba)

## 1. DNS Filter

## 2. Auto-register DNS from Kubernetes Ingress

## 3. Public DNS Resolver
