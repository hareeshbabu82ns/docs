# [KeyBox](http://sshkeybox.com/)

## Installing & Starting
check the instructions in [KeyBox Git](https://github.com/skavanagh/KeyBox#keybox)

## NGINX server reverse proxy config
```bash
server {
		listen 443;
		server_name keybox.hlinode.cloudns.cc;

		ssl_certificate           /etc/nginx/cert.crt;
		ssl_certificate_key       /etc/nginx/cert.key;

		ssl on;
		ssl_session_cache  builtin:1000  shared:SSL:10m;
		ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
		ssl_ciphers HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
		ssl_prefer_server_ciphers on;

		access_log            /var/log/nginx/keybox.access.log;

		location / {
				proxy_http_version 1.1;
				proxy_set_header Upgrade $http_upgrade;
				proxy_set_header Connection "upgrade";

				proxy_set_header        Host $host;
				proxy_set_header        X-Real-IP $remote_addr;
				proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
				proxy_set_header        X-Forwarded-Proto $scheme;

				# Fix the “It appears that your reverse proxy set up is broken" error.
				proxy_pass          https://localhost:8443/;
				proxy_read_timeout  1800s;

				proxy_redirect      http://localhost:8443 https://keybox.hlinode.cloudns.cc;
		}
}
```
## NGINX server reverse proxy config with HTTP
* in the jetty directory edit the **start.ini** file
* change --module=https to **--module=http**
* and change jetty.http.port=8443 to and change **jetty.http.port=8888**
* add config to NGINX servers.conf file to proxy for port 8888
```
server {                                                                                         
        listen 80;                                                                               
        server_name keybox.hlinode.cloudns.cc;                                                   
        location / {                                                                             
                proxy_http_version 1.1;                                                          
                proxy_set_header Upgrade $http_upgrade;                                          
                proxy_set_header Connection "upgrade";                                           
                                                                                                 
                proxy_set_header Host $host;                                                     
                proxy_set_header X-Real-IP $remote_addr;                                         
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;                     
                proxy_set_header X-Forwarded-Proto $scheme;                                      
                                                                                                 
                proxy_pass http://localhost:8888;                                                
                proxy_read_timeout 1800s;                                                        
                                                                                                 
                proxy_redirect http://localhost:8888 http://keybox.hlinode.cloudns.cc;           
        }                                                                                        
}            
```

## Apache2 server reverse proxy config
```
<VirtualHost *:443>
  ServerName keybox.hlinode.cloudns.cc

  ## Logging
  ErrorLog "/var/log/httpd/example_error_ssl.log"
  ServerSignature Off
  CustomLog "/var/log/httpd/example_access_ssl.log" combined    


 ## SSL directives
  SSLEngine on
  SSLProxyEngine On
  SSLProxyCheckPeerCN off
  SSLProxyCheckPeerName off
  SSLProxyVerify none
  SSLProxyCheckPeerExpire off    

  SSLCertificateFile      "/etc/ssl/mycert.crt"
  SSLCertificateKeyFile   "/etc/ssl/mykey.key"
  SSLCACertificatePath    "/etc/pki/tls/certs"    

  ## Proxy rules
  ProxyRequests off
  ProxyPreserveHost On
  ProxyPass / https://localhost:8443/
  ProxyPassReverse / https://localhost:8443/    

  <LocationMatch "/admin/(terms.*)">
        ProxyPass wss://127.0.0.1:8443/admin/$1
        ProxyPassReverse wss://127.0.0.1:8443/admin/$1
  </LocationMatch>    

  RequestHeader set X-Forwarded-Proto "https" env=HTTPS    

</VirtualHost>
```

## Config Step By Step

### Install JDK
* download jdk
* unzip using ```tar -xvf jdk*.tar.gz```
* add jdk to path
  * ```nano ~/.bashrc```
  * export PATH="$HOME/tools/jdk1.8.0_131/bin:$PATH"
  * export JAVA_HOME="$HOME/tools/jdk1.8.0_131"

### Install KeyBox
* download and unzip
* add to path
  * export PATH="$HOME/tools/KeyBox-jetty/bin:$HOME/tools/KeyBox-jetty/jetty/bin:$PATH"
* run ```./startKeyBox.sh``` within KeyBox home folder
* provide Database key
* save SSH key generated from initialization to **~/.ssh/authorized_keys** file
  * Note: this SSK key should be copied to any Linux system to which KeyBox needs to be connected
* KeyBox will be running on **https://localhost:8443** by default

### Setup Nginx to serve KeyBox
* goto /etc/nginx folder
* generate SSL certificate
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/cert.key -out /etc/nginx/cert.crt
```
* open /etc/nginx/conf.d/servers.conf file
* add [SSL Config](https://github.com/hareeshbabu82ns/docs/blob/master/nginx.md#server-with-ssl-https-and-ssh-tunnel-ex-keybox)
* restart nginx (sudo service nginx restart)
* replace server name to **keybox.<domain>**

### Setup KeyBox via Web
* Access KeyBox using https://keybox.hgcloud.cloudns.cc
* configure **admin** user (initial password **changeme**)
* add Systems, Profiles, Users
* assign Users to Profiles and Systems to Profiles
* login with new User and connect with System using SSH terminal over HTTPS
