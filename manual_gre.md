## Setup Tunnel
### SERVER A
Make sure IP forwarding is enabled.
```
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
sysctl -p 
```
Create a new tunnel via GRE protocol
```
ip tunnel add tunnel0 mode gre local SERVER_A_IP remote SERVER_B_IP ttl 255
```
Add a private subnet to be used on the tunnel
```
ip addr add 192.168.0.1/30 dev tunnel0
```
Turn on the tunnel
```
ip link set tunnel0 up
```

### SERVER B
# Create a new tunnel via GRE protocol
ip tunnel add tunnel0 mode gre local SERVER_B_IP remote SERVER_A_IP ttl 255
# Add a private subnet to be used on the tunnel
ip addr add 192.168.0.2/30 dev tunnel0
# Turn on the tunnel
ip link set tunnel0 up
# Create a new routing table
echo '100 GRE' >> /etc/iproute2/rt_tables
# Make sure to honor the rules for the private subnet via that table
ip rule add from 192.168.168.0/30 table GRE
# Make sure all traffic goes via SERVER_A ip
ip route add default via 192.168.168.1 table GRE

# Test if the traffic is outgoing through SERVER_A
curl http://ipinfo.io --interface tunnel0

### SERVER A
# Accept to and from traffic for the private IP of SERVER_B
iptables -A FORWARD -d 192.168.0.2 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -s 192.168.0.2 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
# Setup a portforward of PORT 80 from SERVER_A to port 8000 in SERVER_B
iptables -t nat -A PREROUTING -d SERVER_A -p tcp -m tcp --dport 80 -j DNAT --to-destination 192.168.0.2:8000

## Interfaces / Setup On Boot
### SERVER A
auto gre1
iface gre1 inet static
  address 192.168.200.1/30
  netmask 255.255.255.252
  pre-up ip tunnel add tunnel0 mode gre remote OTHER_IP_ADDRESS local PUFFERFISH_HOST_IP_1
  post-up iptables -t nat -A POSTROUTING -s 192.168.200.0/30 ! -o gre+ -j SNAT --to-source PUFFERFISH_HOST_IP_2
  post-up iptables -t nat -A PREROUTING -d PUFFERFISH_HOST_IP_2 -j DNAT --to-destination 192.168.200.2
  post-up iptables -A FORWARD -d 192.168.200.2 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
  post-down ip tunnel del tunnel0

### SERVER B
auto gre1
iface gre1 inet static
  address 192.168.200.2/30
  netmask 255.255.255.252
  pre-up ip tunnel add tunnel0 mode gre remote PUFFERFISH_HOST_IP_1 local OTHER_IP_ADDRESS
  post-up ip rule add from 192.168.200.0/30 table PUFFHOST
  post-up ip route add default via 192.168.200.1 table PUFFHOST
  post-down ip tunnel del tunnel0
