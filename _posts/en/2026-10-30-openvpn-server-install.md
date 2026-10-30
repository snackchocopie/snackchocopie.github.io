---
title: "[Beginner's Guide] Building Your Own VPN Server with AWS Lightsail + OpenVPN"
lang: en
permalink: /en/openvpn-server-install
author: ramen
date: 2026-10-30 16:50:00 +0900
categories: [OpenVPN]
tags: [vpn, linux, aws, lightsail]
canonical_url: https://qiita.com/snackchocopie/items/50f08dd766ca7c61439b
---

![logo.jpeg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/36bfbdf0-6c3c-4ff6-98d8-574dc2ec2e68.jpeg)

# Building an OpenVPN Server on AWS Lightsail (Basics)

In this post, I will demonstrate an easy-to-follow walkthrough for setting up your own VPN service.

We will use **AWS Lightsail** for the server and **OpenVPN** for the VPN server software.  
Since this is a basic walkthrough, detailed security configurations will not be covered in this article.

## What is Lightsail?

Lightsail is a simple VPS (Virtual Private Server) service provided by AWS.  
It is available for a fixed monthly fee, and each plan includes a set amount of data transfer.  
(Additional charges apply if you exceed the included data transfer allowance.)

![1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/3458e2c5-78d5-4f62-a64d-fd5bd8e4cf56.png)

*AWS Lightsail main dashboard*

Currently, I already have one server in use, and we will create a new server alongside it.


## Creating a Server
![2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/0072d828-6c80-4ec6-b15b-6a4938828b71.png)

*Select Operating System for the platform and Ubuntu 24 for the Blueprint*

As shown in the image, click **Operating System** for the platform and select **Ubuntu 24** for the Blueprint.

Think of a Blueprint as a pre-configured template provided within AWS.


__Click the link marked in red to select a region.__


![3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/65002cff-35bb-4c7e-99a8-82c74d0b172c.png)

Here, we will select the **Oregon** region from the US regions.

![4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/1f219bef-3b50-4495-8efe-bab0d54b1099.png)

For the networking type, select **Dual-Stack**, which enables both IPv4 and IPv6.

If you do not need an IPv4 address, selecting **IPv6-only** is also fine.

For reference, an IPv4 address is the familiar format separated into four numeric blocks, such as `123.123.123.123`.  
On the other hand, IPv6 is a much longer and more complex address format, such as `2001:0db8:85a3:0000:0000:8a2e:0370:7334`.

![5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/5d4d836e-beb2-485b-8044-18e9c5755d88.png)

Give the instance (the unit used to refer to a server) a unique, easy-to-identify name.

A snapshot is a backup image prepared in case of an unexpected failure.  
Check this option if needed. Since this is a simple demonstration, we will leave it unchecked.

**Tags** act like labels used to identify and manage servers created within AWS services.  
Since we only need to distinguish the country here, we will set only the `country` tag key.

![6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/a17fd11c-a56f-4032-b14b-1c08689636b8.png)

Once the instance is created, a public IPv4 address and an IPv6 address are automatically assigned.  

Additionally, one private IPv4 address used within Amazon's private network is also generated.  
(For now, it is enough just to know that such an address exists.)

Because this article is just a simple demonstration, showing these addresses publicly is fine, but in actual production use, it is strongly recommended not to expose them publicly.  
(Doing so could make your server a target for DDoS attacks.)


## Assigning a Static IPv4 Address (Attach)

Unlike IPv6 addresses, an IPv4 address can change every time you stop and start the server.  
This is because IPv4 addresses are an exhausting address space and are systematically managed within AWS.

To keep (fix) your IPv4 address, you need to pin it.  
In Lightsail, this is called a **Static IP**.

![7.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/53b4a03b-2154-4a1f-9174-72d5afe8e0e6.png)
![8.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/2507d6aa-bd06-4a27-a7e8-f0c9dc580b24.png)

*Select Attach Static IP under the Networking tab*


Click **Attach Static IP**, specify your desired name, and confirm.

## Allowing Network Traffic

![9.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/82255c72-19af-4441-9e41-38f0aff1fa2d.png)

By default, **AWS (Lightsail)** blocks all ports except TCP **port 80 (used for HTTP)** and **port 22 (SSH remote connection)**.

Much of the internet traffic we generally use relies on UDP, and OpenVPN—which we are using here—also communicates over UDP by default (though this can be changed).

In the Firewall rules section at the bottom of the Networking tab, open the UDP protocol and the UDP port used to communicate with OpenVPN.

Note that the image above opens everything for demonstration purposes.  
Please keep in mind that in a real service, you should avoid opening all ports and instead operate firewall rules strictly based on the principle of least privilege.

![10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/a2e025a8-0832-4bb0-be9b-4b2483108954.png)

Since the default UDP port used by OpenVPN is **1194**, we will allow this port.

Also, open all allowed IPv4 and IPv6 source IPs as shown below.
 
*Allow UDP port 1194 and all source IPs*

## Issuing an SSH Key for Remote Server Connection

![22.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/d3ba9e64-37f6-4392-ad48-4ac47e95ef7e.png)

