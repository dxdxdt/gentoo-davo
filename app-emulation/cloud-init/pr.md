# Experimental Cloud-init Netifrc net renderer support
Dear Larry,

Consider this PR as one step closer to bringing Gentoo on cloud services.

Currently, Gentoo on cloud requires glibc becuase:

- Gentoo doesn't support the traditional Debian style "ifupdown" scripts(dubbed
  "ENI" by Cloud-init)
- Cloud-init had no Netifrc renderer(until now)

So, we're left with two choices: NetworkManager and networkd w/ Systemd as
initd.

Some may find this wasteful. Some are not just fans of Systemd at all. Some may
want to use the Musl profile to get more memory out of cloud instances.

For most cases, running Cloud-init without network rendering is okay as long as
only one address of each IP version(4, 6) on one network interface is involved.
Anything more than that, for example, multiple addresses, pushing routes, DNS
servers and so on cannot be achieved by simply running dhcpcd due to the
limitation of DHCP as well as provider's architectural design.

As Netifrc and standalone dhcpcd were the only options with Musl, complex cloud
network setup on cloud instances was not possible.

The patches add the experimental Netifrc network renderer support. Musl Gentoo
users can now enjoy cloud-init's network rendering.

Wiki docs to follow upon approval. Upstream merging after series of automated
tests. The tests are not included in the patch file. The working upstream is
https://github.com/dxdxdt/cloud-init/tree/netifrc if you want have a look.

Also adds the missing `sys-apps/locale-gen` RDEPEND for locale set up from the
IMDS.

## INSTALL
After emerging the package, add the services to relevant runlevels:

```sh
rc-update add cloud-init-ds-identify boot
rc-update add cloud-init-local boot

rc-update add cloud-final
rc-update add cloud-init
rc-update add cloud-init-hotplug
```

This should be enough on an Openrc system because Cloud-init can autodetect the
only available net renderer on the system(Netifrc).

Just in case you're having trouble, net renderer can be specified like so:

/etc/cloud/cloud.cfg:
```
system_info:
  distro: gentoo
  default_user:
    name: gentoo
    lock_passwd: True
    gecos: gentoo Cloud User
    groups: [users, wheel]
    primary_group: users
    no_user_group: true
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
  network:
    renderers: ['netifrc']
    activators: ['netifrc']
```

## USE or not USE
The Netifrc support patch is applied only when emerged with USE="netifrc". The
patch has minimal side effects. Perhaps the USE flag shouldn't be used?

## Hotplug support
"Hotplug" is when you attach a network interface to the running instance and the
cloud-init handles that event and renders the network config on the fly.

Cloud-init's hotplug support on Openrc is still work in progress. See
https://github.com/canonical/cloud-init/issues/6351

I recommend rebooting or powering off the instance before attaching/detaching an
interface. The service `cloud-init-hotplug` is responsible for this feature.
Remove it from the runlevel if you don't want it.

To invalidate net cache, run: `cloud-init clean -c network`

## locale-gen support
Glibc supports fully functional locale-gen command. Musl does not come with a
functioning locale-gen command so cloud-init-config will crash when the IMDS
pushes the locale setting to the instance.

Create the file at `/etc/cloud/cloud.cfg.d/99-musl-disable-locale.cfg` with the
following contents to block locale settings:

```
locale: false
```

## What's been tested
### EC2
- Multiple static addresses
- Multiple ENI
- Hotplugging

metadata:
```python
{
    "version": 2,
    "ethernets": {
        "ens6": {
            "dhcp4": True,
            "dhcp4-overrides": {"route-metric": 200},
            "dhcp6": False,
            "match": {"macaddress": "0e:14:5a:19:f5:59"},
            "set-name": "ens6",
        },
        "ens5": {
            "dhcp4": True,
            "dhcp4-overrides": {"route-metric": 100},
            "dhcp6": True,
            "match": {"macaddress": "0e:87:95:5a:5c:5d"},
            "set-name": "ens5",
            "dhcp6-overrides": {"route-metric": 100},
            "addresses": [
                "172.31.1.240/24",
                "2001:db8::f340:f1d1:3962:b2b0/128",
            ],
        },
    },
}
```

rendered:
```sh
# This file is generated from information provided by the datasource. Changes
# to it will not persist across an instance reboot. To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}


config_ens6="dhcp"
dhcpcd_ens6="-4"

config_ens5="172.31.1.240/24
2001:db8::f340:f1d1:3962:b2b0/128
dhcp"
```

## TODO What's not been tested
- `config_...="null"` (virtual routers)
- `routes_` (prefix delegation?)
- `dns_search_`
- `dns_servers_`
- `mtu_` (GCP)
- Rare/weird bare metal set ups like: VLAN, bridge, bond, bond+VLAN,
  bridge+bond+VLAN ...
- And the rest
