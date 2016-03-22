---
title: Certificates and Web Servers
layout: post
---

This post includes simple tutorial of generating and installation of "classical" certificate at web server ([apache](http://www.apache.org/) and [nginx](http://nginx.org/)) running at CentOS 7 Linux. It is focused on generating certificates using [pki.cesnet.cz](https://pki.cesnet.cz), but it can be useful with other certificate authorities.

Requirements
------------

You have to have running Linux server. You can use many web server, but I prefer o use Apache or Nginx. You have to enable port 80 (protocol HTTP) and 443 (protocol HTTPS). You can do it using:

```bash
firewall-cmd --add-service=http
firewall-cmd --add-service=https
```

To check results type:

```bash
firewall-cmd --list-all
```

It should include something like this:

    public (default, active)
      interfaces: em1
      sources: 
      services: dhcpv6-client ssh http https
      ports: 
      masquerade: no
      forward-ports: 
      icmp-blocks: 
      rich rules:

You will probably want to add previous firewall rules a permenent rules:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
```

## Install and Start Apache

If you want to install and run ([apache](http://www.apache.org/) web server, then you can install it using:

```bash
yum -y install httpd mod_ssl openssl
systemctl enable httpd
systemctl start httpd
```

## Install and Start Nginx

If your prefer [nginx](http://nginx.org/), you can install, enable and start it using similar commands:

```bash
yum -y install nginx openssl
systemctl enable httpd
systemctl start httpd
```

Generating of Certificate
-------------------------

At the first time you will need to create request of certificate. You can use on-line tool at address: [https://tcs.cesnet.cz/requestconfig/](https://tcs.cesnet.cz/requestconfig/) for this purpose or you can use following example. Configuration file named `server-req.cfg` for generating request file could look like this:

    default_bits            = 2048
    distinguished_name      = req_distinguished_name
    string_mask             = nombstr
    prompt                  = no
    req_extensions          = req_ext

    [req_distinguished_name]
    organizationName        = Technical University of Liberec
    commonName              = domain

    [req_ext]
    subjectAltName          = @san

    [san]
    DNS.1                   = domain
    DNS.2                   = domain.ip6
    DNS.3                   = example
    DNS.4                   = example.ip6

Then you can create own request file using following command (you will be asked for passphrase for `serverkey.pem`):

```bash
openssl req -new -keyout serverkey.pem -out serverreq.pem -config server-req.cfg
```

When your request is created, then you can submit it on address: [https://tcs.cesnet.cz/requestform/form](https://tcs.cesnet.cz/requestform/form).

Keep in mind that `serverkey.pem` will be encrypted (using passphrase you typed) and encrypted secret key file is not feasible for production, because web server would require to type this passphrase everytime, when web server is started or restarted. For this reason it is wise to decrypt `serverkey.pem` and grant feasible access permissions for this file:

```bash
openssl rsa -in serverkey.pem -out domain_tul_cz.key.pem
chmod go-rwx domain_tul_cz.key.pem
```

When certificate file will be generated, then you will receive notification e-mail with link to certificate file and you will be able to download it to your server. You will also need CA chain. It will be probably this file:

```bash
wget -q https://pki.cesnet.cz/certs/chain_TERENA_SSL_CA_3.pem
```

When you will want to use Nginx web server, then you will also need to create bundled certificate file, because nginx can't use certificate file and CA chain file for some reasons. You have to place them into one file using these commands:

```bash
cp domain_tul_cz.cert.pem domain_tul_cz.bundle.cert.pem
cat chain_TERENA_SSL_CA_3.pem >> domain_tul_cz.bundle.cert.pem
```

When you have all required files (`domain_tul_cz.cert.pem`, `domain_tul_cz.key.pem` and `chain_TERENA_SSL_CA_3.pem`), then you should copy it to appropriate directories:

```bash
cp domain_tul_cz.cert.pem /etc/pki/tls/certs/            # Certificate
cp chain_TERENA_SSL_CA_3.pem /etc/pki/tls/certs/         # CA chain
cp domain_tul_cz.bundle.cert.pem /etc/pki/tls/certs/     # Optionally
cp domain_tul_cz.key.pem /etc/pki/tls/private/           # Secret key
chmod go-rwx /etc/pki/tls/private/domain_tul_cz.key.pem  # Make sure it is secret
```

## Configuration of Apache

Configuration of Apache web server could look like this. I add configuration for virtual web server to own configuration file in directory `/etc/httpd/conf.d`. For hostname `domain.tul.cz` I would create configuration `domain_tul_cz.conf`:

    <VirtualHost 147.230.1.1:80>
        ServerName domain.tul.cz
        ServerAdmin Jiri.Hnidek@tul.cz
        ErrorLog /var/log/httpd/domain_tul_cz.error_log
        CustomLog /var/log/httpd/domain_tul_cz.access_log common
        RewriteEngine on
        RewriteCond %{HTTPS} !=on
        RewriteRule .* https://%{SERVER_NAME}%{REQUEST_URI} [NE,R,L]
    </VirtualHost>

    <VirtualHost 147.230.1.1:443>
        ServerName gitlab.nti.tul.cz
        ServerAdmin Jiri.Hnidek@tul.cz
        DocumentRoot /var/www/html/domain_tul_cz/
        # Logs
        ErrorLog /var/log/httpd/domain_tul_cz.ssl.error_log
        CustomLog /var/log/httpd/domain_tul_cz.ssl.access_log common
        # Security
        SSLEngine on
        SSLProtocol all -SSLv2 -SSLv3
        SSLCipherSuite "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4"
        Header add Strict-Transport-Security: "max-age=15768000;includeSubdomains"
        SSLCompression Off
        SSLCertificateFile /etc/pki/tls/certs/domain_tul_cz.cert.pem
        SSLCACertificateFile /etc/pki/tls/certs/chain_TERENA_SSL_CA_3.pem
        SSLCertificateKeyFile /etc/pki/tls/private/domain_tul_cz.key.pem
        SetEnvIf User-Agent ".*MSIE.*" \
             nokeepalive ssl-unclean-shutdown \
             downgrade-1.0 force-response-1.0
    </VirtualHost>

This configuration "redirect" any request from port 80 (HTTP) to port 443 (HTTPS). You can also add some `index.html` file to directory `/var/www/html/domain_tul_cz/`. After any change of apache configuration file, you have to reload configuration files:

```bash
systemctl reload httpd
```

## Configuration of Nginx

Configuration files for virtual web server are placed in `/etc/nginx/conf.d` and here is example of `domain_tul_cz.conf`:

    server {
      listen       *:80;
      server_name  domain.tul.cz;
      rewrite      ^ https://$server_name$request_uri? permanent;
    }

    server {
      listen *:443;

      server_name domain.tul.cz;
      server_tokens off; ## Don't show the nginx version number, a security best practice
      root /var/www/html/domain_tul_cz/

      # SSL stuff
      ssl                        on;
      ssl_certificate            /etc/pki/tls/certs/domain_tul_cz-bundle-cert.pem;
      ssl_certificate_key        /etc/pki/tls/private/domain_tul_cz-key.pem;
      ssl_protocols              TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers                RC4:HIGH:!aNULL:!MD5;
      ssl_prefer_server_ciphers  on;

      ## Increase this if you want to upload large attachments
      ## Or if you want to accept large git objects over http
      client_max_body_size 0;

      ## Individual nginx logs for this GitLab vhost
      access_log  /var/log/nginx/domain_tul_cz_access.log gitlab_access;
      error_log   /var/log/nginx/domain_tul_cz_error.log;
    }

## Testing of Web Server Configuration

To test your web server use [https://www.ssllabs.com/](https://www.ssllabs.com/). It will show missing CA chain, dangerous ciphers and much more.