---
title: Notes about Linux Firewalld
layout: post
---

Linux had iptables over a decade, but it is probably time for new firewall. We can disagree with this, but Red Hat and other distributions push it to their distributions. Of course, we can still prohibit of using firewalld and we can use iptables, but I tried to deal with new technology and following text contains some basic notes and instructions to configure firewalld in typical server situations.

## Zones ##

Firewalld enables to set different rules for different zones. You can get list of zones with following command:

```bash
firewall-cmd --get-zones
```

with output:

    block dmz drop external home internal public trusted work

and set default zone with following command:

```bash
firewall-cmd --get-default-zone
```

When you creates new VLAN and you are on more friendly LAN, then you can change you zone with following command:

```bash
firewall-cmd --zone=external --change-interface=wlan0
```

## Rich Commands ##

Default Red Hat 7 or CentOS 7 distribution has port 22 opened to the Internet. May be, you will find annoying to see thousands of unsuccessful login attempts after login. First of all it is wise to have strong root password or disable login using password at all. The next thing you can do is limit access to port 22 to some subnet.

> Note: all following commands are used with default zone. It is zone called `public` in this case.

Before we add or remove some rule, then we will list all rules:

```bash
firewall-cmd --list-all
```

The output of previous command could look like this at default CentOS installation:

    public (default, active)
      interfaces: em1
      sources: 
      services: dhcpv6-client ssh
      ports: 
      masquerade: no
      forward-ports: 
      icmp-blocks: 
      rich rules:

When we want to add some more sophisticated rule to SSH service, then we have to add _rich rule_. We want to allow it from some IPv4 subnet and we want to allow only one connection per second:

```bash
firewall-cmd --add-rich-rule='rule family="ipv4" source address="147.230.0.0/16" service name="ssh" accept limit value="1/s"'
```

If we have IPv6 address, then it is recommended to add similar rule for IPv6 subnet:

```bash
firewall-cmd --add-rich-rule='rule family="ipv6" source address="2001:718:1c01::/48" service name="ssh" accept limit value="1/s"'
```

When previous rich rules are set, then we can remove ssh from _services_:

```bash
firewall-cmd --remove-service=ssh
```

The state of commands (listed with `firewall-cmd --list-all`) could look like this:

    public (default, active)
      interfaces: em1
      sources: 
      services: dhcpv6-client
      ports: 
      masquerade: no
      forward-ports: 
      icmp-blocks: 
      rich rules: 
        rule family="ipv6" source address="2001:718:1c01::/48" service name="ssh" accept limit value="1/s"
        rule family="ipv4" source address="147.230.0.0/16" service name="ssh" accept limit value="1/s"

Before we make from previous rules permanent rules it is recommended to test it and connect from allowed subnet. After testing you have to type previous commands once again with option `--permanent`:

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="147.230.0.0/16" service name="ssh" accept limit value="1/s"'
firewall-cmd --permanent --add-rich-rule='rule family="ipv6" source address="2001:718:1c01::/48" service name="ssh" accept limit value="1/s"'
firewall-cmd --permanent --remove-service=ssh
```

Previous commands will save rules to the file `/etc/firewalld/zones/public.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="dhcpv6-client"/>
  <rule family="ipv6">
    <source address="2001:718:1c01::/48"/>
    <service name="ssh"/>
    <accept>
      <limit value="1/s"/>
    </accept>
  </rule>
  <rule family="ipv4">
    <source address="147.230.0.0/16"/>
    <service name="ssh"/>
    <accept>
      <limit value="1/s"/>
    </accept>
  </rule>
</zone>
```

## Own Service ##

When you want to add rule for your own service or service that is not yet supported by firewalld, then you have to several steps. It will be demonstrated for NRPE service. Firs of all you have to create service file. You can copy some existing service file (e.g.: ssh):

```bash
cp /usr/lib/firewalld/services/ssh.xml /etc/firewalld/services/nrpe.xml
```

Modify nrpe.xml using your favorite text editor:

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>NRPE</short>
  <description>Nrpe is a system daemon that will execute various Nagios plugins locally on behalf of a remote (monitoring) host that uses the check_nrpe plugin</description>
  <port protocol="tcp" port="5666"/>
</service>
```

Finally you will have to reload configuration files of firewalld service:

```bash
service firewalld reload
```

or

```bash
systemctl reload firewalld
```

Then you can enable this service at firewall using e.g. rich command:

```bash
firewall-cmd --add-rich-rule='rule family="ipv4" source address="147.230.0.0/16" service name="nrpe" accept limit value="1/s"'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="147.230.0.0/16" service name="nrpe" accept limit value="1/s"'
```

## References ##

[1] https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Using_Firewalls.html

That's all folks. :-)
