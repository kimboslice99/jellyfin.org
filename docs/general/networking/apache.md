---
uid: network-reverse-proxy-apache
title: Apache
---

## Apache HTTP Server Project

"The [Apache HTTP Server Project](https://httpd.apache.org/) is an effort to develop and maintain an open-source HTTP server for modern operating systems including UNIX and Windows. The goal of this project is to provide a secure, efficient and extensible server that provides HTTP services in sync with the current HTTP standards."

## Basic

```conf
<VirtualHost *:80>
    ServerName DOMAIN_NAME

    # Comment to prevent HTTP to HTTPS redirect
    Redirect permanent / https://DOMAIN_NAME/

    ErrorLog /var/log/apache2/DOMAIN_NAME-error.log
    CustomLog /var/log/apache2/DOMAIN_NAME-access.log combined
</VirtualHost>

# If you are not using a SSL certificate, replace the 'redirect'
# line above with all lines below starting with 'Proxy'
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName DOMAIN_NAME
    # This folder exists just for certbot (You may have to create it, chown and chmod it to give apache permission to read it)
    DocumentRoot /var/www/html/jellyfin/public_html

    ProxyPreserveHost On

    # Letsencrypt's certbot will place a file in this folder when updating/verifying certs
    # This line will tell apache to not to use the proxy for this folder.
    ProxyPass "/.well-known/" "!"

    # Tell Jellyfin to forward requests that came from TLS connections
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"
    
    # typical headers
    Header set X-Xss-Protection "0"
    Header set X-Content-Type-Options "nosniff"
    Header set Referrer-Policy "no-referrer"
    Header set X-Robots-Tag "noindex, nofollow, noarchive"

    ProxyPass "/socket" "ws://SERVER_IP_ADDRESS:8096/socket"
    ProxyPassReverse "/socket" "ws://SERVER_IP_ADDRESS:8096/socket"

    ProxyPass "/" "http://SERVER_IP_ADDRESS:8096/"
    ProxyPassReverse "/" "http://SERVER_IP_ADDRESS:8096/"

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/DOMAIN_NAME/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/DOMAIN_NAME/privkey.pem
    Protocols h2 http/1.1

    # Enable only strong encryption ciphers and prefer versions with Forward Secrecy
    SSLCipherSuite HIGH:RC4-SHA:AES128-SHA:!aNULL:!MD5
    SSLHonorCipherOrder on

    # Disable insecure SSL and TLS versions
    SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

    ErrorLog /var/log/apache2/DOMAIN_NAME-error.log
    CustomLog /var/log/apache2/DOMAIN_NAME-access.log combined
</VirtualHost>
</IfModule>
```

## Advanced
You may wish to utilize some security features, this configuration is a good start. It may break some things so be prepared to debug.

