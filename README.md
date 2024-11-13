# Home-docker

In this repository, I share the complete configuration of Docker Compose and the individual services I use. To simplify, I recommend creating a folder named `my_server` where you can save the individual service configurations.

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    user: 1000:1000
    group_add:
      - "105"
    ports:
      - 8096:8096
      - 8920:8920
    volumes:
      - /home/my-user/my_server/jellyfin/config:/config
      - /home/my-user/my_server/jellyfin/cache:/cache
      - type: bind
        source: /mnt/my-disk
        target: /media
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card0:/dev/dri/card0
    restart: 'unless-stopped'
    extra_hosts:
      - 'host.docker.internal:host-gateway'
    networks:
      my_server_network:
        ipv4_address: 192.168.55.5

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - 10080:80
      - 10443:443
    volumes:
      - /home/my-user/my_server/nginx/conf.d:/etc/nginx/conf.d
      - /home/my-user/my_server/nginx/nginx.conf:/etc/nginx/nginx.conf
      - /home/my-user/my_server/nginx/ssl:/etc/nginx/ssl
    restart: unless-stopped
    networks:
      my_server_network:
        ipv4_address: 192.168.55.10

  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /home/my-user/my_server/hassio/config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    privileged: true
    networks:
      my_server_network:
        ipv4_address: 192.168.55.15

networks:
  my_server_network:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.55.0/24
```
## Networks
For convenience, I used a subnet to assign static IP addresses to individual containers.
```yaml
networks:
  my_server_network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 192.168.55.0/24
          gateway: 192.168.55.1
```
## Services  
Currently, my Docker configuration includes the following containers:  
+ **Jellyfin**: Home media library manager.  
+ **Nginx**: Proxy server for redirecting traffic to other containers.  
+ **Home Assistant**: Service for home automation management.  
+ **Mosquitto**: Parallel service to Home Assistant for integrating MQTT devices.  
+ **Cloudflare DDNS**: Container required to periodically update my public IP address on Cloudflare (not needed if you have a static public IP).  
### Jellyfin
```yaml
jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    user: 1000:1000
    group_add:
      - "105"
    ports:
      - 8096:8096
      - 8920:8920
    volumes:
      - /home/my-user/my_server/jellyfin/config:/config
      - /home/my-user/my_server/jellyfin/cache:/cache
      - type: bind
        source: /mnt/my-disk
        target: /media
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
      - /dev/dri/card0:/dev/dri/card0
    restart: 'unless-stopped'
    extra_hosts:
      - 'host.docker.internal:host-gateway'
    networks:
      my_server_network:
        ipv4_address: 192.168.55.5
```
To set up the Jellyfin container, I followed the following link: [Jellyfin Documentation](https://jellyfin.org/docs/general/installation/container/).
To function properly, we need to add uid:gid.
```bash
~$ id
uid=1000(user-name) gid=1000(group-name) .....
```
Among the volumes, we include the config and cache folders, as well as our external HDDs.
If you want to enable hardware acceleration (if your processor supports it), refer to [Hardware Acceleration](https://jellyfin.org/docs/general/administration/hardware-acceleration/) to set the configuration according to your needs. In my case, I have an Intel GPU and had to add the renderD128 and card0 devices and the "render" group.
```bash
~$ getent group render | cut -d: -f3
105
```
Note: I noticed that after each system reboot, Docker starts the containers too quickly, and Debian doesn't manage to initialize the video card in time for Jellyfin to use it.
For this reason, I created a script to run on every boot that, after 60 seconds, restarts the Jellyfin container.
```bash
~$ nano restart-jellyfin.sh
#!/bin/bash
sleep 60
docker compose -f /home/my-user/my_server/docker-compose.yml down jellyfin
docker compose -f /home/my-user/my_server/docker-compose.yml up -d jellyfin

~$ chmod +x restart-jellyfin.sh
~$ nano /etc/systemd/system/restart-jellyfin.service
[Unit]
Description=Restart Jellyfin Container

[Service]
ExecStart=/path/to/your/restart-jellyfin.sh

[Install]
WantedBy=multi-user.target

