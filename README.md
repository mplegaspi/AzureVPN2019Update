AzureVPN
====================

On 2019 Microsoft Ignite, Azure released new VPN gateway SKU VpnGw1-5. New gateway have better performance, support IKEv1 and IKEv2 at the same time and support multiple IKEv1 tunnel. This lab will setup an environment to demo this. <br> 
![](https://github.com/yinghli/AzureVPN/blob/master/vpn.jpg)

Lab Topology
------------
We simulate three sites in this topology. Two sites connect to Azure via IKEv1 IPSec VPN, one site connect to Azure via IKEv2 IPSec VPN. To cover as much use case as possible, we also add point to site VPN and Hub-Spoke architecture at Azure side. <br>

![](https://github.com/yinghli/AzureVPN/blob/master/IKE.jpg)

Parameters            | Azure          |Site1         |Site2         |Site3         |VPNClient
----------------------| -------------  |-----------   |---------     |------------  |------------
Public IP             |40.73.39.223    |52.130.80.146 |40.73.245.64  |52.130.80.50  |Dynamic
Local Network         |10.2.0.0/15     |10.100.0.0/16 |10.150.0.0/16 |10.200.0.0/16 |172.16.0.0/24
Tunnel Type           |IKEv1&IKEv2&SSTP|IKEv1         |IKEv1         |IKEv2         |SSTP


Create New VPN gateway
---------
Create new VPN gateway at Azure China Portal is NOT support currently (2020 Feb). Please update PowerShell to latest version to create gateway. In this example, we create Generation 2 VpvGw5 as example. <br>

Setup resource group, virtual network and gateway subnet. 
```
New-AzResourceGroup -Name TestRG2 -Location chinanorth2
$virtualNetwork = New-AzVirtualNetwork -ResourceGroupName TestRG2 -Location chinanorth2 -Name VNet2 -AddressPrefix 10.2.0.0/16
$subnetConfig = Add-AzVirtualNetworkSubnetConfig -Name Frontend -AddressPrefix 10.2.0.0/24 -VirtualNetwork $virtualNetwork
$virtualNetwork | Set-AzVirtualNetwork
$vnet = Get-AzVirtualNetwork -ResourceGroupName TestRG2 -Name VNet2
Add-AzVirtualNetworkSubnetConfig -Name 'GatewaySubnet' -AddressPrefix 10.2.255.0/24 -VirtualNetwork $vnet
$vnet | Set-AzVirtualNetwork
```
Create vpn gateway public IP
```
$gwpip= New-AzPublicIpAddress -Name VNet2GWIP -ResourceGroupName TestRG2 -Location chinanorth2 -AllocationMethod Dynamic
$vnet = Get-AzVirtualNetwork -Name VNet2 -ResourceGroupName TestRG2
$subnet = Get-AzVirtualNetworkSubnetConfig -Name 'GatewaySubnet' -VirtualNetwork $vnet
$gwipconfig = New-AzVirtualNetworkGatewayIpConfig -Name gwipconfig2 -SubnetId $subnet.Id -PublicIpAddressId $gwpip.Id
```
Create Generation2 VpnGw5
```
New-AzVirtualNetworkGateway -Name VNet2GW -ResourceGroupName TestRG2 -Location chinanorth2 -IpConfigurations $gwipconfig -GatewayType Vpn -VpnType RouteBased -GatewaySku VpnGw5 -VpnGatewayGeneration Generation2
```


Create VPN Connection
----------
After VPN gateway created, we also need to create Local Network Gateway for each site. Here is the example for site1.
```
New-AzLocalNetworkGateway -Name csr100v -ResourceGroupName TestRG1 -Location chinanorth2 -GatewayIpAddress '52.130.80.146' -AddressPrefix '10.100.0.0/16'
```
Then we can create site to site IPSec VPN connection.<br>
For site1, this is IKEv1 IPSec VPN connection. We need to input `-ConnectionProtocol` as IKEv1.
```
$vpngw = Get-AzVirtualNetworkGateway -Name VNet2GW -ResourceGroupName TestRG2
$lng = Get-AzLocalNetworkGateway  -Name csr100v -ResourceGroupName TestRG2
New-AzVirtualNetworkGatewayConnection -Name IKEv1Conn -ResourceGroupName TestRG2 -VirtualNetworkGateway1 $vpngw -LocalNetworkGateway2 $lng -ConnectionType IPsec -ConnectionProtocol IKEv1 -SharedKey 'cisco' -Location chinanorth2
```
For site3, this is IKEv2 connection. We need to input `-ConnectionProtocol` as IKEv2. You can also add customized IPSec policy for each connection.
```
$ipsecpolicy = New-AzIpsecPolicy -IkeEncryption AES256 -IkeIntegrity SHA256 -DhGroup DHGroup24 -IpsecEncryption AES256 -IpsecIntegrity SHA256 -PfsGroup None -SALifeTimeSeconds 14400 -SADataSizeKilobytes 102400000
New-AzVirtualNetworkGatewayConnection -Name IKEv2Conn -ResourceGroupName TestRG2 -VirtualNetworkGateway1 $vpngw -LocalNetworkGateway2 $lng1 -ConnectionType IPsec -ConnectionProtocol IKEv2 -SharedKey 'cisco' -Location chinanorth2 -IpsecPolicies $ipsecpolicy
```

On Premise VPN Site
------------------
We setup Cisco CSR1000v to simulate remote VPN site. <br>
Here is demo configuration on site1 Cisco CSR1000v, site1 use IKEv1 to setup IPSec VPN tunnel.
```
access-list 101 permit ip 10.100.0.0 0.0.255.255 10.2.0.0 0.0.254.255
!
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
 group 2
 lifetime 28800
crypto isakmp key cisco address 40.73.39.223
!
crypto ipsec transform-set azure-ipsec-proposal-set esp-aes 256 esp-sha-hmac
 mode tunnel
!
crypto map azure-crypto-map 10 ipsec-isakmp
 set peer 40.73.39.223
 set security-association lifetime kilobytes 102400000
 set transform-set azure-ipsec-proposal-set
 match address 101
!
interface Loopback0
 ip address 10.100.0.1 255.255.0.0
!
interface GigabitEthernet1
 crypto map azure-crypto-map
```
For detail IKEv2 configuration and setup, please refer [this](https://github.com/yinghli/azure-vpn-csr1000v). <br>

We also setup SSTP Point to Site VPN to simulate remote workers. For detail, please refer [this](https://github.com/yinghli/Azure-P2S-VPN). <br>

Network reachability Testing
-----------
From site1, after VPN tunnle is up. ICMP test to Hub VNET2 and Spoke VNET3 is working.
```
csrvm#ping 10.2.0.4 source 10.100.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.2.0.4, timeout is 2 seconds:
Packet sent with a source address of 10.100.0.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 23/24/26 ms

csrvm#ping 10.3.1.4 source 10.100.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.3.1.4, timeout is 2 seconds:
Packet sent with a source address of 10.100.0.1
!!!!!
```
From site3, which is IKEv2 IPSec VPN setup. ICMP test to Hub VNET2 and Spoke VNET3 is working.
```
csr1000v2#ping 10.2.0.4 source 10.200.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.2.0.4, timeout is 2 seconds:
Packet sent with a source address of 10.200.0.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 24/24/26 ms

csr1000v2#ping 10.3.1.4 source 10.200.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.3.1.4, timeout is 2 seconds:
Packet sent with a source address of 10.200.0.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 23/23/24 ms
```
For remote VPN user, my laptop dial in with SSTP tunnel, get 172.16.0.6 as remote IP.<br>
Host route table show that 10.2.0.0/23 next hop is 172.16.0.6 and ICMP test is good.<br>
```
PPP adapter VNet2:

   Connection-specific DNS Suffix  . :
   IPv4 Address. . . . . . . . . . . : 172.16.0.6
   Subnet Mask . . . . . . . . . . . : 255.255.255.255
   Default Gateway . . . . . . . . . :
   
IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
         10.2.0.0      255.255.0.0         On-link        172.16.0.6     43
     10.2.255.255  255.255.255.255         On-link        172.16.0.6    281
         10.3.0.0      255.255.0.0         On-link        172.16.0.6     43
     10.3.255.255  255.255.255.255         On-link        172.16.0.6    281

C:\Users\yinghli>ping 10.2.0.4

Pinging 10.2.0.4 with 32 bytes of data:
Reply from 10.2.0.4: bytes=32 time=5ms TTL=63
Reply from 10.2.0.4: bytes=32 time=5ms TTL=63
Reply from 10.2.0.4: bytes=32 time=4ms TTL=63
Reply from 10.2.0.4: bytes=32 time=4ms TTL=63

Ping statistics for 10.2.0.4:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 4ms, Maximum = 5ms, Average = 4ms

C:\Users\yinghli>ping 10.3.1.4

Pinging 10.3.1.4 with 32 bytes of data:
Reply from 10.3.1.4: bytes=32 time=3ms TTL=63
Reply from 10.3.1.4: bytes=32 time=5ms TTL=63
Reply from 10.3.1.4: bytes=32 time=5ms TTL=63
Reply from 10.3.1.4: bytes=32 time=6ms TTL=63

Ping statistics for 10.3.1.4:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 3ms, Maximum = 6ms, Average = 4ms
    
```

Transit routing setup
----------
By default, site1, site2, site3 and remote worker can only access Azure VNET resource. They can’t talk to each other. But for IKEv2 and point to site VPN, you can add static route to point remote site address range to VPN tunnel. For IKEv1, you can modify local access list to add remote site address range into “encryption traffic” list. 
For example, for IKEv2 site2, you can add static route.
```
ip route 10.100.0.0 255.255.0.0 Tunnel1
ip route 10.150.0.0 255.255.0.0 Tunnel1
ip route 172.16.0.0 255.255.255.0 Tunnel1
```
For IKEv1 site1, you can modify access list.
```
access-list 101 permit ip 10.100.0.0 0.0.255.255 172.16.0.0 0.0.0.255
access-list 101 permit ip 10.100.0.0 0.0.255.255 10.200.0.0 0.0.255.255
access-list 101 permit ip 10.100.0.0 0.0.255.255 10.150.0.0 0.0.255.255
```
For remote VPN host, you can add static route. 172.16.0.6 is current dynamic IP assigned by VPN server. It may change after next dial in. 
```
route add 10.200.0.0 mask 255.255.0.0 -p 172.16.0.6
route add 10.100.0.0 mask 255.255.0.0 -p 172.16.0.6
route add 10.150.0.0 mask 255.255.0.0 -p 172.16.0.6
```