```conf
<VirtualHost *:80>
    ServerName DOMAIN_NAME

    # Comment to prevent HTTP to HTTPS redirect
    Redirect permanent / https://DOMAIN_NAME/

    ErrorLog /var/log/apache2/DOMAIN_NAME-error.log
    CustomLog /var/log/apache2/DOMAIN_NAME-access.log combined
</VirtualHost>

# If you are not using a SSL certificate, replace the 'redirect'
# line above with all lines below starting with 'Proxy'
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName DOMAIN_NAME
    # This folder exists just for certbot (You may have to create it, chown and chmod it to give apache permission to read it)
    DocumentRoot /var/www/html/jellyfin/public_html

    ProxyPreserveHost On

    # Letsencrypt's certbot will place a file in this folder when updating/verifying certs
    # This line will tell apache to not to use the proxy for this folder.
    ProxyPass "/.well-known/" "!"

    # Tell Jellyfin to forward requests that came from TLS connections
    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"
    
    # typical headers
    Header set X-Xss-Protection "0"
    Header set X-Content-Type-Options "nosniff"
    Header set Referrer-Policy "no-referrer"
    Header set X-Robots-Tag "noindex, nofollow, noarchive"
    
    # X-Frame-Options conditionally jellyfin/jellyfin-webos/issues/63
    BrowserMatchNoCase !(webOS|playstation|Dalvik) xframe
    Header set X-Frame-Options "SAMEORIGIN" ENV=xframe
    
    # CORS
    Header set Cross-Origin-Resource-Policy "same-origin"
    BrowserMatchNoCase (webOS|JellyfinMediaPlayer|Dalvik) corp
    Header set Cross-Origin-Resource-Policy "cross-origin" ENV=corp
    Header set Cross-Origin-Opener-Policy "same-origin"
    Header set Cross-Origin-Embedder-Policy "require-corp"

    # CSP
    BrowserMatchNoCase !(playstation) securable
    Header set Content-Security-Policy "default-src 'none'; img-src 'self' https://i.ytimg.com https://image.tmdb.org https://m.media-amazon.com data:; script-src 'self' blob: https://www.youtube.com https://www.gstatic.com; media-src 'self' blob:; style-src 'self' 'unsafe-inline'; manifest-src 'self' ; font-src 'self'; connect-src 'self'; frame-src 'self' https://www.youtube.com; frame-ancestors 'none'; base-uri 'self'" ENV=securable

    # Permissions Policy
    Header set Permissions-Policy "geolocation=(), accelerometer=(), battery=(), ambient-light-sensor=(), clipboard-read=(), clipboard-write=(), display-capture=(), xr-spatial-tracking=(), interest-cohort=(), keyboard-map=(), local-fonts=(), magnetometer=(), sync-xhr=(), microphone=(), payment=(), publickey-credentials-get=(), document-domain=(), bluetooth=(), camera=(), gamepad=(self), usb=(self), encrypted-media=(), autoplay=(self \"https://www.youtube.com\"), fullscreen=(self), cast=(self), picture-in-picture=(self \"https://www.youtube.com\")"

    ProxyPass "/socket" "ws://SERVER_IP_ADDRESS:8096/socket"
    ProxyPassReverse "/socket" "ws://SERVER_IP_ADDRESS:8096/socket"

    ProxyPass "/" "http://SERVER_IP_ADDRESS:8096/"
    ProxyPassReverse "/" "http://SERVER_IP_ADDRESS:8096/"

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/DOMAIN_NAME/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/DOMAIN_NAME/privkey.pem
    Protocols h2 http/1.1

    # Enable only strong encryption ciphers and prefer versions with Forward Secrecy
    SSLCipherSuite HIGH:RC4-SHA:AES128-SHA:!aNULL:!MD5
    SSLHonorCipherOrder on

    # Disable insecure SSL and TLS versions
    SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

    ErrorLog /var/log/apache2/DOMAIN_NAME-error.log
    CustomLog /var/log/apache2/DOMAIN_NAME-access.log combined
</VirtualHost>
</IfModule>
```

If you encouter errors, you may have to enable `mod_proxy`, `mod_ssl`, `proxy_wstunnel`, `http2`, `headers` and `remoteip` support manually.

```bash
sudo a2enmod proxy proxy_http ssl proxy_wstunnel remoteip http2 headers
```

## Apache with Subpath (example.org/jellyfin)

When connecting to server from a client application, enter `http(s)://DOMAIN_NAME/jellyfin` in the address field.

Set the [base URL](/docs/general/networking#base-url) field in the Jellyfin server. This can be done by navigating to the Admin Dashboard -> Networking -> Base URL in the web client. Fill in this box with `/jellyfin` and click Save. The server will need to be restarted before this change takes effect.

:::caution

HTTP is insecure. The following configuration is provided for ease of use only. If you are planning on exposing your server over the Internet you should setup HTTPS. [Let's Encrypt](https://letsencrypt.org/getting-started/) can provide free TLS certificates which can be installed easily via [certbot](https://certbot.eff.org/).

:::

The following configuration can be saved in `/etc/httpd/conf/extra/jellyfin.conf` and included in your vhost.

```conf
# Jellyfin hosted on http(s)://DOMAIN_NAME/jellyfin
<Location /jellyfin/socket>
    ProxyPreserveHost On
    ProxyPass "ws://127.0.0.1:8096/jellyfin/socket"
    ProxyPassReverse "ws://127.0.0.1:8096/jellyfin/socket"
</Location>
<Location /jellyfin>
    ProxyPass "http://127.0.0.1:8096/jellyfin"
    ProxyPassReverse "http://127.0.0.1:8096/jellyfin"
</Location>
```
