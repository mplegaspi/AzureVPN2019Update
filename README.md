AzureVPN
====================

On 2019 Microsoft Ignite, Azure release new VPN gateway SKU VpnGw1-5. New gateway have better performance, support IKEv1 and IKEv2 at the same time and support multiple IKEv1 tunnel. This lab will setup an environment to demo this. <br> 
![](https://github.com/yinghli/AzureVPN/blob/master/vpn.jpg)

Lab Topology
------------
We simulate three sites in this topology. Two sites connect to Azure via IKEv1 IPSec VPN, one site connect to Azure via IKEv2 IPSev VPN. In order to cover as much use case as possible, we aslo add point to site VPN and Hub-Spoke architecture at Azure side. <br>
![](https://github.com/yinghli/AzureVPN/blob/master/IKE.jpg)


Create New VPN gateway
---------


Create VPN Connection
----------

Setup P2S VPN
---------

Network reachability Testing
-----------

Transit routing setup
----------
