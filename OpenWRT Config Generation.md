---
layout: post
title: OpenWRT Config Generation
aliases:
  - "Blog: OpenWRT config generation"
date: 2024-02-22
tags: 
draft: false
---
In OpenWRT, all config files under `/etc/config` are generated dynamically. There are scripts in `/etc/uci-defaults` to construct this configuration.

The scripts in `/etc/uci-defaults` are executed on first boot and then removed, but we could find them in the image. For example, under imagebuilder root, `build_dir/target-aarch64_cortex-a72_musl/root-bcm27xx/etc/uci-defaults/` are the rootfs of bcm27xx targets.

```shell
10_migrate-shadow        14_migrate-dhcp-release  50-dnsmasq-migrate-resolv-conf-auto.sh
12_network-generate-ula  15_odhcpd                90-resize-filesystem-to-partition-size
13_fix-group-user        20_migrate-feeds         99-override-network-proto-lan-to-dhcp
```

The content of `12_network-generate-ula` is

```shell
[ "$(uci -q get network.globals.ula_prefix)" != "auto" ] && exit 0

r1=$(dd if=/dev/urandom bs=1 count=1 |hexdump -e '1/1 "%02x"')
r2=$(dd if=/dev/urandom bs=2 count=1 |hexdump -e '2/1 "%02x"')
r3=$(dd if=/dev/urandom bs=2 count=1 |hexdump -e '2/1 "%02x"')

uci -q batch <<-EOF >/dev/null
        set network.globals.ula_prefix=fd$r1:$r2:$r3::/48
        commit network
EOF

exit 0
```

If we want to change this default configuration, just add another uci-defaults script with larger number in the filename, i.e. `00_customize_network_setting`.

Sometimes we want to use built-in scripts to save our time of writing the script using raw UCI commands. There are functions in `build_dir/target-aarch64_cortex-a72_musl/root-bcm27xx/lib/functions/` to provide such functionality. For example, there is `network.sh` which defines some functions to make the scripting more easily.
