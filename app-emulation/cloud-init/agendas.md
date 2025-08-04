# Experimental Cloud-init Netifrc net renderer support
Dear Larry,

Gentoo on cloud - musl good on memory footprint. The problem: no NM, networkd -
no sophisticated net config. Eg. multiple network interfaces on EC2, multiple
IPv4 and IPv6 addresses on an interface, custom routes.

## USE or not USE
The Netifrc support patch is applied only when emerged with USE="netifrc". The
patch has minimal side effects. Don't use USE="netifrc"?

## Hotplug support
Attaching an iface works. The iface will be rendered and the service started as
expected. Detaching would not. Possible race condition problem.

## Missing RDEPEND: locale-gen
TODO

## CAVEAT: detached interfaces
TODO: you'll lose netmount
