# 設定上網(LAN,WAN,DHCP,NAT..etc)
## 設定LAN port
```
/interface bridge
add name=bridge1
```
### 將實體port設定至LAN  
(ether2~{4}或是更多)
```
/interface bridge port
add bridge=bridge1 interface=ether2 internal-path-cost=10 path-cost=10
add bridge=bridge1 interface=ether3 internal-path-cost=10 path-cost=10
add bridge=bridge1 interface=ether4 internal-path-cost=10 path-cost=10
```
## 設定WAN, LAN (提供給firewall rule使用, option)
```
/interface list
add name=WAN
add name=LAN
/interface list member
add interface=ether1 list=WAN
add interface=bridge1 list=LAN
```
## 設定WAN&LAN ip
```
/ip address
add address=外網ip interface=ether1 network=子網路遮罩 
add address=192.168.1.1 interface=bridge1 network=192.168.1.0
```
### 如果是撥號(hinet)
```
/interface pppoe-client
add add-default-route=yes dial-on-demand=yes disabled=no interface=ether1 name=pppoe-out1 use-peer-dns=yes user=xxxx@hinet.net
```
## 設定內網dhcp
```
/ip pool
add name=dhcp1 ranges=192.168.1.100-192.168.1.200
/ip dhcp-server
add address-pool=dhcp1 interface=bridge1 name=server1
/ip dhcp-server network
add address=192.168.1.0/24 dns-server=168.95.1.1,8.8.8.8 gateway=192.168.1.1 netmask=24
```
## 設定NAT
```
/ip firewall nat
add action=masquerade chain=srcnat src-address=192.168.1.0/24
```
# 設定基本功能
## 時間
```
/system clock
set time-zone-name=Asia/Taipei
/system ntp client
set enabled=yes
/system ntp client servers
add address=tock.stdtime.gov.tw
```
## 服務(為了安全建議都關閉或是限制登入ip)
```
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set ssh address=192.168.1.0/24
set api disabled=yes
set api-ssl disabled=yes
```
## 防止WinBox(port 8291)被try  
(允許錯3次,3次全錯阻擋10min)  
(ref:https://mhelp.pro/mikrotik-fail2ban-blocking-brute-force-attacks/)
```
/ip firewall filter
add action=jump chain=output comment="F2B Winbox: Jump to Fail2Ban-Destination-IP chain" content="invalid user name or password" jump-target=Fail2Ban-Destination-IP protocol=tcp src-port=8291
add action=add-dst-to-address-list address-list=BlackList address-list-timeout=10m chain=Fail2Ban-Destination-IP comment="3 Attempt --> BlackList" dst-address-list=LoginFailure02
add action=add-dst-to-address-list address-list=LoginFailure02 address-list-timeout=2m chain=Fail2Ban-Destination-IP comment="2 Attempt --> LoginFailure02" dst-address-list=LoginFailure01
add action=add-dst-to-address-list address-list=LoginFailure01 address-list-timeout=1m chain=Fail2Ban-Destination-IP comment="1 Attempt --> LoginFailure01"
/ip firewall raw
add action=drop chain=prerouting comment="Drop all" src-address-list=BlackList
```
# 清LOG
```
/system script
add dont-require-permissions=no name=ClearLog owner=admin policy=\
    ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon source=":\
    local memoryline [:put [/sys logg action get \"memory\" \"memory-lines\" ]\
    \_]\r\
    \n/system logging action set memory memory-lines=1\r\
    \n/system logging action set memory memory-lines=\$memoryline\r\
    \n:log info \"Clear Log\""
```
# 設定VPN
## IPSec
```
/ip ipsec proposal
add enc-algorithms=aes-128-cbc,aes-128-ctr lifetime=30m name=proposal1 pfs-group=modp1536
/ip ipsec profile
add dh-group=modp1536 dpd-interval=2m dpd-maximum-failures=5 enc-algorithm=aes-128 name=profile1
/ip ipsec peer
add exchange-mode=ike2 name=peer1 passive=yes profile=profile1
/ip ipsec identity
add my-id=fqdn:test01 peer=peer1 remote-id=fqdn:test01 auth-method=pre-shared-key secret="密碼"
/ip ipsec policy
add dst-address=對方網段 peer=peer1 src-address=192.168.1.0/24(本地網段) tunnel=yes proposal=proposal1
/ip firewall nat
add action=accept chain=srcnat dst-address=對方網段 src-address=192.168.1.0/24(本地網段)
/ip route
add disabled=no distance=1 dst-address=對方網段 gateway=bridge1 routing-table=main scope=30 suppress-hw-offload=no target-scope=10
```
## Wireguard  
(對端內網:192.168.10.0/24,對端tunnel ip:10.255.255.1/24)
```
/interface wireguard
add listen-port=13231 mtu=1420 name=wireguard1 private-key="你的private-key" public-key="你的public-key"
/ip address
add address=10.255.255.3/24 interface=wireguard1 network=10.255.255.0
/interface wireguard peers
add allowed-address=10.255.255.1/24,192.168.10.0/24 endpoint-address=對端wan_ip endpoint-port=13231 interface=wireguard1 name=wireguard1 persistent-keepalive=30s public-key="對端public-key"
/ip firewall filter
add action=accept chain=input comment="allow WireGuard" dst-port=13231 protocol=udp
/ip route
add dst-address=192.168.10.0/24 gateway=wireguard1
```
# Forwarding
## Forwarding SSH
```
/ip firewall filter
add action=accept chain=forward dst-address=(SSH server IP) dst-port=22 protocol=tcp
/ip firewall nat
add action=dst-nat chain=dstnat dst-address=(WAN ip) dst-port=22 protocol=tcp to-addresses=(SSH server IP) to-ports=22
```
