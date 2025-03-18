---
title: Setup UFW on Ubuntu
description: >
  Die Ubuntu Firewall.
image: 
  path: https://i.ibb.co/hF3jCC5z/15343120-m.jpg
sitemap: true
categories: [Linux]
tag: [Ubuntu]
comments: false
---

## Setup UFW on Ubuntu

### Installation

[UFW](https://help.ubuntu.com/community/UFW) is installed by default on Ubuntu. If it has been uninstalled for some reason, you can install it with 

```bash
 sudo apt install ufw
```

### **Enable IPv6**

In recent versions of Ubuntu, IPv6 is enabled by default. In practice that means most firewall rules added to the server will include both an IPv4 and an IPv6 version, the latter identified by v6 within the output of UFW's status command. To make sure IPv6 is enabled, you can check your UFW configuration file at /etc/default/ufw. Open this file using nano or your favorite command line editor:

```bash
sudo nano /etc/default/ufw
```

Then make sure the value of IPV6 is set to yes.

### **Setting Up Default Policies**

If you were to enable your UFW firewall now, it would deny all incoming connections. This means that you'll need to create rules that explicitly allow legitimate incoming connections — SSH or HTTP connections, for example — if you want your server to respond to those types of requests. If you're using a cloud server, you will probably want to allow incoming SSH connections so you can connect to and manage your server.

### **Allowing the OpenSSH UFW Application Profile**

Upon installation, most applications that rely on network connections will register an application profile within UFW, which enables users to quickly allow or deny external access to a service. You can check which profiles are currently registered in UFW with:

```bash
sudo ufw app list

Available applications:
  OpenSSH

sudo ufw allow OpenSSH
```

### **Allowing SSH by Service Name**

Another way to configure UFW to allow incoming SSH connections is by referencing its service name: ssh.

```bash
sudo ufw allow ssh
sudo ufw allow http
```

UFW knows which ports and protocols a service uses based on the /etc/services file.

### **Allowing SSH by Port Number**

Alternatively, you can write the equivalent rule by specifying the port instead of the application profile or service name. For example, this command works the same as the previous examples:

```bash
sudo ufw allow 22
```

f you configured your SSH daemon to use a different port, you will have to specify the appropriate port. For example, if your SSH server is listening on port 2222, you can use this command to allow connections on that port.

```bash
sudo ufw allow 2222
```

Bash

### **Specific Port Ranges**

You can specify port ranges with UFW. Some applications use multiple ports, instead of a single port.

```bash
sudo ufw allow 6000:6007/tcp
sudo ufw allow 6000:6007/udp
```

### **Specific IP Addresses or Subnets**

When working with UFW, you can also specify IP addresses within your rules. For example, if you want to allow connections from a specific IP address, such as a work or home IP address of 203.0.113.4, you need to use the from parameter, providing then the IP address you want to allow:

```bash
sudo ufw allow from 203.0.113.4
sudo ufw allow from 203.0.113.0/24
```

You can also specify a port that the IP address is allowed to connect to by adding to any port followed by the port number. For example, If you want to allow 203.0.113.4 to connect to port 22 (SSH), use this command:

```bash
sudo ufw allow from 203.0.113.4 to any port 22
sudo ufw allow from 203.0.113.0/24 to any port 22
```

### **Connections to a Specific Network Interface**

If you want to create a firewall rule that only applies to a specific network interface, you can do so by specifying "allow in on" followed by the name of the network interface.

```bash
sudo ufw allow in on eth0 to any port 80
sudo ufw allow in on eth1 to any port 3306
```

### **Denying Connections**

If you haven't changed the default policy for incoming connections, UFW is configured to deny all incoming connections. Generally, this simplifies the process of creating a secure firewall policy by requiring you to create rules that explicitly allow specific ports and IP addresses through.

However, sometimes you will want to deny specific connections based on the source IP address or subnet, perhaps because you know that your server is being attacked from there. Also, if you want to change your default incoming policy to allow (which is not recommended), you would need to create deny rules for any services or IP addresses that you don't want to allow connections for.

To write deny rules, you can use the commands previously described, replacing allow with deny.

For example, to deny HTTP connections, you could use this command:

```bash
sudo ufw deny http
sudo ufw deny from 203.0.113.4
sudo ufw deny out 25
```

### **Deleting Rules**

Knowing how to delete firewall rules is just as important as knowing how to create them. There are two different ways to specify which rules to delete: by rule number or by its human-readable denomination (similar to how the rules were specified when they were created).

```bash
sudo ufw status numbered

Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22                         ALLOW IN    15.15.15.0/24
[ 2] 80                         ALLOW IN    Anywhere

sudo ufw delete 2

sudo ufw delete allow 80
sudo ufw delete allow http
```

### **Enabling UFW**

Your firewall should now be configured to allow SSH connections. To verify which rules were added so far, even when the firewall is still disabled, you can use:

```bash
sudo ufw show added
sudo ufw enable
sudo ufw disable
sudo ufw reset
sudo ufw status
```
