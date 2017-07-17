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

---
One would think this blog here is hosted on Github Pages or something of that sort. But you would be quite wrong.

I wanted to try something new, and I love playing around with Cloud stuff and Golang. So what I ended up doing is running this very website inside a docker container consisting out of Caddy webserver and Hugo.

## Dockerfile

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

Now let's go through this step by step.
I base the image off `FROM zzrot/alpine-caddy:latest` which is probably the best alpine /w caddy image out there right now.

```
ENV HUGO_VERSION=0.25.1
ADD https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz /tmp
RUN tar -xf /tmp/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz -C /tmp \
   && mv /tmp/hugo /usr/bin/hugo \
   && rm -rf /tmp/hugo_${HUGO_VERSION}_linux_amd64

```

Originally I used alpine's package repository for Hugo, but as I found out recently the version there is terribly out of date.
As such I now download the release directly from github, currently pineed to latest, `0.25.1` version.

I also added the CA bundle (`RUN apk add --no-cache ca-certificates`) as down the road I might need to make https requests.

Then I replace (`COPY Caddyfile /etc/Caddyfile`) the default zzrot's config with mine (more about it below).

Finally I expose port 3000 (`EXPOSE 3000`) for traefik to auto detect it.

## Caddyfile

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

        log / stdout "{>Cf-Connecting-Ip} - [{when}] \"{method} {uri} {proto}\" {status} {size} \"{>Referer}\" \"{>User-Agent}\""
        errors stdout

        git github.com/zet4/zeta.pm /var/www/app {
                hook /webhook secret_here
                then git submodule update --init --recursive
                then hugo -v --destination=/var/www/html
        }
}

```

So this looks a bit more fun right? Let's go through this one too.
I have a Caddy http block listen on `:3000` which I exposed in the Dockerfile previously, the block's root for files is `/var/www/html`.
I have also enabled `minify` and `gzip` extensions for the website, so far so good.

I also have setup a response header that caches assets (`header /js /css /images Cache-Control "max-age=2592000"`).
There are also 3 other headers added to the response for security reasons.

Next up we have request logs: `log / stdout "{>Cf-Connecting-Ip} - [{when}] \"{method} {uri} {proto}\" {status} {size} \"{>Referer}\" \"{>User-Agent}\""`
The server I run this on is behind cloudflare, cloudflare is a sort of proxy, because of that I never see the source address in the right place, instead its in the Cf-Connecting-Ip header, thus I had to update output format to use that http header instead.

Finally there is the git extension block (`git github.com/zet4/zeta.pm /var/www/app`), when the server is first started it will pull the git repository of the website and put it into `/var/www/app`.
It will also start listening to a webhook (`hook /webhook secret_here`).

Both on start and properly authenticated webhook request the server will first call `git submodule update --init --recursive` followed by `then hugo -v --destination=/var/www/html`, which will first pull submodules (the theme of the blog is a submodule) and then build the website to the directory.

## docker-compose.yaml

```yaml
version: '2'

services:
    traefik:
        image: traefik
        ports:
            - "80:80"
            - "443:443"
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - /home/zeta/new-setup/traefik.toml:/traefik.toml
            - /home/zeta/new-setup/certs/:/etc/traefik/certs
            - /home/zeta/new-setup/traefik-logs/:/etc/traefik/log
        restart: always
        labels:
            - "traefik.backend=traefik"
            - "traefik.port=8080"
            - "traefik.frontend.rule=Host:traefik.zeta.pm"
    blog:
        build: ./blog-web
        restart: always
        volumes:
            - /home/zeta/new-setup/blog-web/cache:/var/www/html
        labels:
            - "traefik.backend=blog"
            - "traefik.frontend.rule=Host:zeta.pm"

```

You might notice so far we only have a container thats listening on port 3000, how does a normal user get access to it? 
Well that's where the compose file above comes in, it creates 2 services (this is a simplified version of the one I have running currently).
One for blog, and one for loadbalancer written in Go called Traefik, think of it as Nginx but with smart configs.
In my case I have it configured to pull from docker labels, as you noticed before, blog's Dockerfile exposes port 3000, Traefik by default will take the first exposed port (but it can be changed with a seperate label), and route requests following frontend rules.
In my case its configured to serve blog service on Host:zeta.pm

And that's it, now with a simple `docker-compose up -d` I can start the entire thing, it will build the image, download traefik, and run in the background.