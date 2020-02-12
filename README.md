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
Create new VPN gateway is NOT support currently (2020 Feb). Please update PowerShell to latest version to create gateway. In this example, we create Generation 2 VpvGw5 as example. <br>
`
New-AzResourceGroup -Name TestRG2 -Location chinanorth2
$virtualNetwork = New-AzVirtualNetwork -ResourceGroupName TestRG2 -Location chinanorth2 -Name VNet2 -AddressPrefix 10.2.0.0/16
$subnetConfig = Add-AzVirtualNetworkSubnetConfig -Name Frontend -AddressPrefix 10.2.0.0/24 -VirtualNetwork $virtualNetwork
$virtualNetwork | Set-AzVirtualNetwork
$vnet = Get-AzVirtualNetwork -ResourceGroupName TestRG2 -Name VNet2
Add-AzVirtualNetworkSubnetConfig -Name 'GatewaySubnet' -AddressPrefix 10.2.255.0/24 -VirtualNetwork $vnet
$vnet | Set-AzVirtualNetwork
$gwpip= New-AzPublicIpAddress -Name VNet2GWIP -ResourceGroupName TestRG2 -Location chinanorth2 -AllocationMethod Dynamic
$vnet = Get-AzVirtualNetwork -Name VNet2 -ResourceGroupName TestRG2
$subnet = Get-AzVirtualNetworkSubnetConfig -Name 'GatewaySubnet' -VirtualNetwork $vnet
$gwipconfig = New-AzVirtualNetworkGatewayIpConfig -Name gwipconfig2 -SubnetId $subnet.Id -PublicIpAddressId $gwpip.Id
New-AzVirtualNetworkGateway -Name VNet2GW -ResourceGroupName TestRG2 -Location chinanorth2 -IpConfigurations $gwipconfig -GatewayType Vpn -VpnType RouteBased -GatewaySku VpnGw5 -VpnGatewayGeneration Generation2
` 


Create VPN Connection
----------

Setup P2S VPN
---------

Network reachability Testing
-----------

Transit routing setup
----------
