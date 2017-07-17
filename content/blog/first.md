---
title: Caddy, Hugo & Github
slug: building-this-blog
description: Just an experiment.
date: '2017-07-17T11:28:16.222Z'
categories:
- coding
tags:
- hugo
- caddy
- docker
showpagemeta: true
showcomments: true
source: ''
draft: true

---
One would think this blog here is hosted on Github Pages or something of that sort. But you would be quite wrong.

I wanted to try something new, and I love playing around with Cloud stuff and Golang. So what I ended up doing is running this very website inside a docker container consisting out of Caddy webserver and Hugo.

<span style="color: rgb(40, 40, 40); font-size: 1.17em; word-spacing: 0.5px;">Dockerfile:</span>

```
FROM zzrot/alpine-caddy:latest

ENV HUGO_VERSION=0.25.1
ADD https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz /tmp
RUN tar -xf /tmp/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz -C /tmp \
   && mv /tmp/hugo /usr/bin/hugo \
   && rm -rf /tmp/hugo_${HUGO_VERSION}_linux_amd64
RUN apk add --no-cache ca-certificates

COPY Caddyfile /etc/Caddyfile

EXPOSE 3000

```

#### Caddyfile:

```
:3000 {
    root /var/www/html
        minify
        gzip

        header /js /css /images Cache-Control "max-age=2592000"

        header / {
                X-Frame-Options DENY
                Referrer-Policy "same-origin"
                X-XSS-Protection "1;mode=block"
        }

        log / stdout "{&amp;gt;Cf-Connecting-Ip} - [{when}] \"{method} {uri} {proto}\" {status} {size} \"{&amp;gt;Referer}\" \"{&amp;gt;User-Agent}\""
        errors stdout

        git github.com/zet4/zeta.pm /var/www/app {
                hook /webhook secret_here
                then git submodule update --init --recursive
                then hugo -v --destination=/var/www/html
        }
}

```