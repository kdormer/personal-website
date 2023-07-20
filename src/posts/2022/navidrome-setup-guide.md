---
title: "Navidrome Tutorial"
ctitle: "🎵 Navidrome Tutorial 🎵"
date: 2022-06-25
description: Dislike relying on centralised corporations for something as basic as streaming music? Me too. Here's how to set up Navidrome to stream your own music from your own server.
---

## Introduction

Do you dislike relying on centralised corporations such as Spotify to listen to your music meanwhile not actually *owning* any of it? If so you're in good company, because I do too.
For a long time I was going back and forth between using Spotify and listening to the music I have stored locally. The primary reason behind
this is that I have a _large_ music taste. I was faced with the conundrum of somehow having the same music collection on my phone and on my computer.

In the end I thankfully stumbled upon [Navidrome](https://www.navidrome.org/). Navidrome:

- is [free and open-source](https://github.com/navidrome/navidrome/)
- is Lightweight enough to run on a Raspberry Pi
- is Writen in Go
- is compatible with any Subsonic client
- can transcode music on the fly
- has a built-in web interface
- can handle _huge_ music libraries

Anyway, enough waffle. Here's how to set-up Navidrome so that you can stream your own music from your own server.

## Setup

First, we need to make sure `ffmpeg` is installed:

```bash
apt install ffmpeg
```

After this we will create a new user to run Navidrome under. Note, this is a non-login system user:

```bash
sudo useradd -r -s /bin/false navidrome
```

Next we need to create Navidrome's directory structure:

```bash
sudo install -d -o navidrome -g navidrome /opt/navidrome
sudo install -d -o navidrome -g navidrome /var/lib/navidrome
```

Now we can install Navidrome itself. At the time of writing _v0.47.5_ is the latest version. Make sure to check the
[releases page](https://github.com/navidrome/navidrome/releases) for a newer version and use that download link instead.

```bash
wget https://github.com/navidrome/navidrome/releases/download/v0.47.5/navidrome_0.47.5_Linux_x86_64.tar.gz -O Navidrome.tar.gz
sudo tar -xvzf Navidrome.tar.gz -C /opt/navidrome/
sudo chown -R navidrome:navidrome /opt/navidrome
```

Now we need to point Navidrome at the directory containing our music collection. Create a new file called
`navidrome.toml` in `/var/lib/navidrome` and enter the following:

```bash
MusicFolder = "<library_path>"
```

Replace `<library_path>` with the path to your own music collection and ensure that the `navidrome` user or group
can access this directory. If in doubt use the following command:

```bash
sudo chrgrp -R navidrome <library_path>
```

Once this has been completed, we need to create a new `systemd` unit file. Create a file called `navidrome.service`
in `/etc/systemd/system` and enter the following:

```systemd
[Unit]
Description=Navidrome Music Server and Streamer compatible with Subsonic/Airsonic
After=remote-fs.target network.target
AssertPathExists=/var/lib/navidrome

[Install]
WantedBy=multi-user.target

[Service]
User=navidrome
Group=navidrome
Type=simple
ExecStart=/opt/navidrome/navidrome --configfile "/var/lib/navidrome/navidrome.toml"
WorkingDirectory=/var/lib/navidrome
TimeoutStopSec=20
KillMode=process
Restart=on-failure
DevicePolicy=closed
NoNewPrivileges=yes
PrivateTmp=yes
PrivateUsers=yes
ProtectControlGroups=yes
ProtectKernelModules=yes
ProtectKernelTunables=yes
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
RestrictNamespaces=yes
RestrictRealtime=yes
SystemCallFilter=~@clock @debug @module @mount @obsolete @reboot @setuid @swap
ReadWritePaths=/var/lib/navidrome
PrivateDevices=yes
ProtectSystem=full
```

Once this has been done we can start Navidrome and check that it is running correctly:

```bash
sudo systemctl daemon-reload
sudo systemctl start navidrome.service
sudo systemctl status navidrome.service
```

Assuming everything went smoothly you should be able to access the Navidrome
interface at `http://example.com:4533`, with `example.com` being your own domain name.

## Post-Install Configuration

While Navidrome is now installed properly and can be used, it is **highly** recommended
that you implement the following post-installation steps.

### Set Encryption Key

Due to limitations with the Subsonic API, Navidrome is unable to properly hash passwords and thus
encrypts them. We need to change the default encryption key.

Go to `navidrome.toml` in `/var/lib/navidrome` again and set the `PasswordEncryptionKey` option.
I suggest using a site such as [this](https://passwordsgenerator.net/) to generate a strong
random key.

```bash
MusicFolder = "<library_path>"
PasswordEncryptionKey = "<key>"
```

### Reverse Proxy

Out of the box Navidrome uses its own HTTP server and cannot handle HTTPS. This spells bad news
for transmitting secure information such as passwords over the information. And let's face it,
having to append the port number to the URL is plain ugly.

Instead, we should run Navidrome behind a reverse proxy such as [nginx](https://www.nginx.com/)
and proxy-pass the requests to Navidrome. This way we get both prettier URLs _and_ HTTPS, two birds with one stone.
Put this following config in `/etc/nginx/sites-available` under whatever filename you want.

```nginx
server {
  listen 443 ssl;
  listen [::]:443;
  server_name music.example.com;

  location / {
      proxy_pass http://localhost:4533;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-Forwarded-Protocol $scheme;
      proxy_set_header X-Forwarded-Host $http_host;
      proxy_buffering off;
  }
}
```

In this configuration we are using a music subdomain. Ensure you set the appropriate DNS records.
Now we need to run the following to obtain an HTTPS certificate and automatically
install it into our nginx configuration.

```bash
sudo apt install python3-certbot-nginx
certbot --nginx
```

Follow the instructions presented and select the music subdomain. After `certbot` works its magic,
your Nginx config file should look more like the following:

```nginx
server {
    if ($host = music.example.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    listen [::]:80;
    return 301 https://$server_name$request_uri;
    server_name music.example.com;
}

server {
  listen 443 ssl;
  listen [::]:443;
  server_name music.example.com;
  ssl_certificate /etc/letsencrypt/live/music.example.com/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/music.example.com/privkey.pem; # managed by Certbot

  location / {
      proxy_pass http://localhost:4533;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_set_header X-Forwarded-Protocol $scheme;
      proxy_set_header X-Forwarded-Host $http_host;
      proxy_buffering off;
  }
}
```

After this, we need to set up a crontab to make it so that we don't have to
manually renew our certificates.

Execute the following command:

```bash
sudo crontab -e
```

And enter the following:
```bash
0 0 1 * * certbot --nginx renew
```


Finally we need to configure Navidrome to only accept proxied requests from nginx
and not respond to any request from the outside world.

Once again to to `navidrome.toml` and append the following line:

```bash
Address = "localhost"
```

### External Integrations

To get artist images and other features you should consider integrating Navidrome
with the Spotify & Last.fm APIs. Easy steps to achieve this can be found [here](https://www.navidrome.org/docs/usage/external-integrations/).
