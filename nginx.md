# NGINX server

## Installing
```bash
$> sudo apt-get update
$> sudo apt-get install nginx
$> nginx -v
```
## Important Files/Directories
* /etc/nginx/ - base directory
* /etc/nginx/nginx.conf - main configuration file
* /etc/nginx/conf.d/* - server configurations to be included in nginx.conf
* /etc/nginx/sites-enabled/default - base sites included in nginx.conf

## Operating
```bash
$> sudo service nginx start
$> service nginx reload
$> service nginx status
```

## Configuration

### SSL/HTTPS config
* Create SSL Certificate (self certification)
```bash
$> cd /etc/nginx
$> sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/cert.key -out /etc/nginx/cert.crt
```

### Reverse Proxy
```
server {
		listen 80;
		server_name hlinode.cloudns.cc;
		root /home/hareesh/sites/bootstrap;

  location /node/ {
			proxy_pass http://localhost:4000/;
		}

  location /jsondb/ {
			proxy_pass http://localhost:3000/;
		}
	}
```
### Sub Domain
```
server {
		listen 80;
		server_name gql.hlinode.cloudns.cc;

		location / {
				proxy_pass http://localhost:4000;
				proxy_set_header Host $host;
				proxy_set_header X-Real-IP $remote_addr;
				proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
				proxy_connect_timeout 150;
				proxy_send_timeout 100;
				proxy_read_timeout 100;
				proxy_buffers 4 32k;
				client_max_body_size 8m;
				client_body_buffer_size 128k;
		}
}
```
### Restricting PORT 80 (HTTP) and Redirecting to 443 (HTTPS)
```
server {
    listen 80;
    return 301 https://$host$request_uri;
}
```
### Server with SSL (HTTPS) and SSH tunnel (ex. KeyBox)
```
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
### Web + API server config [Read More](https://medium.freecodecamp.org/an-introduction-to-nginx-for-developers-62179b6a458f)
```
upstream remoteApplicationServer {
    server 10.10.10.10;
}

upstream remoteAPIServer {
    server 20.20.20.20;
    server 20.20.20.21;
    server 20.20.20.22;
    server 20.20.20.23;
}


server {
    listen 80;
    server_name www.customapp.com customapp.com
    root /var/www/html;
    index index.html

        location / {
            alias /var/www/html/customapp/;
            try_files $uri $uri/ =404;
        }

        location /remoteapp {
            proxy_set_header   Host             $host:$server_port;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_pass http://remoteAPIServer/;
        }

        location /api/v1/ {
            proxy_pass https://remoteAPIServer/api/v1/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
            proxy_redirect http:// https://;
        }
}
```
* [Additional Info](https://www.digitalocean.com/community/tutorials/how-to-configure-nginx-with-ssl-as-a-reverse-proxy-for-jenkins)
* [SSH Keys](http://wiki.eclipse.org/Jetty/Howto/Configure_SSL#Understanding_Certificates_and_Keys)
