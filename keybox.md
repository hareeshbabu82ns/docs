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

