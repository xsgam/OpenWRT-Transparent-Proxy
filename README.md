# OpenWRT Transparent Proxy 

Use iptables and transocks in Openwrt to transparently forward the TCP connection to a remote SOCKS5 server or a HTTP proxy , allowing the PC to achieve transparent proxy access to the Internet through WRT.



## 1、transocks 

>https://github.com/cybozu-go/transocks

```bash
transocks is a background service to redirect TCP connections transparently to a SOCKS5 server or a HTTP proxy server like Squid.

Currently, transocks supports only Linux iptables with DNAT/REDIRECT target.
```

- build transocks in Windows , and upload to OpenWRT via ssh client (ex. MobaXterm)
```shell
@echo off

mkdir release

SET OUTPATH=./release/
SET OUTNAME=gotrans
SET CGO_ENABLED=0
SET LDFLAGS=-w -s
SET GO_MAIN=cmd\transocks\main.go

SET GOOS=linux
SET GOARCH=386
echo build %GOOS%_%GOARCH%
go build -o %OUTPATH%%OUTNAME%_%GOOS%_%GOARCH% -ldflags "%LDFLAGS%" %GO_MAIN%

SET GOOS=linux
SET GOARCH=amd64
echo build %GOOS%_%GOARCH%
go build -o %OUTPATH%%OUTNAME%_%GOOS%_%GOARCH% -ldflags "%LDFLAGS%" %GO_MAIN%
```

- edit config transocks.toml

```bash
# listening address of transocks.
listen = "0.0.0.0:1081"    # default is "localhost:1081"

proxy_url = "socks5://10.20.30.40:1080"  # for SOCKS5 server
#proxy_url = "http://10.20.30.40:3128"   # for HTTP proxy server
```

- start transocks in openwrt

```shell
root@OpenWrt:~# ./transocks -f transocks.toml
```



## 2、Redirecting connections by iptables

Use DNAT or REDIRECT target in OUTPUT chain of the nat table.

- create a shell script file:  /root/start_fw.sh

```shell
iptables-save -c | grep -v "TRANSOCKS" | iptables-restore -c
iptables-restore -n <<-EOF
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:TRANSOCKS - [0:0]

#-A PREROUTING -p tcp -j TRANSOCKS
-A prerouting_lan_rule -p tcp -j TRANSOCKS
-A OUTPUT -p tcp -j TRANSOCKS

-A TRANSOCKS -d 0.0.0.0/8 -j RETURN
-A TRANSOCKS -d 10.0.0.0/8 -j RETURN
-A TRANSOCKS -d 127.0.0.0/8 -j RETURN
-A TRANSOCKS -d 169.254.0.0/16 -j RETURN
-A TRANSOCKS -d 172.16.0.0/12 -j RETURN
-A TRANSOCKS -d 192.168.0.0/16 -j RETURN
-A TRANSOCKS -d 224.0.0.0/4 -j RETURN
-A TRANSOCKS -d 240.0.0.0/4 -j RETURN
-A TRANSOCKS -p tcp -j REDIRECT --to-ports 1081
COMMIT
```

- run shell  script to update iptables

  > /root/start_fw.sh

- If you don't deed transparent proxy,  use this command to reset iptables

  > iptables-save -c | grep -v "TRANSOCKS" | iptables-restore -c

   

##　3. Config DNS over TLS on OpenWrt.

> https://openwrt.org/docs/guide-user/services/dns/dot_unbound

- Replacing dnsmasq with odhcpd and Unbound

   Remove dnsmasq and use odhcpd for both DHCP and DHCPv6. 

  ```shell
  opkg update
  opkg remove dnsmasq odhcpd-ipv6only
  opkg install odhcpd
  uci -q delete dhcp.@dnsmasq[0]
  uci set dhcp.lan.dhcpv4="server"
  uci set dhcp.odhcpd.maindhcp="1"
  uci commit dhcp
  service odhcpd restart
  ```

   Use Unbound for DNS. 

  ```shell
  opkg update
  opkg install unbound-control unbound-daemon
  uci set unbound.@unbound[0].add_local_fqdn="3"
  uci set unbound.@unbound[0].add_wan_fqdn="1"
  uci set unbound.@unbound[0].dhcp_link="odhcpd"
  uci set unbound.@unbound[0].dhcp4_slaac6="1"
  uci set unbound.@unbound[0].unbound_control="1"
  uci commit unbound
  service unbound restart
  uci set dhcp.odhcpd.leasetrigger="/usr/lib/unbound/odhcpd.sh"
  uci commit dhcp
  service odhcpd restart
  ```

  

- Install the required packages. Enable DNS encryption. 

  ```shell
  # Install packages
  opkg update
  opkg install unbound-daemon
  opkg install luci-app-unbound
   
  # Enable DNS encryption
  uci set unbound.fwd_google.enabled="1"
  uci set unbound.fwd_google.fallback="0"
  uci commit unbound
  service unbound restart
  
  
  ```

- Change to Cloudflare DNS

  ```shell
  # Configure DoT provider
  uci set unbound.fwd_google.enabled="0"
  uci set unbound.fwd_cloudflare.enabled="1"
  uci set unbound.fwd_cloudflare.fallback="0"
  uci commit unbound
  service unbound restart
  ```

# Testing

- Test TCP Redirecting

  ```
  root@OpenWrt:~# wget https://1.1.1.1 
  ```

  use wget to connect http server 1.1.1.1 without proxy,  iptables will redirect http connection to 1081 that listen by transocks, and transocks will forward http connection to remote socks5 server

- Test DNS

  ```
  root@OpenWrt:~# nslookup openwrt.org localhost
  ```

  Check your DNS provider and test DNSSEC validation.

  - https://dnsleaktest.com
  - https://dnssec-tools.org/test

- Test PC

  - openwrt will send DOT request to Cloudflare DNS.

  ```
  c:\>nslookup www.x.com
  ```

  - start Firefox/Chrome , visit some website。

    

# Troubleshooting

- Collect and analyze the following information. 

  ```
  # Restart services
  service log restart; service unbound restart
   
  # Log and status
  logread -e unbound; netstat -l -n -p | grep -e unbound
   
  # Runtime configuration
  pgrep -f -a unbound
  head -v -n -0 /etc/resolv.* /tmp/resolv.* /tmp/resolv.*/*
   
  # Persistent configuration
  uci show unbound
  ```

# Reference

> https://openwrt.org/docs/guide-user/services/dns/dot_unbound

> https://openwrt.org/docs/guide-user/base-system/dhcp_configuration#replacing_dnsmasq_with_odhcpd_and_unbound
>
> https://github.com/cybozu-go/transocks

