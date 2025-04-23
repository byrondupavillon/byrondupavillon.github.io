---
title: "🧠 Nginx Cheatsheet"
date: 2025-04-23
description: "A Nginx cheatsheet covering configuration, directives, and common use cases."
categories:
  - nginx
tags:
  - nginx
  - web server
  - configuration
  - cheatsheet
---
# Nginx Cheatsheet
Nginx is a high-performance web server and reverse proxy server. This cheatsheet provides a quick reference to common Nginx configurations, directives, and use cases.

## 📍 Basics

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        root /var/www/html;
        index index.html index.htm;
    }
}
```

- Basic server block
- Serves static files

## 🔁 URL Rewriting & Proxying

```nginx
location = /mail/unsubscribe {
    rewrite ^(/mail/unsubscribe/[a-f0-9\-]+/[a-f0-9\-]+)/.* $1 break;
    proxy_pass http://upstream-server;
    proxy_connect_timeout 3;
    proxy_next_upstream error timeout invalid_header http_502 http_503 http_504;
}
```

- `location = /mail/unsubscribe` matches the exact URL

```nginx
location ~* ^/mail/unsubscribe/([a-f0-9\-]+)/([a-f0-9\-]+)/ETVi {
    rewrite ^(/mail/unsubscribe/[a-f0-9\-]+/[a-f0-9\-]+)/.* $1 break;
    proxy_pass http://upstream-server;
    proxy_connect_timeout 3;
    proxy_next_upstream error timeout invalid_header http_502 http_503 http_504;
}
```

- `location ~*` matches the URL using a case-insensitive regex. 
    - `~/~*` for regex patterns (case-sensitive/case-insensitive, respectively)
- `rewrite` is used to strip trailing segments from the URL
- `break` stops processing further rewrite rules
- `proxy_pass` sends traffic to another server (e.g., load balancing or routing APIs)


## 🧹 Redirects

```nginx
location /old-path {
    return 301 https://example.com/new-path;
}
```

- Simple permanent redirect from old to new URL