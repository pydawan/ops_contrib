# LetsEncrypt Certs


To generate or renewe the SSL certificats via [letsencrypt](https://letsencrypt.org) the domain must be accessible from the internet. The webserver for the domain must be able to serve static assets. The letsencrypt tool will generate a random file in the root directory of the domain. The generated random file must be accessible from the Internet to verify the owner ship of the domain. For Nginx the configuration of the domain in `/etc/nginx/conf.d/default.conf` can look like this: 

```
server {
  listen 80;
  server_name www.versioneye.com;
  location / {
    root /var/www;
  }
}
```

With that the Nginx server will serve static files from the `/var/www` directory for the domain `www.versioneye.com`. Now the certificates can be generated with this command: 

```
sudo letsencrypt certonly -a webroot --webroot-path=/var/www/html -d www.versioneye.com
```

Or for multiple domains like this:

```
sudo letsencrypt certonly -a webroot --webroot-path=/var/www/html -d www.versioneye.com -d versioneye.com
```

By default letsencrypt is placing the certificates into `/etc/letsencrypt/live`. Now in Nginx the certificates has to be referenced in the domain section for port 443. The `/etc/nginx/conf.d/default.conf` could look like this:

```
proxy_cache_path /tmp/nginx levels=1:2 keys_zone=veye_zone:10m inactive=120m;
proxy_cache_key "$scheme$request_method$host$request_uri";

upstream veye_web {
  server 127.0.0.1:8080;
}

upstream veye_api {
  server 127.0.0.1:9090;
}

server {
  listen 80;
  server_name  versioneye.MY_COMPANY.com;
  return       301 https://versioneye.MY_COMPANY.com$request_uri;
}

server {
    listen 443 ssl;
    server_name versioneye.MY_COMPANY.com;

    ssl_certificate /etc/letsencrypt/live/versioneye.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/versioneye.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS";

    rewrite_log on;

    location /api/ {
        proxy_connect_timeout 1200;
        proxy_send_timeout    1200;
        proxy_read_timeout    1200;
        send_timeout          1200;

        proxy_redirect off;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_pass http://veye_api;
    }

    location /assets/ {
        proxy_redirect off;
        proxy_pass_header Cookie;
        proxy_ignore_headers Set-Cookie;
        proxy_hide_header Set-Cookie;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_cache veye_zone;
        proxy_cache_valid 200 302  120m;
        proxy_cache_valid 404      1m;

        proxy_pass http://veye_web;
    }

    location / {
        proxy_redirect off;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_pass http://veye_web;
    }
}
```

After this changes the Nginx has to be restarted. On Ubuntu that can be achieved with: 

```
service nginx restart
```

Lets Encrypt certificates are valid for 90 days only!
