# Firewall-CMD Cheatsheet

> Note that most of the commands above will use the 'Public' Zone. This will need to be tailored to the zone you wish to manage. 
{.is-info}

<br>

## Managing firewall-cmd

<br>

## Display whether firewall-cmd is running
```bash
firewall-cmd --state
```
### Restart firewall-cmd
```bash
systemctl restart firewall-cmd
```
### Reload firewall-cmd
```bash
firewall-cmd --reload 
```

<br>

## Control Startup at Boot

<br>

### Enable firewall-cmd
```bash
systemctl enable firewalld
```
### Disable firewall-cmd
```bash
systemctl disable firewalld
```

<br>

## List Default and Active Zones

<br>

### Get Default Zone
```bash
firewall-cmd --get-default-zone
```
### Get Active Zone
```bash
firewall-cmd --get-active-zones
```
### List All Zones
```bash
firewall-cmd --list-all
```

<br>

## Interface Control

<br>

### Add an Interface to a Zone
```bash
firewall-cmd --zone=public --change-interface=eth1 --permanent
```

### Remove an Interface to a Zone
```bash
firewall-cmd --zone=public --remove-interface=eth1 --permanent
```

<br>

## Port and Service control for Zones

<br>

### List Services Globally
```bash
firewall-cmd --get-services
```
### List Services for a Zone
```bash
firewall-cmd --zone=public --list-service
```
### Add Services to a Zone
```bash
firewall-cmd --zone=public --add-service=samba --add-service=samba-client --permanent
```
### Remove Services from a Zone
```bash
firewall-cmd --zone=public --remove-service=samba --add-service=samba-client --permanent
```
### List Ports from a Zone
```bash
firewall-cmd --list-ports
```
### Add Ports to a Zone
```bash
firewall-cmd --zone=public --add-port=5000/tcp --permanent
```

<br>

## Creation of Zone, Adding IP, and Adding specific rules

<br>

### Create Zone
```bash
firewall-cmd --new-zone=special --permanent
firewall-cmd --reload
```
### Adding a Source to the Zone
```Bash
firewall-cmd --zone=special --add-source=192.0.2.4/32  --permanent
```
### Adding a Port to the Zone
```Bash
firewall-cmd --zone=special --add-port=4567/tcp --permanent
```
