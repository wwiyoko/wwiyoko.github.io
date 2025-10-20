---
title: Setup DNS over TLS with Quad9 on systemd & non-systemd
date: 2025-10-20
description: A guide on how to setup DNS over TLS with Quad9 DNS service on systemd and non-systemd.
---

Put simply, using DoT (DNS over TLS) is to encrypt DNS queries every time you
enter a website address in the search bar. Without DoT, DNS queries are sent in
plaintext, which means your ISP can read what addresses you are trying to browse
on the internet.

This article was meant as a guide to prevent internet censorship
from governments and unsecure DNS traffic, especially on Linux, which is using
systemd or a non-systemd.

You can use [Cloudflare `1.1.1.1`](https://one.one.one.one/dns/) as the DNS resolver,
but here I will write for [Quad9](https://quad9.net/). The steps are the same;
the only difference is to change the DNS address to `1.1.1.1` if you are going to
use Cloudflare.

## systemd

In Debian derivatives, you need to install the `bind9-dnsutils` and `systemd-resolved`
package. Then enable the services as root: `systemctl enable --now systemd-resolved`.

Edit `/etc/systemd/resolved.conf`:

```
[Resolve]
DNS=9.9.9.9
DNSOverTLS=yes
```

If you are using `NetworkManager`, you need to edit every connected network
with `nmtui`. Inside IPv4 configuration, set the method to automatic (DHCP)
and add `9.9.9.9` DNS server.

Restart the network as root: `systemctl restart systemd-resolved NetworkManager`

Then verify the connection: `dig +short txt proto.on.quad9.net`

If the response is `dot.`, then it is working. You can also visit
[on.quad9.net](https://on.quad9.net/) to check if the Quad9 DNS is working.

## non-systemd

In other non-systemd such as runit, OpenRC, and s6, we need two packages
to be installed:

- [`stubby`](https://dnsprivacy.org/dns_privacy_daemon_-_stubby/about_stubby/)
as a local DNS resolver that forwards the DNS queries to DoT servers or DNS providers.
- [`dnsmasq`]() as a DNS forwarder, and it used to cache queries. So the traffic
will go from `dnsmasq` to look up the cache; if it is not available, then it
will forward to `stubby`.

Both tools need to be enabled. For those of you who are using runit as an init
system, you can enable it by symlink the services as root:

```
ln -s /etc/sv/stubby /var/service/
ln -s /etc/sv/dnsmasq /var/service/
```

Edit `/etc/stubby/stubby.yml`:

```yaml
resolution_type: GETDNS_RESOLUTION_STUB
dns_transport_list:
  - GETDNS_TRANSPORT_TLS
tls_authentication: GETDNS_AUTHENTICATION_REQUIRED
tls_query_padding_blocksize: 128
edns_client_subnet_private : 1
round_robin_upstreams: 0
idle_timeout: 10000
listen_addresses:
  - 127.0.0.1@8053
  - 0::1@8053
upstream_recursive_servers:
  - address_data: 9.9.9.9
    tls_auth_name: "dns.quad9.net"
```

Edit `/etc/dnsmasq.conf`:

```
no-resolv
server=127.0.0.1#8053
server=::1#8053
listen-address=127.0.0.1,::1
bind-interfaces
no-hosts
cache-size=10000
```

Edit `/etc/resolv.conf`:

```
nameserver 127.0.0.1
nameserver ::1
```

And then, to prevent the network manager from changing the address `resolv.conf`,
enter the command as root:

```sh
chattr +i /etc/resolv.conf
```

Restart `stubby`, `dnsmasq`, and your network manager service with your
preferred init system. Then verify and check the connection with
`dig +short txt proto.on.quad9.net`.

If the response is `dot.`, then it is working. You can also visit
[on.quad9.net](https://on.quad9.net/) to check if the Quad9 DNS is working.

## References

- https://www.cloudflare.com/learning/dns/dns-over-tls/
- https://docs.quad9.net/Setup_Guides/Linux_and_BSD/Ubuntu_22.04_%28Encrypted%29/
- https://dnsprivacy.org/dns_privacy_daemon_-_stubby/
- https://s13h.xyz/posts/dot/
- https://forums.linuxmint.com/viewtopic.php?t=345236