~$ sudo systemctl enable restart-jellyfin.service
```
### Nginx (reverse proxy)
The configuration depends largely on how you decide to set it up and, of course, on which services you want to access.
In my case, I mapped nginx.conf and conf.d to include the server configurations, and the ssl folder for the certificates (generated on Cloudflare).
```yaml
nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - 10080:80
      - 10443:443
    volumes:
      - /home/my-user/my_server/nginx/conf.d:/etc/nginx/conf.d
      - /home/my-user/my_server/nginx/nginx.conf:/etc/nginx/nginx.conf
      - /home/my-user/my_server/nginx/ssl:/etc/nginx/ssl
    restart: unless-stopped
    networks:
      my_server_network:
        ipv4_address: 192.168.55.10
```
Below is the nginx.conf and the server configuration that redirects to the Jellyfin container.
```bash
# nginx.conf
events {}
http {
    include /etc/nginx/conf.d/*.conf;
}
```
```bash
# conf.d/jellyfin.conf;
server {
    listen 80;
    listen [::]:80;
    server_name my.server.name;

    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    server_name my.server.name;
    client_max_body_size 20M;

    ssl_protocols TLSv1.3 TLSv1.2;
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    add_header X-Frame-Options https://my.server.name;
    add_header X-XSS-Protection "0";
    add_header X-Content-Type-Options "nosniff";

    add_header Permissions-Policy "accelerometer=(), ambient-light-sensor=(), battery=(), bluetooth=(), camera=(), clipboard-read=(), display-capture=(), document-domain=(), encrypted-media=(), gamepad=(), geolocation=(), gyroscope=(), hid=(), idle-detection=(), interest-cohort=(), keyboard-map=(), local-fonts=(), magnetometer=(), microphone=(), payment=(), publickey-credentials-get=(), serial=(), sync-xhr=(), usb=(), xr-spatial-tracking=()" always;

    location /jellyfin {
        return 302 $scheme://$host/jellyfin/;
    }
    location / {
        proxy_pass http://jellyfin:8096/;
        proxy_pass_request_headers on;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;

        proxy_buffering off;
    }
    location /socket {
        proxy_pass http://jellyfin:8096;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Protocol $scheme;
        proxy_set_header X-Forwarded-Host $http_host;
    }
}
```
### Home Assistant + Mosquitto
The configuration for Home Assistant does not require any particular customizations.
```yaml
homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /home/my-user/my_server/hassio/config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    privileged: true
    networks:
      my_server_network:
        ipv4_address: 192.168.55.15
    ports:
      - 8123:8123
```
For Mosquitto, create the following directories: config, data, log, and a configuration file
```yaml
mosquitto:
    image: eclipse-mosquitto
    container_name: mosquitto
    restart: unless-stopped
    volumes:
      - /home/my-user/my_server/mosquitto:/mosquitto
      - /home/my-user/my_server/mosquitto/data:/mosquitto/data
      - /home/my-user/my_server/mosquitto/log:/mosquitto/log
    ports:
      - 1883:1883
      - 9001:9001
    networks:
      my_server_network:
        ipv4_address: 192.168.55.20
```
```bash
~$ mkdir /home/my-user/my_server/mosquitto/config 
~$ mkdir /home/my-user/my_server/mosquitto/data 
~$ mkdir /home/my-user/my_server/mosquitto/log
~$ nano /home/my-user/my_server/mosquitto/config/mosquitto.conf
# mosquitto.conf
log_dest stdout
log_dest file /mosquitto/log/mosquitto.log
log_type warning
log_timestamp true
connection_messages true
listener 1883
```
After starting the container, create a user and password.
```bash
~$ docker exec -it mosquitto sh
~$ mosquitto_passwd -c /mosquitto/config/mosquitto.passwd my_mqtt_user
~$ exit
~$ nano /home/my-user/my_server/mosquitto/config/mosquitto.conf
# add the following lines
## Authentication ##
password_file /mosquitto/config/mosquitto.passwd
allow_anonymous false
```
### Cloudflare
Service required if you don't have a static public IP. Periodically (in this configuration, every 5 minutes) it updates the DNS records on Cloudflare to propagate any changes to the public IP.
```yaml
cloudflare-ddns:
    image: oznu/cloudflare-ddns
    container_name: cloudflare_ddns
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Rome
      - API_KEY=xxxxxxxxxx
      - ZONE=my.server.name
      - PROXIED=true
      - CRON=*/5 * * * *
    restart: unless-stopped
```
It should be enough to generate an API_KEY with the following permissions:
+ Zone - Settings - Read
+ Zone - Zone - Read
+ Zone - DNS - Edit