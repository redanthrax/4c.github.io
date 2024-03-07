+++
title = 'Azure Hosted Filebeat Syslog'
date = 2022-05-16T09:12:23-08:00
draft = false
summary = 'Setup all-in-one syslog filebeat relay server on Azure'
+++

# Logshipper
Below are instructions for configuring multiple filebeat processes to listen for
Syslog over multiple ports and sending those logs securely to a kibana service
(in our case Perch).

The hardware used are Meraki firewalls configured with a Site-to-Site VPN
connection directly to the server.

### Prereqs
This document assumes you’ve setup SELinux on Debian and are in passive mode.

# VPN Server Setu

### Initial Azure Configuration
- Navigate to the Network Interface of the VM and Enable IP Forwarding
- Open UDP Port 500 and UDP Port 4500 in the network security group
- Change the private IP address from Dynamic to Static (usually 10.)
- Create a Route Table in the resource group
- Add a route to match the firewall private subnet to the VM private IP address

```text
Client Subnet: 192.168.100.0/24
VM IP: 10.0.0.4
```

- Associate the Route Table Subnet with the VM subnet.

### Debian Configuration
- Install Strongswan

```bash
sudo apt install strongswan -y
```

- Set the following kernel parameters in /etc/sysctl.conf

```text
net.ipv4.ip_forward = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv6.conf.tun0.disable_ipv6 = 1
```

- Load the sysctl file

```bash
sudo sysctl -p /etc/sysctl.conf
```

- Edit the global configuration /etc/ipsec.conf

```text
# ipsec.conf - strongSwan IPsec configuration file
# basic configuration

config setup
    charondebug="all"
    uniqueids=yes
    strictcrlpolicy=no

conn %default
    ikelifetime=1440m
    rekeymargin=3m
    keyingtries=%forever
    keyexchange=ikev1
    authby=secret
    dpdaction=restart
    dpddelay=30

conn client1
    left=%defaultroute
    leftsubnet=10.0.0.0/24 #Azure VM Subnet
    leftid=20.xxx.xxx.28 #Azure VM Public IP
    leftfirewall=yes
    right=203.xxx.xxx.242 #Remote Meraki MX IP
    rightsubnet=192.168.100.0/24 #Remote MX Subnet
    rightid=203.xxx.xxx.242
    auto=add
    ike=aes256-sha1-modp1024
    esp=aes256-sha1
    keyexchange=ikev1
```

- Generate a Pre-Shared key

```bash
head -c 24 /dev/urandom | base64
```

- Set the IPSec Pre-Shared Key by editing /etc/ipsec.secrets

```text
# VMPublicIP   MXPublicIP
20.xxx.xxx.28 203.xxx.xxx.242 : PSK "YourPreSharedKey!"
```

- Start the service on boot

```bash
sudo systemctl enable strongswan-starter
```

- Start the service

```bash
sudo systemctl start strongswan-starter
```

- Start the VPN

```bash
sudo ipsec restart
```

- Start the VPN tunnel

```bash
sudo ipsec up client1
```

- Get the status of the tunnel

```bash
sudo ipsec status
```

### Meraki Configuration
- Setup a new Site-to-Site configuration
- Select IKEv1
- Use an IPSEC Policy with the following settings
```text
Preset: Custom
Phase 1
Encryption: AES 256
Authentication: SHA1
Diffie-Hellman group: 2
Lifetime: 28800
Phase 2
Encryption: AES 256
Authentication: SHA1, MD5
PFS group: Off
Lifetime: 3600
```

- Set the Private subnets to the same as the VM private subnet
- Set the availability to All networks or a tag specified for just the firewall
- Check the connection of the VPN using the following command

```bash
sudo ipsec status
```

You should now see a connection established.

### Additional Client VPN Configuration
- Add a route to the route table as client1.
- Add additional conn client2 block to /etc/ipsec.conf

