# Set up encryption for your Homeassistant installation
Your Homeassistant is installed in a docker container. The normal "add-ons" in Homeassistant is not available so you need to manage your certificate outside HA.

To do this you need two extra containes running. Nginx - a small reverse proxy that will intercapt the https-traffix, take care of the certificate, and route all traffic to homeassistant.
Certbot is a utility that can talk to Letsencrypt

# Setting Up Certbot and Nginx

This guide provides instructions to set up Certbot and Nginx for managing SSL certificates and serving content securely.

## Prerequisites
- Docker and Docker Compose installed on your system.
- A domain name pointing to your server's IP address.
- Access to the `/Users/admin/docker/homeassistant` directory.

## Setup Instructions

### 1. Configure Docker Compose
Add the following to the `docker-compose.yml`
```
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: always
    networks:
      - cert-network
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/config:/etc/nginx/conf.d #for configuration files.
      - ./certbot/etc:/etc/letsencrypt #for accessing certificates.
      - ./certbot/lib:/var/lib/letsencrypt
      - ./nginx/webroot:/var/www/html #for serving webroot.

  certbot:
    image: certbot/certbot
    container_name: certbot
    restart: always
    networks:
      - cert-network
    volumes:
      - ./certbot/etc:/etc/letsencrypt # for storing certificates.
      - ./certbot/lib:/var/lib/letsencrypt #for internal Certbot data.
      - ./nginx/webroot:/var/www/html #for domain verification.
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do sleep 6h & wait $!; certbot renew; done'"
```
### 2. Configure homeassistant
Update the `configuration.yaml` configuration file for homeassistant
```
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.22.0.0/16 
```
The network mentioned need to be your internal network where nginx traffic is coming from. You can see in the homeassistant log files

### 3. Configure Nginx
1. Navigate to the `nginx/config` directory.
2. Create or edit configuration files for your domain. Example:

   ```nginx
   server {
       listen 80;
       server_name yourdomain.com;

       location /.well-known/acme-challenge/ {
           root /var/www/html;
       }

       location / {ui
           return 301 https://$host$request_uri;
       }
   }

   server {
       listen 443 ssl;
       server_name yourdomain.com;

       ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3; # Ensure only modern protocols are used
        ssl_prefer_server_ciphers on;
        
       location / {
           proxy_pass http://homeassistant:8123;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;

           # WebSocket support
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade
           proxy_set_header Connection "upgrade";
       }
   }
   ```

### 4. Obtain SSL Certificates
1. Start the Nginx and Certbot containers:
   ```bash
   docker-compose up -d nginx certbot
   ```
2. Run Certbot to obtain certificates:
   ```bash
   docker run --rm \
    -v ./certbot/etc:/etc/letsencrypt \
    -v ./certbot/lib:/var/lib/letsencrypt \
    -v ./nginx/webroot:/var/www/html \
    -v ./certbot/log:/var/log/letsencrypt \
    certbot/certbot certonly --webroot \
    -w /var/www/html --agree-tos \
    -d yourdomain.com
   ```


### 5. Enable Automatic Certificate Renewal
Certbot is configured to renew certificates through the entrypoint in the docker-compose file. It runs automatically every 6 hours. Ensure the Certbot container is running:
```bash
docker-compose up -d certbot
```
In the docker-compose.yml the container your be set to restart:always, to survive any host reboot.

### 6. Restart Nginx
After obtaining or renewing certificates, restart the Nginx container:
```bash
docker-compose restart nginx
```

This setup does not include a direct hook from certificate renew to webserver restart. My system has a nightly restart for other reasons, so after this restart any certitifacte updates will take effect.

## Notes
- Replace `yourdomain.com` with your actual domain name.
- Ensure DNS records for your domain are correctly configured.
- Ensure that port forwarding has been setup in your home router