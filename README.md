---
title: Google Cloud VPN Interop Guide for [PRODUCT]
description: Describes how to build site-to-site IPsec VPNs between Cloud VPN on Google Cloud Platform (GCP) and  [VENDOR] [PRODUCT]
author: [AUTHOR]
tags: VPN, interop, [VENDOR], [PRODUCT]
date_published: YYYY-mm-dd
---



# Using Cloud VPN with Strongswan

Learn how to build site-to-site IPsec VPNs between
[Cloud VPN](https://cloud.google.com/vpn/docs/) on Google Cloud Platform (GCP) and
Strongswan.


- [Introduction](#introduction)
- [Terminology](#Terminology)
- [Topology](#topology)
- [Product environment](#product-environment)
- [Before you begin](#before-you-begin)
    - [GCP account and project](#gcp-account-and-project)
    - [Permissions](#permissions)
- [Configure the GCP side](#configure-the-gcp-side)
    - [Initial tasks](#initial-tasks)
        - [Select a GCP project name](#select-a-gcp-project-name)
        - [Create a custom VPC network and subnet](#create-a-custom-vpc-network-and-subnet)
        - [Create the GCP external IP address](#create-the-gcp-external-ip-address)
    - [Configuring route-based IPsec VPN using static routing](#configuring-route-based-ipsec-vpn-using-static-routing)
        - [Configure the VPN gateway](#configure-the-VPN-gateway)
        - [Configure the VPN tunnel](#Configure-the-VPN-tunnel)
        - [Configure firewall](#Configure-firewall)
- [Configure the Strongswan side](#configure-the-Strongswan-side)
    - [GCP-compatible settings for IPSec and IKE](#gcp-compatible-settings-for-ipsec-and-ike)
    - [Creating the VPN configuration](#creating-the-vpn-configuration)
    - [Testing the configuration](#testing-the-configuration)
- [Troubleshooting IPsec on Strongswan side](#troubleshooting-ipsec-on-Strongswan-side)
- [Reference documentation](#reference-documentation)
    - [GCP documentation](#gcp-documentation)
    - [Strongswan documentation](#Strongswan-documentation)
- [Appendix: Using gcloud commands](#appendix-using-gcloud-commands)
    - [Running gcloud commands](#running-gcloud-commands)
    - [Configuration parameters and values](#configuration-parameters-and-values)
    - [Setting environment variables for gcloud command parameters](#setting-environment-variables-for-gcloud-command-parameters)
    - [Configuring IPsec VPN using static routing](#Configuring-IPsec-VPN-using-static-routing)


![screen shot 2018-08-20 at 3 58 44 pm](https://user-images.githubusercontent.com/42278789/44371087-fc9a8b80-a491-11e8-86c7-22499e7b5913.png)

Author: <author name and email address>

## Introduction

This guide walks you through the process of configuring
Strongswan for integration with the
[Cloud VPN service](https://cloud.google.com/vpn/docs) on GCP.

For more information about Cloud VPN, see the
[Cloud VPN Overview](https://cloud.google.com/compute/docs/vpn/overview).

## Terminology

Below are definitions of terms used throughout this guide.

<This is some sample terminology. Add any terminology to this section that needs
explanation.>

-  **GCP VPC network**—A single virtual network within a single GCP project.
-  **On-premises gateway**—The VPN device on the non-GCP side of the
connection, which is usually a device in a physical data center or in
another cloud provider's network. GCP instructions are written from the
point of view of the GCP VPC network, so "on-premises gateway" refers to the
gateway that's connecting _to_ GCP.
-  **External IP address** or **GCP peer address**—a single static IP address
within a GCP project that exists at the edge of the GCP network.
-  **Static routing**—Manually specifying the route to subnets on the GCP
side and to the on-premises side of the VPN gateway.


## Topology

Cloud VPN supports the following topologies:
![screen shot 2018-08-20 at 3 26 12 pm](https://user-images.githubusercontent.com/42278789/44370966-73835480-a491-11e8-9703-bc90e185d65e.png)

-  A site-to-site IPsec VPN tunnel configuration using static routing.

For detailed topology information, see the following resources:

-  For basic VPN topologies, see 
[Cloud VPN Overview](https://cloud.google.com/vpn/docs/concepts/overview).

## Product environment

The <vendor-name><product-name> equipment used in this guide is as follows:

-  Vendor—Strongswan
-  Model—5.6.1
-  Software release—CentOS 7.5


## Before you begin

Follow the steps in this section to prepare for VPN configuration.

**Note**: This guide assumes that you have basic knowledge of the
[IPsec](https://wikipedia.org/wiki/IPsec) protocol.

### GCP account and project

Make sure you have a GCP account. When you begin, you must select or create a
GCP project where you will build the VPN. For details, see
[Creating and Managing Projects](https://cloud.google.com/resource-manager/docs/creating-managing-projects).

### Permissions

To create a GCP network, a subnetwork, and other entities described in this
guide, you must be able to sign in to GCP as a user who has
[Network Admin](https://cloud.google.com/compute/docs/access/iam#network_admin_role)
permissions. For details, see
[Required Permissions](https://cloud.google.com/vpn/docs/how-to/creating-vpn-dynamic-routes#required_permissions)
in the article
[Creating a VPN Tunnel using Dynamic Routing](https://cloud.google.com/vpn/docs/how-to/creating-vpn-dynamic-routes).

## Configure the GCP side

This section covers how to configure Cloud VPN. This section includes
instructions for configuring static routing.

There are two ways to create VPN gateways on GCP: using the Google Cloud
Platform Console and using the
[gcloud command-line tool](https://cloud.google.com/sdk/).
This section describes how to perform the tasks using the GCP Console. To see
the `gcloud` commands for performing these tasks, see the
[appendix](#appendix-using-gcloud-commands).

### Initial tasks

Complete the following procedures before configuring either a dynamic or static
GCP VPN gateway and tunnel.

#### Select a GCP project name

1. [Open the GCP Console](https://console.google.com).
2. At the top of the page, select the GCP project you want to use.

    **Note**: Make sure that you use the same GCP project for all of the GCP
    procedures in this guide.

#### Create a custom VPC network and subnet

1. In the GCP Console,
[go to the VPC Networks page](https://pantheon.corp.google.com/networking/networks/list).
1. Click **Create VPC network**.
1. For **Name**, enter a name such as `vpn-vendor-test-network`. Remember
this name for later.
1. Under **Subnets, Subnet creation mode**, select the **Custom** tab and
then populate the following fields:

+ **Name**—The name for the subnet, such as `vpn-subnet-1`.
+ **Region**—The region that is geographically closest to the
    on-premises gateway, such as  `us-central1`.
+ **IP address range**—A range such as `10.240.0.0/16`.

5. In the **New subnet** window, click **Done**.
1. Click **Create**. You're returned to the **VPC networks** page, where it
takes about a minute for this network and its subnet to appear.

#### Create the GCP external IP address

1.  In the GCP Console,
[go to the External IP addresses page](https://pantheon.corp.google.com/networking/addresses/list).

1. Click **Reserve Static Address**.
1. Populate the following fields for the Cloud VPN address:

-  **Name**—The name of the address, such as `vpn-test-static-ip`.
    Remember the name for later.
-  **Region**—The region where you want to locate the VPN gateway.
    Normally, this is the region that contains the instances you want to
    reach.
    
4. Click **Reserve**. You are returned to the **External IP addresses** page.
After a moment, the page displays the static external IP address that you
have created.

1. Make a note of the IP address that is created so that you can use it to
configure the VPN gateway later.


### Configuring route-based IPsec VPN using static routing
   This section covers the steps for creating a GCP IPsec VPN using static routing.
#### Configure the VPN gateway

1. In the GCP Console, 
[go to the VPN page](https://console.cloud.google.com/networking/vpn/list).
1. Click **Create VPN connection**.
1. Populate the following fields for the gateway:

-  **Name**—The name of the VPN gateway. This name is displayed in the
    console and used in by the gcloud tool to reference the gateway. Use a
    name like `gcp-to-strongswan`.
-  **Network**—The VPC network that you created previously (for
    example,  `vpn-vendor-test-network`) that contains the instances that the
    VPN gateway will serve.
-  **Region**—The region where you want to locate the VPN gateway.
    Normally, this is the region that contains the instances you want to reach.
-  **IP address**—Select the 
    [static external IP address](#create-the-gcp-external-ip-address)
    (for example, `vpn-test-static-ip`) that you created for this gateway
    in the previous section.
    ![vpn_address_strong_swan-1](https://user-images.githubusercontent.com/42278789/44417291-f0faa380-a529-11e8-9c39-ec5f160bd8d9.png)



#### Configure the VPN tunnel
Populate the fields for at least one tunnel:

-  **Name**—The name of the VPN tunnel, such as `vpn-test-tunnel1`.
-  **Remote peer IP address**—The public external IP address of the
    on-premises VPN gateway.
-  **IKE version**—`IKEv2` or `IKEv1`. IKEv2 is preferred, but IKEv1 is
    supported if it is the only supported IKE version that the on-premises
    gateway can use.
-  **Shared secret**—A character string used in establishing encryption
    for the tunnel. You must enter the same shared secret into both VPN
    gateways. For more information, see
    [Generating a Strong Pre-shared Key](https://cloud.google.com/vpn/docs/how-to/generating-pre-shared-key).


1. In the configuration for a tunnel, under **Routing options**,
    choose **route based**.
1. For **Remote network IP ranges**, set the IP address range or ranges
    of the on-premises network, which is the network on the other side of the
    tunnel from the Cloud VPN gateway you are currently configuring.

1. Click **Create** to create the gateway and initiate all tunnels. This step
automatically creates a network-wide route and the necessary forwarding
rules for the tunnel. The tunnels will not pass traffic until you've
configured the firewall rules.

![screen shot 2018-08-21 at 10 36 16 am](https://user-images.githubusercontent.com/42278789/44418752-1689ac00-a52e-11e8-9cf0-d3f9350f27f2.png)


#### Configure firewall
1. Configure firewall rules to allow inbound traffic from the on-premises
network subnets. You must also configure the on-premises network firewall to
allow inbound traffic from your VPC subnet prefixes.

1. [Go to the Firewall rules page](https://console.cloud.google.com/networking/firewalls).
1. Click **Create firewall rule**.
1. Populate the following fields:
    -  **Name**—A name such as `vpnrule1`.
    -  **VPC network**—The name you used earlier for the VPC network,
        such as `vpn-vendor-test-network`.
    -  **Source filter**—A filter to apply your rule to specific
        sources of traffic. In this case, choose source IP ranges.
    -  **Source IP ranges**—The peer ranges to accept from the peer
        VPN gateway.
    -  **Allowed protocols and ports**—The string `tcp;udp;icmp`.

1. Click **Create**.

## Configure the Strongswan side

### GCP-compatible settings for IPSec and IKE

Configuring the vendor side of the VPN network requires you to use IPsec and IKE
settings that are compatible with the GCP side of the network. The following
table lists settings and information about values compatible with GCP VPN.
Use these settings for the procedures in the subsections that follow.


<table>
<thead>
<tr>
<th><strong>Setting</strong></th>
<th><strong>Description or value</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td>IPsec Mode</td>
<td>ESP+Auth Tunnel mode (Site-to-Site)</td>
</tr>
<tr>
<td>Auth Protocol</td>
<td><code>psk</code></td>
</tr>
<tr>
<td>Shared Secret</td>
<td>Also known as an IKE pre-shared key. Choose a strong password by following
<a
href="https://cloud.google.com/vpn/docs/how-to/generating-pre-shared-key">these
guidelines</a>. The shared secret is very sensitive as it allows access
into your network.</td>
</tr>
<tr>
<td>Start</td>
<td><code>auto</code> (on-premises device should automatically restart the
connection if it drops)</td>
</tr>
<tr>
<td>PFS (Perfect Forward Secrecy)</td>
<td>on</td>
</tr>
<tr>
<td>DPD (Dead Peer Detection)</td>
<td>Recommended: <code>Aggressive</code>. DPD detects when the Cloud VPN
restarts and routes traffic using alternate tunnels.</td>
</tr>
<tr>
<td>INITIAL_CONTACT (sometimes called <i>uniqueids</i>)</td>
<td>Recommended: <code>on</code> (sometimes called <code>restart</code>). The
purpose is to detect restarts faster so that perceived downtime is
reduced.</td>
</tr>
<tr>
<td>TSi (Traffic Selector - Initiator)</td>
<td>Subnet networks: the ranges specified by the GCP local traffic selector. If
no local traffic selector range was specified because the VPN is in an auto-mode
VPC network and is announcing only the gateway's subnet, that subnet range is
used. <br>
<br>
Legacy networks: the range of the network.</td>
</tr>
<tr>
<td>TSr (Traffic Selector - Responder)</td>
<td>IKEv2: The destination ranges of all of the routes that have the next hop
VPN tunnel set to this tunnel on the GCP side.<br>
<br>
IKEv1: Arbitrarily, the destination range of one of the routes that has the
next hop VPN tunnel set to this tunnel on the GCP side.</td>
</tr>
<tr>
<td>MTU</td>
<td>The MTU of the on-premises VPN device must be set to 1460 or lower. ESP
packets leaving the device must not exceed 1460 bytes. You must enable
prefragmentation on your device, which means that packets must be
fragmented first, then encapsulated. For more information, see <a
href="https://cloud.google.com/vpn/docs/concepts/mtu-considerations">Maximum
Transmission Unit (MTU) considerations</a>.</td>
</tr>
<tr>
<td>IKE ciphers</td>
<td>For details about IKE ciphers for IKEv1 or IKEv2 supported by GCP,
including the additional ciphers for PFS, see <a
href="https://cloud.google.com/vpn/docs/concepts/supported-ike-ciphers">Supported
IKE Ciphers</a>.</td>
</tr>
</tbody>
</table>

### Creating the VPN configuration


Follow the procedure listed in the configuration code snippet below to create
the base Layer 3 network configuration of Strongswan . Note the following:

-  At least one internal-facing network interface is required in order to
connect to your on-premises network, and one external-facing interface is
required in order to connect to GCP.

From the home directory, create a file named ipsec.conf under/etc/strongswan directory. Populate it with the following contents, replacing the placeholders with your environment's values:

```
conn myconn
  authby=psk
  auto=start
  dpdaction=hold
  esp=aes256-sha512-modp2048!
  ike=aes256-sha1-modp2048!
  keyexchange=ikev2
  mobike=no
  type=tunnel
  left=%any
  leftid=209.119.81.229
  leftsubnet=192.168.2.0/24
  leftauth=psk
  leftikeport=4500
  right=146.148.83.11
  rightsubnet=10.240.0.0/16
  rightauth=psk
  rightikeport=4500
```
Then, run the following commands, replacing [secret-key] with a secret key (a string value):

```
$ sudo apt-get update
$ sudo apt-get install strongswan -y
$ echo "%any : PSK \"[secret-key]\"" | sudo tee /etc/strongswan/ipsec.secrets > /dev/null
$ sudo sysctl -w net.ipv4.ip_forward=1
$ sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
$ sudo systemctl restart strongswan

```

### Testing the configuration

It's important to test the VPN connection from both sides of a VPN tunnel. For either side, make sure that the subnet that a machine or virtual machine is located in is being forwarded through the VPN tunnel.

First, create VMs on both sides of the tunnel. Make sure that you configure the
VMs on a subnet that will pass traffic through the VPN tunnel.

-  Instructions for creating virtual machines in Compute Engine are located
in the 
[Getting Started Guide](https://cloud.google.com/compute/docs/quickstart).
-  Instructions for creating virtual machine for <vendor-name><product-name>
platforms are located at <link here>.

After VMs have been deployed on both the GCP and <vendor-name><product-name>
platforms, you can use an ICMP echo (ping) test to test network connectivity
through the VPN tunnel.

On the Strongswan side, use the following instructions to test the connection to a
machine that's on GCP

1. Ping a VM machine on GCP.
1. To check on the status of the IKEv2 security associations use the swanctl --list-sas command:
swanctl --list-sas

<img width="615" alt="strongswan-ping test" src="https://user-images.githubusercontent.com/42278789/44420787-ba298b00-a533-11e8-9c1e-6b5dee78db5d.png">




## Troubleshooting IPsec on Strongswan side

If you are experiencing issues with your VPN setup based on the instructions above, try these tips to troubleshoot your setup:
1. Check the status of your connection:

```
sudo strongswan status
```
<img width="803" alt="screen shot 2018-06-14 at 4 11 16 pm" src="https://user-images.githubusercontent.com/42278789/44421355-1a6cfc80-a535-11e8-9920-ae66f83b4423.png">

If myconn isn't listed, start up the connection:
$ sudo strongswan up myconn

<img width="1527" alt="screen shot 2018-06-14 at 4 13 52 pm" src="https://user-images.githubusercontent.com/42278789/44421397-3e304280-a535-11e8-9400-3125641bdfa7.png">

2.Determine whether the two VPN endpoints are able to communicate at all.


Run tcpdump on the receiving end to determine that your VM instance can receive the packets:
tcpdump -nn -n host [public-ip-of-GCP-VPN-gateway] -i any -v
<img width="1365" alt="screen shot 2018-06-14 at 4 37 04 pm" src="https://user-images.githubusercontent.com/42278789/44421564-c7e01000-a535-11e8-9136-ef0331769953.png">

3. Turn on more verbose logging by adding the following lines to your /etc/strongswan/ipsec.conf file:

```
config setup
  charondebug="ike 3, mgr 3, chd 3, net 3"


conn myconn
  authby=psk
  auto=start
  ...
```

Next, retry your connection. Although the connection should still fail, you can check the log for errors. The log file should be located at /var/log/messages on your linux.

<img width="1373" alt="screen shot 2018-06-14 at 4 45 42 pm" src="https://user-images.githubusercontent.com/42278789/44421629-fa8a0880-a535-11e8-91cf-5c5556d6b5b4.png">


## Reference documentation

You can refer to the following <vendor-name><product-name> documentation and
Cloud VPN documentation for additional information about both products.

### GCP documentation

To learn more about GCP networking, see the following documents:

-  [VPC Networks](https://cloud.google.com/vpc/docs)
-  [Cloud VPN Overview](https://cloud.google.com/compute/docs/vpn/overview)
-  [Creating Route-based VPNs](https://cloud.google.com/vpn/docs/how-to/creating-route-based-vpns)
-  [Creating Policy-based VPNs](https://cloud.google.com/vpn/docs/how-to/creating-policy-based-vpns)
-  [Advanced Cloud VPN Configurations](https://cloud.google.com/vpn/docs/concepts/advanced)
-  [Troubleshooting Cloud VPN](https://cloud.google.com/compute/docs/vpn/troubleshooting)

### Strongswan documentation

For more product information on Strongswan, see the following
VPN feature configuration guides and datasheets:

-  [StrongSwan - Documentation](https://www.strongswan.org/documentation.html)


## Appendix: Using gcloud commands

The instructions in this guide focus on using the GCP Console. However, you can
perform many of the tasks for the GPC side of the VPN configuration by using the
[gcloud command-line tool](https://cloud.google.com/sdk/gcloud/). Using `gcloud`
commands can be faster and more convenient if you're comfortable with using a
command-line interface.

### Running gcloud commands

You can run `gcloud` commands on your local computer by installing the [Cloud
SDK](https://cloud.google.com/sdk/). Alternatively, you can run `gcloud`
commands in [Cloud Shell](https://cloud.google.com/shell/), a browser-based
command line. If you use Cloud Shell, you don't need to install the SDK on your
own computer, and you don't need to set up authentication.

**Note**: The `gcloud` commands presented in this guide assume you are working
in a Linux environment. (Cloud Shell is a Linux environment.)

### Configuration parameters and values

The `gcloud` commands in this guide include parameters whose value you must
provide. For example, a command might include a GCP project name or a region or
other parameters whose values are unique to your context. The following table
lists the parameters and gives examples of the values. The section that follows
the table describes how to set Linux environment variables to hold the values
you need for these parameters.

<table>
<thead>
<tr>
<th><strong>Parameter description</strong></th>
<th><strong>Placeholder</strong></th>
<th><strong>Example value</strong></th>
</tr>
</thead>
<tbody>

<tr>
<td>Vendor name</td>
<td><code>[VENDOR_NAME]<code></td>
<td>(Your product's vendor name. This value should have no spaces or
punctuation in it other than underscores or hyphens, because it will be
used as part of the names for GCP entities.)</td>
</tr>

<tr>
<td>GCP project name </td>
<td><code>[PROJECT_NAME]<code></td>
<td><code>vpn-guide<code></td>
</tr>

<tr>
<td>Shared secret</td>
<td><code>[SHARED_SECRET]<code></td>
<td>See <a
href="https://cloud.google.com/vpn/docs/how-to/generating-pre-shared-key">Generating
a Strong Pre-shared Key</a>.</td>
</tr>

<tr>
<td>VPC network name</td>
<td><code>[VPC_NETWORK_NAME]<code></td>
<td><code>vpn-vendor-test-network<code></td>
</tr>

<tr>
<td>Subnet on the GCP VPC network (for example, <code>vpn-vendor-test-network</code>)</td>
<td><code>[VPC_SUBNET_NAME]<code></td>
<td><code>vpn-subnet-1<code></td>
</tr>

<tr>
<td>GCP region. Can be any region, but it should be geographically close to the
on-premises gateway.</td>
<td><code>[REGION]<code></td>
<td><code>us-east1<code></td>
</tr>

<tr>
<td>Pre-existing external static IP address that you configure for the internet
side of the Cloud VPN gateway.</td>
<td><code>[STATIC_EXTERNAL_IP]<code></td>
<td><code>vpn-test-static-ip<code></td>
</tr>

<tr>
<td>IP address range for the GCP VPC subnet (<code>vpn-subnet-1</code>)</td>
<td><code>[SUBNET_IP]<code></td>
<td><code>172.16.100.0/24<code></td>
</tr>

<tr>
<td>IP address range for the on-premises subnet. You will use this range when
creating rules for inbound traffic to GCP.</td>
<td><code>[IP_ON_PREM_SUBNET]<code></td>
<td><code>10.0.0.0/8<code></td>
</tr>

<tr>
<td>External static IP address for the internet interface of <vendor
name><product-name></td>
<td><code>[CUST_GW_EXT_IP]</code> </td>
<td>For example, <code>199.203.248.181</code></td>
</tr>

<tr>
<td>Cloud Router name (for dynamic routing)</td>
<td><code>[CLOUD_ROUTER_NAME]<code></td>
<td><code>vpn-test-vendor-rtr<code></td>
</tr>

<tr>
<td>BGP interface name</td>
<td><code>[BGP_IF]<code></td>
<td><code>if-1<code></td>
</tr>

<tr>
<td>BGP session name (for dynamic routing)</td>
<td><code>[BGP_SESSION_NAME]<code></td>
<td><code>bgp-peer1<code></td>
</tr>

<tr>
<td>The name for the first GCP VPN gateway.</td>
<td><code>[VPN_GATEWAY_1]<code></td>
<td><code>vpn-test-[VENDOR_NAME]-gw-1</code>, where <code>[VENDOR_ NAME]</code>
is the <code>[VENDOR_NAME]</code> string</td>
</tr>

<tr>
<td>The name for the first VPN tunnel for
<code>vpn-test-[VENDOR_NAME]-gw-1</code></td>
<td><code>[VPN_TUNNEL_1]<code></td>
<td><code>vpn-test-tunnel1<code></td>
</tr>

<tr>
<td>The name of a firewall rule that allows traffic between the on-premises
network and GCP VPC networks</td>
<td><code>[VPN_RULE]<code></td>
<td><code>vpnrule1<code></td>
</tr>

<tr>
<td>The name for the <a
href="https://cloud.google.com/sdk/gcloud/reference/compute/routes/create">static
route</a> used to forward traffic to the on-premises network.<br>
<br>
<strong>Note</strong>: You need this value only if you are creating a VPN
using a static route.</td>
<td><code>[ROUTE_NAME]<code>

</td>
<td><code>vpn-static-route<code>

</td>
</tr>
<tr>
<td>The name for the forwarding rule for the <a
href="https://wikipedia.org/wiki/IPsec#Encapsulating_Security_Payload">ESP
protocol</a></td>
<td><code>[FWD_RULE_ESP]<code></td>
<td><code>fr-esp<code></td>
</tr>

<tr>
<td>The name for the forwarding rule for the <a
href="https://wikipedia.org/wiki/User_Datagram_Protocol">UDP
protocol</a>, port 500</td>
<td><code>[FWD_RULE_UDP_500]<code></td>
<td><code>fr-udp500<code></td>
</tr>

<tr>
<td>The name for the forwarding rule for the UDP protocol, port 4500</td>
<td><code>[FWD_RULE_UDP_4500]<code></td>
<td><code>fr-udp4500<code></td>
</tr>

</tbody>
</table>

### Setting environment variables for gcloud command parameters

To make it easier to run `gcloud` commands that contain parameters, you can
create environment variables to hold the values you need, such as your project
name, the names of subnets and forwarding rules, and so on. The `gcloud`
commands presented in this section reference variables that contain your
values.

To set the environment variables, run the following commands at the command line
_before_ you run `gcloud` commands, substituting your own values for all the
placeholders in square brackets, such as `[PROJECT_NAME]`, `[VPC_NETWORK_NAME]`,
and `[SUBNET_IP]`. If you don't know what values to use for the placeholders,
use the example values from the parameters table in the preceding section.

```
export PROJECT_NAME=[PROJECT_NAME]
export REGION=[REGION]
export VPC_SUBNET_NAME=[VPC_SUBNET_NAME]
export VPC_NETWORK_NAME=[VPC_NETWORK_NAME]
export FWD_RULE_ESP=[FWD_RULE_ESP]
export FWD_RULE_UDP_500=[FWD_RULE_UDP_500]
export FWD_RULE_UDP_4500=[FWD_RULE_UDP_4500]
export SUBNET_IP=[SUBNET_IP]
export VPN_GATEWAY_1=[VPN_GATEWAY_1]
export STATIC_EXTERNAL_IP=[STATIC_EXTERNAL_IP]
export VPN_RULE=[VPN_RULE]
export IP_ON_PREM_SUBNET=[IP_ON_PREM_SUBNET]
export CLOUD_ROUTER_NAME=[CLOUD_ROUTER_NAME]
export BGP_IF=[BGP_IF]
export BGP_SESSION_NAME=[BGP_SESSION_NAME]
export VPN_TUNNEL_1=[VPN_TUNNEL_1]
export CUST_GW_EXT_IP=[CUST_GW_EXT_IP]
export ROUTE_NAME=[ROUTE_NAME]
```


### Configuring IPsec VPN using static routing

This section describes how to use the `gcloud` command-line tool to configure
IPsec VPN with static routing. To perform the same task using the GPC Console,
see
[Configuring IPsec VPN using static routing](#configuring-route-based-ipsec-vpn-using-static-routing)
earlier in this guide.

The procedure suggests creating a custom VPC network. This is preferred over
using an auto-created network. For more information, see
[Networks and Tunnel Routing](https://cloud.google.com/vpn/docs/concepts/choosing-networks-routing#network-types)
in the Cloud VPN documentation.

**Note**: Before you run the `gcloud` commands in this section, make sure that
you've set the variables as described earlier under
[Setting environment variables for gcloud command parameters](#setting-environment-variables-for-gcloud-command-parameters).

1. Create a custom VPC network. Make sure there is no conflict with your
local network IP address range. Note the following:

    -  For `[RANGE]`, substitute an appropriate CIDR range, such as
    `172.16.100.0/24`.

    ```
    gcloud compute networks create $VPC_NETWORK_NAME \
        --project $PROJECT_NAME \
        --subnet-mode custom

    gcloud compute networks subnets create $VPC_SUBNET_NAME \
        --project $PROJECT_NAME \
        --network $VPC_NETWORK_NAME \
        --region $REGION \
        --range [RANGE]
    ```

1. Create a VPN gateway in the region you are using. Normally, this is the
region that contains the instances you want to reach.

    ```
    gcloud compute target-vpn-gateways create $VPN_GATEWAY_1 \
        --project $PROJECT_NAME \
        --network $VPC_NETWORK_NAME \
        --region $REGION
    ```

This step creates an unconfigured VPN gateway in your GCP VPC network.

1. Reserve a static IP address in the VPC network and region where you
created the VPN gateway. Make a note of the address that is created for use
in future steps.

    ```
    gcloud compute addresses create $STATIC_EXTERNAL_IP \
        --project $PROJECT_NAME \
        --region $REGION
    ```

1. Create three forwarding rules, one each to forward ESP, IKE, and NAT-T
traffic to the Cloud VPN gateway. Note the following:

    -  For `[STATIC_IP_ADDRESS]`, use the static IP address that you reserved in
    the previous step.

    ```
    gcloud compute forwarding-rules create $FWD_RULE_ESP \
        --project $PROJECT_NAME \
        --region $REGION \
        --ip-protocol ESP \
        --target-vpn-gateway $VPN_GATEWAY_1 \
        --address [STATIC_IP_ADDRESS]

    gcloud compute forwarding-rules create $FWD_RULE_UDP_500 \
        --project $PROJECT_NAME \
        --region $REGION \
        --ip-protocol UDP \
        --target-vpn-gateway $VPN_GATEWAY_1 \
        --ports 500 \
        --address [STATIC_IP_ADDRESS]

    gcloud compute forwarding-rules create $FWD_RULE_UDP_4500 \
        --project $PROJECT_NAME \
        --region $REGION \
        --ip-protocol UDP \
        --target-vpn-gateway $VPN_GATEWAY_1 \
        --ports 4500 \
        --address [STATIC_IP_ADDRESS]
    ``` 

1. Create a VPN tunnel on the Cloud VPN Gateway that points to the external
IP address of your on-premises VPN gateway. Note the following:

-  Set the IKE version. The following command sets the IKE version to
    2, which is the default, preferred IKE version. If you need to set it to
    1, use `--ike-version 1`.
-  For `[SHARED_SECRET]`, supply the shared secret.  For details, see
    [Generating a Strong Pre-shared Key](https://cloud.google.com/vpn/docs/how-to/generating-pre-shared-key).
-  For `[LOCAL_TRAFFIC_SELECTOR_IP]`, supply an IP address range, like
    `172.16.100.0/24`,  that will be accessed on the GCP side of the  tunnel,
    as described in
    [Traffic selectors](https://cloud.google.com/vpn/docs/concepts/choosing-networks-routing#static-routing-networks)
    in the GCP VPN networking documentation.

    ```
    gcloud compute vpn-tunnels create $VPN_TUNNEL_1 \
        --project $PROJECT_NAME \
        --peer-address $CUST_GW_EXT_IP \
        --region $REGION \
        --ike-version 2 \
        --shared-secret [SHARED_SECRET] \
        --target-vpn-gateway $VPN_GATEWAY_1 \
        --local-traffic-selector [LOCAL_TRAFFIC_SELECTOR_IP]
    ``` 

    After you run this command, resources are allocated for this VPN tunnel, but it
    is not yet passing traffic.

1. Use a
[static route](https://cloud.google.com/sdk/gcloud/reference/compute/routes/create)
to forward traffic to the destination range of IP addresses in your
    on-premises network. The region must be the
    same region as for the VPN tunnel.

    ```
    gcloud compute routes create $ROUTE_NAME \
        --project $PROJECT_NAME \
        --network $VPC_NETWORK_NAME \
        --next-hop-vpn-tunnel $VPN_TUNNEL_1 \
        --next-hop-vpn-tunnel-region $REGION \
        --destination-range $IP_ON_PREM_SUBNET
    ```

1. If you want to pass traffic from multiple subnets through the VPN tunnel,
repeat the previous step to forward the IP address of each of the subnets.

1. Create firewall rules to allow traffic between the on-premises network and
GCP VPC networks.

    ```
    gcloud compute firewall-rules create $VPN_RULE \
        --project $PROJECT_NAME \
        --network $VPC_NETWORK_NAME \
        --allow tcp,udp,icmp \
        --source-ranges $IP_ON_PREM_SUBNET
    ```


