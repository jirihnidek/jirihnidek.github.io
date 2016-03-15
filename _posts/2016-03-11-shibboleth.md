---
title: Shibboleth
layout: post
---

[Shibboleth](https://shibboleth.net/) enables to use [Single Sign-on](https://en.wikipedia.org/wiki/Single_sign-on) at your web servers. It is widely used at our university. Thus I decided to write some notes about process of deploying this system at web server. In theory it is possible to use shibboleth with any web server, but only Apache web server is officially supported and this tutorial will focus on configuring of combination Shibboleth plus Apache web server.

## Installation of Shibboleth

First of all you need to install it. Despite it is open-source project you will probably will not find RPM/DEB package in your favorite Linux distribution. Never mind, there are [repositories](https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPLinuxInstall) for most Linux distribution. When you use CentOS 7 like me, you can download configuration of repository using:

```bash
wget -q http://download.opensuse.org/repositories/security://shibboleth/CentOS_7/security:shibboleth.repo
```

Then copy/move downloaded file to `/etc/yum.repos.d/shibboleth.repo`

When you use different version of CentOS/RedHat, then you can find repository file at: [http://download.opensuse.org/repositories/security://shibboleth/](http://download.opensuse.org/repositories/security://shibboleth/).

Install shibboleth using:

```bash
yum -y install shibboleth
```

## Configuration of Shibboleth (SP)

You need to have Apache web server configured and running. It should be available at [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) address using HTTPS protocol. Yes, you need certificate for your server and you can't test shibboleth configuration at virtual machine with private address (e.g. 192.168.1.10). When your organization is member of [CESNET](http://www.cesnet.cz), then you can use [CESNET CA](https://pki.cesnet.cz/) for generating certificate for your server.

Example of apache configuration file (no Shibboleth authentication yet):

    <VirtualHost 147.230.18.79:80>
        ServerName fqdn.address.tul.cz
        ServerAdmin Jiri.Hnidek@tul.cz
        DocumentRoot /var/www/html/fqdn_address_tul_cz/
        ErrorLog /var/log/httpd/fqdn_address_tul_cz.error_log
        CustomLog /var/log/httpd/fqdn_address_tul_cz.access_log common
        Redirect permanent / https://fqdn.address.tul.cz/
    </VirtualHost>

    <VirtualHost 147.230.18.79:443>
        ServerName fqdn.address.tul.cz
        ServerAdmin Jiri.Hnidek@tul.cz
        DocumentRoot /var/www/html/fqdn_address_tul_cz/
        ErrorLog /var/log/httpd/fqdn_address_tul_cz.ssl.error_log
        CustomLog /var/log/httpd/fqdn_address_tul_cz.ssl.access_log common
        # Security
        SSLEngine on
        SSLProtocol all -SSLv2 -SSLv3
        SSLCipherSuite ALL:!ADH:!EXPORT:!SSLv2:RC4+RSA:+HIGH:+MEDIUM:+LOW
        SSLCertificateFile /etc/pki/tls/certs/fqdn_address_tul_cz-cert.pem
        SSLCertificateKeyFile /etc/pki/tls/private/fqdn_address_tul_cz-key.pem
        SSLCACertificateFile /etc/pki/tls/certs/chain_TERENA_SSL_CA_2.pem
        SetEnvIf User-Agent ".*MSIE.*" \
            nokeepalive ssl-unclean-shutdown \
            downgrade-1.0 force-response-1.0
        # Secure directory with Shibboleth
        <Location /secure/>
            AuthType shibboleth
            ShibRequireSession On
            ShibRequestSetting applicationId default
            require valid-user
        </Location>
    </VirtualHost>

You also need to synchronize system time of server with some NTP server. You can use `chronyd` service for this purpose. At CentOS 7 it configured and running by default. Check it using:

```bash
systemctl status chronyd
```

The most complicated thing is editing of shibboleth configuration files. You will probably need to edit two files:

* /etc/shibboleth/shibboleth2.xml (configuration of SP and connection to your IdP)
* /etc/shibboleth/attribute-map.xml (configuration of mapping data from IdP to SP)

It is recommended to use template `/etc/shibboleth/example-shibboleth2.xml`. The content of this file depends on your IdP configuration and you should cooperate with administrator of your IdP server.

File `attribute-map.xml` should contain probably only following change:

```xml
<Attribute name="urn:mace:dir:attribute-def:mail" id="mail"/>
<Attribute name="urn:oid:0.9.2342.19200300.100.1.3" id="mail"/>
```

You can add more attributes to `attribute-map.xml`, but you will have to know your administrator o IdP to send you these attributes in response from IdP.

When you are ready with your `shibboleth2.xml` file and other config files (e.g. `attribute-map.xml`, `template-TUL.xml`), then create following directory and allow group `shibd` to write to this directory:

```bash
mkdir /etc/shibboleth/metadata/
chgrp shibd /etc/shibboleth/metadata/
chmod g+w /etc/shibboleth/metadata/
```

Shibboleth daemon will also need to read your certification file and private key. It is necessary to grant access to key file:

```bash
chmod o-r /etc/pki/tls/private/fqdn_address_tul_cz-key.pem
chgrp shibd /etc/pki/tls/private/fqdn_address_tul_cz-key.pem
chmod g+r /etc/pki/tls/private/fqdn_address_tul_cz-key.pem
```

, then you have to restart shibboleth daemon and apache server:

```bash
systemctl restart shibd
systemctl restart httpd
```

Then you should be able to get metadata from URL. Of course you can use other web client, then wget :-):

```bash
wget https://fqdn.address.tul.cz/Shibboleth.sso/Metadata
```

Content of this file send to your IdP administrator and he or she will recreate and sign batch of metadata. It will be available at address defines in `shibboleth2.xml` file:

```xml
<!-- Example of remotely supplied batch of signed metadata. -->
<MetadataProvider type="XML" validate="true"
      uri="https://shibbo.tul.cz/metadata/tul-metadata.xml"
      backingFilePath="/etc/shibboleth/metadata/tul-metadata.xml" reloadInterval="7200">
</MetadataProvider>
```

When your server is listed in https://shibbo.tul.cz/metadata/tul-metadata.xml, then your server is ready for authentication using Shibboleth.

## Testing of Shibboleth (SP)

Optionally you want to check configuration of your SP, then you have to white-list IP of your computer in `shibboleth2.xml`:

```xml
<!-- Status reporting service. -->
<Handler type="Status" Location="/Status" acl="127.0.0.1 ::1 147.230.17.49 2001:718:1c01:17:92b1:1cff:fe9a:449f"/>
```
Then you can reach status report at address:

```bash
wget https://fqdn.address.tul.cz/Shibboleth.sso/Status
```

Information about session is available at URL:

```bash
wget https://fqdn.address.tul.cz/Shibboleth.sso/Session
```

## References

[1] [http://www.eduid.cz/cs/tech/sp/shibb-2.4](http://www.eduid.cz/cs/tech/sp/shibb-2.4)