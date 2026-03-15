# Reolink Compatible FTPS Server (vsftpd)

[![Docker Pulls](https://img.shields.io/docker/pulls/joeyjojojr/reolink-alpine-vsftpd-server.svg?logo=docker&logoColor=white&style=flat-square)](https://hub.docker.com/r/joeyjojojr/reolink-alpine-vsftpd-server/)
[![Docker Stars](https://img.shields.io/docker/stars/joeyjojojr/reolink-alpine-vsftpd-server.svg?logo=docker&logoColor=white&style=flat-square)](https://hub.docker.com/r/joeyjojojr/reolink-alpine-vsftpd-server/)
[![Docker Image Size](https://img.shields.io/docker/image-size/joeyjojojr/reolink-alpine-vsftpd-server/latest?logo=docker&logoColor=white&style=flat-square)](https://hub.docker.com/r/joeyjojojr/reolink-alpine-vsftpd-server/)
[![GitHub Stars](https://img.shields.io/github/stars/Tom-Jojo/reolink-alpine-vsftpd-server?logo=github&style=flat-square)](https://github.com/Tom-Jojo/reolink-alpine-vsftpd-server)
[![GitHub Forks](https://img.shields.io/github/forks/Tom-Jojo/reolink-alpine-vsftpd-server?logo=github&style=flat-square)](https://github.com/Tom-Jojo/reolink-alpine-vsftpd-server/forks)

Small and flexible docker image with vsftpd server modified with optional data channel encryption (default YES for security, but set NO for legacy Reolink cameras compatibility using PROT C instead of PROT P). Fork of https://github.com/delfer/docker-alpine-ftp-server

The original image forces full encryption on both control channel (login) and data channel (file uploads). Many older Reolink cameras support explicit FTPS (AUTH TLS on port 21) so the username/password is encrypted, but they never send the PROT P command and always use PROT C (or no PROT at all). This means the video upload data channel stays in clear text.

The original image rejects these cameras because it requires PROT P on every data connection.

## This fork adds one simple environment variable

`FORCE_LOCAL_DATA_SSL` (default YES)

- `YES` = original strict behaviour (both login and video upload fully encrypted)
- `NO`  = relaxed data channel (login remains encrypted via TLS, but video uploads are allowed in clear text with PROT C)

This creates a mixed-use scenario on a single server:
- Username and password are always sent encrypted for every camera
- Video uploads can be encrypted (PROT P) for modern cameras or unencrypted (PROT C) for older Reolink cameras
- No need to run two separate FTP servers

## Usage
```
docker run -d \
    -p "21:21" \
    -p 21000-21100:21000-21100 \
    -e USERS="camera1|changeme123" \
    -e ADDRESS=ftp.example.com \
    joeyjojojr/reolink-alpine-vsftpd-server:latest
```
With FTPS + relaxed data channel (for older Reolink cameras):
```
docker run -d \
    --name vsftpd \
    -p "21:21" \
    -p 21000-21100:21000-21100 \
    -e USERS="camera1|changeme123|/home/ftp/camera1|2000|2000 camera2|changeme123|/home/ftp/camera2|2001|2001" \
    -e ADDRESS=ftp.example.com \
    -e TLS_CERT="/etc/letsencrypt/live/ftp.example.com/fullchain.pem" \
    -e TLS_KEY="/etc/letsencrypt/live/ftp.example.com/privkey.pem" \
    -e FORCE_LOCAL_DATA_SSL="NO" \
    -v /path/to/ftp/data:/home/ftp \
    -v /path/to/letsencrypt:/etc/letsencrypt:ro \
    --restart unless-stopped \
    joeyjojojr/reolink-alpine-vsftpd-server:latest
```
## Configuration
Environment variables:
- `USERS` - space and `|` separated list (optional, default: `alpineftp|alpineftp`)
  - format `name1|password1|[folder1][|uid1][|gid1] name2|password2|[folder2][|uid2][|gid2]`
- `ADDRESS` - external address to which clients can connect for passive ports (optional, should resolve to ftp server ip address)
- `MIN_PORT` - minimum port number to be used for passive connections (optional, default `21000`)
- `MAX_PORT` - maximum port number to be used for passive connections (optional, default `21100` in this fork)
- `TLS_CERT` - path to fullchain.pem (enables FTPS)
- `TLS_KEY` - path to privkey.pem (enables FTPS)
- `FORCE_LOCAL_DATA_SSL` - `YES` (default - require PROT P / encrypted data) or `NO` (allow PROT C / clear data for mixed-use scenario with legacy cameras)

## USERS examples

- `user|password foo|bar|/home/foo`
- `user|password|/home/user/dir|10000`
- `user|password|/home/user/dir|10000|10000`
- `user|password||10000`
- `user|password||10000|82` : add to an existing group (www-data)

## FTPS (File Transfer Protocol + SSL) with Let's Encrypt + Cloudflare DNS-01

Use Certbot with DNS-01 challenge (no port 80 needed) for automatic renewal.

docker compose example (recommended full setup):

```
services:
  ftp:
    image: joeyjojojr/reolink-alpine-vsftpd-server:latest
    container_name: vsftpd
    ports:
      - "21:21"
      - "21000-21100:21000-21100"
    environment:
      USERS: >-
        camera1|changeme123|/home/ftp/camera1|2000|2000
        camera2|changeme123|/home/ftp/camera2|2001|2001
        camera3|changeme123|/home/ftp/camera3|2002|2002
      ADDRESS: "ftp.example.com"
      TLS_CERT: "/etc/letsencrypt/live/ftp.example.com/fullchain.pem"
      TLS_KEY: "/etc/letsencrypt/live/ftp.example.com/privkey.pem"
      FORCE_LOCAL_DATA_SSL: "NO"
      MIN_PORT: "21000"
      MAX_PORT: "21100"
    volumes:
      - /path/to/ftp/data:/home/ftp
      - /path/to/certbot:/etc/letsencrypt:ro
    restart: unless-stopped
    depends_on:
      - certbot

  certbot:
    image: certbot/dns-cloudflare:latest
    container_name: certbot
    entrypoint: /bin/sh
    command:
      - -c
      - |
        trap 'exit 0' TERM INT
        echo "Certbot renewal loop started"
        while true; do
          certbot renew \
            --non-interactive \
            --dns-cloudflare \
            --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
            --dns-cloudflare-propagation-seconds 60 \
            --agree-tos \
            --email your.email@example.com \
            --preferred-challenges dns-01 || echo "Renew failed - will retry"
          find /etc/letsencrypt/live /etc/letsencrypt/archive -type d -exec chmod 755 {} \;
          find /etc/letsencrypt/live /etc/letsencrypt/archive -type f -exec chmod 644 {} \;
          echo "Sleeping 12 hours..."
          sleep 12h &
          wait $!
        done
    volumes:
      - /path/to/certbot:/etc/letsencrypt
      - /path/to/certbot/cloudflare.ini:/etc/letsencrypt/cloudflare.ini:ro
    restart: unless-stopped
```
Place your Cloudflare API token in /path/to/certbot/cloudflare.ini (format: dns_cloudflare_api_token = YOUR_TOKEN)

- Do not forget to replace ftp.example.com, your.email@example.com, paths and passwords with your actual values.
- Certificates renew automatically (if needed) every ~12 hours in the loop.

## Reolink Official FAQ References

Reolink provides detailed documentation on FTP/FTPS support and setup:

- Which Cameras/NVRs Support FTP Uploading: <a href="https://support.reolink.com/hc/en-us/articles/900000625446-Which-Cameras-NVRs-Support-FTP-Uploading/" target="_blank">https://support.reolink.com/hc/en-us/articles/900000625446-Which-Cameras-NVRs-Support-FTP-Uploading/</a>
- Introduction to FTPS and SFTP: <a href="https://support.reolink.com/hc/en-us/articles/16877901019417-Introduction-to-FTPS-and-SFTP/" target="_blank">https://support.reolink.com/hc/en-us/articles/16877901019417-Introduction-to-FTPS-and-SFTP/</a>
- How to Set up FTP for Reolink Products: <a href="https://support.reolink.com/hc/en-us/articles/360020081034-How-to-Set-up-FTP-for-Reolink-Products/" target="_blank">https://support.reolink.com/hc/en-us/articles/360020081034-How-to-Set-up-FTP-for-Reolink-Products/</a>
- How to Upload Continuous Recording to FTP: <a href="https://support.reolink.com/hc/en-us/articles/900000624226-How-to-Upload-Continuous-Recording-to-FTP/" target="_blank">https://support.reolink.com/hc/en-us/articles/900000624226-How-to-Upload-Continuous-Recording-to-FTP/</a>
- Can the Recordings Saved to FTP Server be Overwritten: <a href="https://support.reolink.com/hc/en-us/articles/900000669006-Can-the-Recordings-Saved-to-FTP-Server-be-Overwritten/" target="_blank">https://support.reolink.com/hc/en-us/articles/900000669006-Can-the-Recordings-Saved-to-FTP-Server-be-Overwritten/</a>


Pull the image:
docker pull joeyjojojr/reolink-alpine-vsftpd-server:latest

Thanks to the original delfer/docker-alpine-ftp-server project!
