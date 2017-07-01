# Installing Apache
```bash
$> sudo apt-get install apache2 apache2-doc apache2-utils
```

## Important Directories/Files
* /etc/apache2
* /etc/apache2/apache2.conf
* /etc/apache2/sites-available/<site>.conf

## Working with Apache
* sudo service apache2 start
* sudo service apache2 stop
* sudo service apache2 restart
* sudo service apache2 status

## Working with Apache Virtual Hosts
* sudo a2dissite 000-default.conf - Disable the default Apache virtual host
* sudo a2ensite custom.conf - Enable Custom Site Config

## Working with Apache Modules
* sudo apt-cache search libapache2* - list of available modules
* sudo apt-get install [module-name]
* sudo a2enmod [module-name] - enable module
* a2dismod [module-name] - disable module

## Sample Virtual Server Config file - under sites-available directory
```
<VirtualHost *:80>
     ServerAdmin webmaster@example.com
     ServerName example.com
     ServerAlias www.example.com
     DocumentRoot /var/www/example.com/public_html/
     ErrorLog /var/www/example.com/logs/error.log
     CustomLog /var/www/example.com/logs/access.log combined
</VirtualHost>
```
* Note: create **public_html** and **logs** directories under Website Source Directory

## Prepare for Reverse Proxy
* Install modules necessary
```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_balancer
sudo a2enmod lbmethod_byrequests
sudo systemctl restart apache2
```
* Few more Modules 
  * mod_proxy_ftp - for FTP
  * mod_proxy_connect - for SSL tunneling
  * mod_proxy_wstunnel - for WebSockets

* Sample Default Host Config file bootstrap.conf under /etc/apache2/sites-available
* this will serve a static file from /var/www/bootstrap location of the server
```
<VirtualHost *:80>
     ServerAdmin webmaster@hcloud.cloudns.com
     ServerName www.hcloud.cloudns.com
     ServerAlias hcloud.cloudns.com
     DocumentRoot /var/www/bootstrap/
     ErrorLog /var/www/bootstrap/logs/error.log
     CustomLog /var/www/bootstrap/logs/access.log combined
</VirtualHost>
```
* Sample Virtual Host Config file servers.conf under /etc/apache2/sites-available
```
<VirtualHost *:80>
    ServerName app.hcloud.cloudns.cc

    ProxyPreserveHost On
    ProxyPass / http://localhost:4000/
    ProxyPassReverse / http://localhost:4000/
</VirtualHost>

<VirtualHost *:80>
    ServerName jsondb.hcloud.cloudns.cc

    ProxyPreserveHost On
    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000/
</VirtualHost>
```
## Working with Apache Virtual Hosts for SSL (HTTPS @ PORT 443)
* Create Self-signed SSL certificate
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```
* Create DH Group to increase strength of security
```
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```
* Create an Apache Configuration Snippet with Strong Encryption Settings
```
sudo nano /etc/apache2/conf-available/ssl-params.conf
```
```
# from https://cipherli.st/
# and https://raymii.org/s/tutorials/Strong_SSL_Security_On_Apache2.html

SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3
SSLHonorCipherOrder On
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
#Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains; preload"
Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
# Requires Apache >= 2.4
SSLCompression off 
SSLSessionTickets Off
SSLUseStapling on 
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"

SSLOpenSSLConfCmd DHParameters "/etc/ssl/certs/dhparam.pem"
```
* Modify default ssl host config **/etc/apache2/sites-available/default-ssl.conf**
```
  ServerAdmin <your_email@example.com>
  ServerName <server_domain_or_IP>
  
  ...
  
  SSLCertificateFile      /etc/ssl/certs/apache-selfsigned.crt
  SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

  ...

  BrowserMatch "MSIE [2-6]" \
     nokeepalive ssl-unclean-shutdown \
     downgrade-1.0 force-response-1.0
```

* Modify default Virtual Host to redirect to HTTPS
```
# /etc/apache2/sites-available/000-default.conf
<VirtualHost *:80>
        . . .

        Redirect permanent "/" "https://your_domain_or_IP/"

        . . .
</VirtualHost>
```
* Adjust Firewall settings
```
sudo ufw app list
sudo ufw status
sudo ufw allow 'Apache Full'
sudo ufw delete allow 'Apache'
sudo ufw enable
sudo ufw status
```

* Enable SSL modules for Apache
```
sudo a2enmod ssl
sudo a2enmod headers
```
* Enable Default SSL site
```
sudo a2ensite default-ssl
```
* Enable our custom Config
```
sudo a2enconf ssl-params
```
* Test the Apache Config and Restart
```
sudo apache2ctl configtest
sudo systemctl restart apache2
```
* Sample SSL file - default-ssl.conf
```
<IfModule mod_ssl.c>

        <VirtualHost _default_:443>
                ServerAdmin hareeshbabu82@gmail.com
                ServerName hcloud.cloudns.cc

                DocumentRoot /var/www/bootstrap/

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                SSLEngine on

                SSLCertificateFile      /etc/ssl/certs/apache-selfsigned.crt
                SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

                <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                SSLOptions +StdEnvVars
                </FilesMatch>
                <Directory /usr/lib/cgi-bin>
                                SSLOptions +StdEnvVars
                </Directory>

                BrowserMatch "MSIE [2-6]" \
                        nokeepalive ssl-unclean-shutdown \
                        downgrade-1.0 force-response-1.0

        </VirtualHost>
        <VirtualHost *:443>
                ServerAdmin hareeshbabu82@gmail.com
                ServerName keybox.hcloud.cloudns.cc

                #DocumentRoot /var/www/bootstrap/

                ErrorLog ${APACHE_LOG_DIR}/keybox.error.log
                CustomLog ${APACHE_LOG_DIR}/keybox.access.log combined

                SSLEngine on
                SSLProxyEngine On
                SSLProxyCheckPeerCN off
                SSLProxyCheckPeerName off
                SSLProxyVerify none
                SSLProxyCheckPeerExpire off

                SSLCertificateFile      /etc/ssl/certs/apache-selfsigned.crt
                SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

                ## Proxy rules
                ProxyRequests off
                ProxyPreserveHost On

                ProxyPass / https://localhost:8443/
                ProxyPassReverse / https://localhost:8443/


                <LocationMatch "/admin/(terms.*)">
                        ProxyPass ws://127.0.0.1:8443/admin/$1
                        ProxyPassReverse ws://127.0.0.1:8443/admin/$1

                        ProxyPass wss://127.0.0.1:8443/admin/$1
                        ProxyPassReverse wss://127.0.0.1:8443/admin/$1
                </LocationMatch>
        
                        RequestHeader set X-Forwarded-Proto "https" env=HTTPS
        
        </VirtualHost>
        
</IfModule>                                                
```
* Test connection with https://<site>/

## Link/Docs
* [LiNode](https://www.linode.com/docs/web-servers/apache/apache-web-server-on-ubuntu-14-04)
* [DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-use-apache-as-a-reverse-proxy-with-mod_proxy-on-ubuntu-16-04)
* [SSL Config](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-16-04)