```text
conn client2
    left=%defaultroute
    leftsubnet=10.0.0.0/24 #Azure VM Subnet
    leftid=20.xxx.xxx.28 #Azure VM Public IP
    leftfirewall=yes
    right=202.xxx.xxx.242 #Remote Meraki MX IP
    rightsubnet=192.168.200.0/24 #Remote MX Subnet
    rightid=202.xxx.xxx.242
    auto=add
    ike=aes256-sha1-modp1024
    esp=aes256-sha1
```

- Generate a Pre-Shared key

```bash
head -c 24 /dev/urandom | base64
```

- Set the IPSec Pre-Shared Key by editing /etc/ipsec.secrets

```text
# VMPublicIP   MXPublicIP
20.xxx.xxx.28 203.xxx.xxx.242 : PSK "YourPreSharedKey!"
```

- Configure the connection on the Meraki FW
- Bring up the connection

```bash
sudo ipsec restart
sudo ipsec up client2
```

- Validate the connection status

```bash
sudo ipsec status
```

### Setup and configure Filebeat
- Find a link to the latest version of Filebeat from Debian:
[Download Filebeat](https://www.elastic.co/downloads/beats/filebeat)
- Download the Filebeat deb on the Debian server

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.3.1-amd64.deb
```

- Install filebeat

```bash
sudo dpkg -i filebeat-8.3.1-amd64.deb
```

- Enable the system module

```bash
sudo filebeat modules enable system
```

- Enable the cisco module

```bash
sudo filebeat modules enable cisco
```

- Create the config directory

```bash
sudo mkdir /etc/filebeat/configs
```

- Create the client configuration for filebeat
- Edit /etc/filebeat/configs/client1.yml
- Change the syslog_port to an unused port

```text
filebeat.config:
	modules:
		enabled: true
		path: modules.d/*.yml
		reload.enabled: true
		reload.period: 10s
output.elasticsearch:
    hosts: ["ingest.perchsecurity.com:443/elastic"]
    headers:
    X-Perch-Header: "Perch API Key"
    protocol: "https"
    compression_level: 5
	allow_older_versions: true
migration.6_to_7.enabled: true
filebeat.modules:
- module: cisco
	meraki:
	enabled: true
	var.syslog_host: 0.0.0.0
	var.syslog_port: 42000
	var.log_level: 5
```

- Create the Filebeat data directory for clients

```bash
sudo mkdir -p /opt/filebeat/client1
```

- Test the configuration

```bash
sudo filebeat -e -c /etc/filebeat/configs/client1.yml --path.data /opt/filebeat/client1
```

- Create the service file

```bash
sudo vi /etc/systemd/system/filebeat-client1.service
```

```text
[Unit]
Description=Filebeat for Client1

[Service]
User=root
WorkingDirectory=/usr/share/filebeat
ExecStart=filebeat -e -c /etc/filebeat/configs/client1.yml --path.data /opt/filebeat/client1
Restart=always

[Install]
WantedBy=multi-user.target
```

- Reload the daemon service

```bash
sudo systemctl daemon-reload
```

- Enable and start the service

```bash
sudo systemctl enable filebeat-client1.service && sudo systemctl start filebeat-client1.service
```

### Adding additional clients
- Create a new client data directory

```bash
sudo mkdir /opt/filebeat/client2
```

- Copy the existing Filebeat config to a new client

```bash
sudo cp /etc/filebeat/configs/client1.yml /etc/filebeat/configs/client2.yml
```

- Edit the new config and change the syslog port and API key
- Copy the filebeat service file from another client

```bash
sudo cp /etc/systemd/system/filebeat-client1.service /etc/systemd/system/filebeat-client2.service
```

- Edit the new service file and specify a different config and client data directory
- Reload the daemon, enable the service, start the service

```bash
sudo systemctl daemon-reload && sudo systemctl enable filebeat-client2.service && sudo systemctl start filebeat-client2.service
```

### Send Meraki Logs
- Navigate to Network Wide -> General -> Configure in the Meraki Dashboard
- Add a syslog server for the LAN IP of the log shipper server
- Select all logs
- Validate the logs are ingesting into your logging software

### Post Setup
- Take the necessary steps in SELinux to button up the system