As shown in the section in the image above, first download the **SSH key**.  
(Please manage your key securely!)

## Connecting to the Server and Installing the OpenVPN Server

![11.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/1e94c329-f934-41f7-ad82-aa46b65bcdac.png)

Without installing a separate SSH client program, we will connect to the server via the browser using AWS's built-in SSH environment.

Return to the Connect tab and click the **Connect using SSH** button to access the server.



## Easy Installation Using Open-Source OpenVPN-Install

Now let's install the OpenVPN server in earnest.

If you try to install OpenVPN manually from scratch, complex environment configurations can easily lead to spending a lot of time troubleshooting.  
To avoid this from the start, let's take advantage of an easy-to-use open-source tool.

The open-source project we will use is **openvpn-install by angristan**.  
https://github.com/angristan/openvpn-install

Nowadays, there are many excellent open-source projects available, and I believe it is best to actively leverage them whenever a suitable solution exists.

### Update the package manager

```shell
$ sudo apt update
```
### Download the file and grant execution permissions

```shell
$ curl -O https://raw.githubusercontent.com/angristan/openvpn-install/master/openvpn-install.sh
$ chmod +x openvpn-install.sh
```

The `chmod +x` command grants execution permission to the file.

In Linux, files downloaded from external sources using tools like `curl` have execution permissions removed by default for security reasons, so we need to grant execution permission.  
Without execution permission, the file cannot be run.


## Running openvpn-install.sh Interactively
![14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/3651e8ea-477b-4d26-aa48-973886eb4288.png)

As shown in the image above, once execution permission (`x`) is granted, the file name is displayed in green.

Run the file with the following command:

```shell
$ sudo ./openvpn-install.sh interactive
```

Running the command above starts an interactive installation process as shown below.

For the prompts that appear one by one, you can mostly press Enter to proceed with the default values without any issues.  
I also configured most options using the default values.

Below are the answers I selected:

```shell
What IP version should clients use to connect to this server?
> 1 (IPv4)

It seems this server is behind NAT. What is its public IPv4 address or hostname?
> 35.88.182.243

What IP versions should VPN clients use?
> 3 (Dual-stack (IPv4 + IPv6))

IPv4 VPN subnet:
> 1 (Default: 10.8.0.0/24)

IPv6 VPN subnet:
> 1 (Default: fd42:42:42:42::/112)

Route client internet traffic through the VPN? [y/n]
> y

Allow VPN clients to access each other? [y/n]
> n

Allow VPN clients to access the server's local network? [y/n]
> n

What port do you want OpenVPN to listen to?
> 1 (Default: 1194)

What protocol do you want OpenVPN to use?
> 1 (UDP)

What DNS resolvers do you want to use with the VPN?
> 3 (Cloudflare)

Allow multiple devices per client? [y/n]
> n

Do you want to customize the tunnel MTU?
> 1 (Default: 1500)

Choose the authentication mode:
> 1 (PKI)

Do you want to customize encryption settings? [y/n]
> n

Tell me a name for the client.
> tester

How many days should the client certificate be valid for?
> 3650

Do you want to protect the configuration file with a password?
> 2 (Use a password for the client)
```

By the way, for **Route client internet traffic through the VPN?**, make sure to select **Y**.

If you select **N**, all internet traffic will not be routed through the VPN server, which may restrict access when trying to use services from other countries.


## Preparation Before Connecting to the VPN Server

Once all prompts in interactive mode are answered, the script automatically generates the PKI and performs the required setup steps.

Finally, a `.ovpn` file is generated as shown in the image below.  
This file is the key configuration file needed to connect to the OpenVPN server.

We need to download this file to our local computer.  
To do so, we will use a protocol called **SCP**. SCP is a file transfer protocol that utilizes SSH communication over TCP port 22.

![20.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/147888bb-45b1-4bd8-99c8-61c9f37bd63f.png)
![21.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/02a4a558-983f-4a96-a216-6ac431d3674e.png)

```powershell
scp -i C:\Users\me\aws\sshkey.pem ubuntu@35.88.182.243:/home/ubuntu/tester.ovpn C:\Users\me\aws\tester.ovpn
```

In **Windows 11**, SSH and SCP are installed by default.  
If they are not installed on your system, please install them separately.

Run the command above in **PowerShell** or **Command Prompt (CMD)** on your PC.

Of course, the **server IP**, **username**, and **key name** in the example above are sample values from my environment, so please replace them with your own details.


## Testing the Connection with the OpenVPN Client

![24.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/55efb6ac-6eeb-45ff-ad3e-3f6f76b29809.png)

Now let's install the **OpenVPN Client** and test the connection.

The OpenVPN Client is the official client application provided by OpenVPN.  
I recommend using this official client rather than other third-party apps.

Once the app is installed, import the `.ovpn` file we downloaded earlier and try connecting.

![25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/652fa2d3-50c6-45ae-8076-d7132d36eb59.png)
![26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4523380/2018b3f7-97b9-4204-b21b-c9d5b801c20f.png)

You can see that the connection has been established successfully.

Additionally, when checking the location on [mylocation.org](https://mylocation.org), we can confirm that it is properly displayed as the Oregon region.

