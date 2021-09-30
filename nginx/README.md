# Setup nginx & simple wordpress app, autogen let's encrypt certificate

## Step 1: Buy a domain on GoDaddy, e.g: fago-labs.online
## Step 2. Register Cloudflare account & add domain
## Step 3: Get Cloudflare API key 
## Step 4: Install docker & docker-compose on proxy host

## Step 5: Generate let's encrypt certificate
- Create certbot config for cloudflare plugin. Please change cloudflare email and cloudflare api key below: 

```sh
cat << EOF > /tmp/cloudflare.ini
dns_cloudflare_email = cloudflare_email
dns_cloudflare_api_key = cloudflare_api_key
EOF
```

- Generate certificate

```sh
mkdir -p /etc/letsencrypt /var/lib/letsencrypt
domain=cms.fago-labs.online
docker pull certbot/dns-cloudflare
docker run -it --rm --name certbot \
-v "/etc/letsencrypt:/etc/letsencrypt" \
-v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
-v "/tmp:/tmp" \
certbot/dns-cloudflare certonly --dns-cloudflare --dns-cloudflare-credentials /tmp/cloudflare.ini -d $domain

rm -f /tmp/cloudflare.ini
```

## Step 5: Setup upstream application
## Step 6: Setup nginx & config reverse proxy on proxy host

- Run proxy:

```sh
mkdir -p /opt/nginx/conf.d /opt/nginx/log
cat << EOF > /opt/nginx/docker-compose.yaml
version: '3'

services:
  reverse-proxy:
    container_name: reverse-proxy
    hostname: reverse-proxy
    image: nginx
    ports:
      - 80:80
      - 443:443
    volumes:
      - /opt/nginx/conf.d:/etc/nginx/conf.d
      - /opt/nginx/log:/var/log/nginx
      - /etc/letsencrypt:/etc/letsencrypt
      - /var/lib/letsencrypt:/var/lib/letsencrypt
    network_mode: host
EOF
docker-compose -f /opt/nginx/docker-compose.yaml up -d
```

- Config upstream
  - Change domain & upstream before config:

```sh
domain=cloudrity-demo.fago-labs.online
upstream=http://127.0.0.1:5000
```

  - Add upstream config, for example:

```sh
cat << EOF > /opt/nginx/conf.d/$domain.conf
server {
   listen 80;
   listen [::]:80; 	
   server_name $domain;
   access_log /var/log/nginx/access-$domain.log;
   error_log /var/log/nginx/error-$domain.log;

   return 301 https://\$host\$request_uri;
}

# SSL configuration
server {
   listen 443 ssl http2;
   listen [::]:443 ssl http2; 	
   server_name $domain;
   access_log /var/log/nginx/access-$domain.log;
   error_log /var/log/nginx/error-$domain.log;

   ssl_certificate      /etc/letsencrypt/live/$domain/fullchain.pem;
   ssl_certificate_key  /etc/letsencrypt/live/$domain/privkey.pem;
   
   # Improve HTTPS performance with session resumption
   ssl_session_cache shared:SSL:10m;
   ssl_session_timeout 10m;
   
   # Enable server-side protection against BEAST attacks
   ssl_protocols TLSv1.2;
   ssl_prefer_server_ciphers on;
   ssl_ciphers "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384";
   	
   # RFC-7919 recommended: https://wiki.mozilla.org/Security/Server_Side_TLS#ffdhe4096
   #ssl_dhparam /etc/ssl/ffdhe4096.pem;
   #ssl_ecdh_curve secp521r1:secp384r1;
   
   # Aditional Security Headers
   # ref: https://developer.mozilla.org/en-US/docs/Security/HTTP_Strict_Transport_Security
   add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
   
   # ref: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
   add_header X-Frame-Options DENY always;
   
   # ref: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options
   add_header X-Content-Type-Options nosniff always;
   
   # ref: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection
   add_header X-Xss-Protection "1; mode=block" always;
   
   # Enable OCSP stapling 
   # ref. http://blog.mozilla.org/security/2013/07/29/ocsp-stapling-in-firefox
   ssl_stapling on;
   ssl_stapling_verify on;
   ssl_trusted_certificate /etc/letsencrypt/live/$domain/fullchain.pem;
   resolver 1.1.1.1 1.0.0.1 [2606:4700:4700::1111] [2606:4700:4700::1001] valid=300s; # Cloudflare
   resolver_timeout 5s;

   location / {
     proxy_pass      $upstream;
   }
}
EOF
```

- Reload nginx:

```sh
docker exec reverse-proxy nginx -s reload
```
