---
title: "Caddy! Team 443 FTW" 
date: 2018-02-19T11:01:54+05:30
project_url: "https://github.com/mholt/caddy"
categories: ["Ops"]
tags: ["golang", "caddy", "webserver", "https", "tls", "LetsEncrypt", "nginx", "docker"]
---

This one is a gamechanger. It really is.

On a recent Golang obsessed Github browsing spree, I read the following project description: "Fast, cross-platform HTTP/2 web server with automatic HTTPS". I've read that before, and more often than not ended up disappointing myself and resorting to hacks like [this] (https://blog.tux-sudo.com/posts/letsencrypt-nginx-docker/) to create an automatic TLS system for my websites. There are other ways, but almost all of them were semi-automatic in the long run (yay cron!). I decided to try out [Caddy] (https://caddyserver.com/).

After about an hour, I had the "Luke, I'm your father" moment. Now, I'm happily munching secure cookies on the dark side. Check out the line below:

`tls tiredsysadmin@deathstar.com`

That is all you have to configure to get a truly automatic Let's Encrypt HTTPS-enabled website served by caddy. 

What's more? Caddy really feels like a breath of fresh air in terms of configuration. It is written with usability in mind. I thought we already had it pretty good with nginx (but I come from Apache, so make of it what you will), but this is even better.

Check out a sample configuration for an entrypoint that I use:

```
tux-sudo.com {
  tls tanmay.chaudhry@gmail.com

  redir / blog.tux-sudo.com 301
}
```

Yeah, okay it's not much (but isn't that the beauty?). When was the last time you saw an auto-TLS configured in 2 lines? Also, no more returns/redirect to write for HTTP traffic. This does auto-redirects. Stuff like this is why it feels like a true out-of-the-box HTTPS solution. You can still go deep and configure stuff to work the way you want, but more often than not I just want it to work.

But I had a slightly more complex use-case. I wanted to proxy to my prometheus instance. This is running on a raspberry-pi on my internal network behind a dynamic IP. For ISP reasons, I was not able to bind to 443 for the incoming internet. That creates a problem for the let's encrypt challenge. Aw, back to DNS records and manual cert installation? Well, no. Check this out:

```
prometheus.tux-sudo.com {
  log / /var/log/caddy/caddy-prometheus.log
  basicauth / hah nicetry 
  tls {
    dns route53
  }
  proxy / localhost:9090 {
    transparent
  }
}
```

Yes, that is all. Set your DNS provider creds in environment variables and keep munching those cookies. 

The real clincher is that everything here works exactly the same inside containers. 

```
version: '3'

services:
  web:
    image: abiosoft/caddy
    container_name: web
    ports:
      - "80:80"
      - "443:443"
    environment:
      - CADDYPATH=/etc/caddycerts
    volumes:
      - $HOME/.caddy:/etc/caddycerts
      - ./config/caddyfile:/etc/Caddyfile
```

If you're an enthusiast who writes abandonable personal projects every weekend, I urge you to try this out. It is supposed to be production ready as well if you're interested, but that's not something I have tested.

[Caddy](https://caddyserver.com/) : [Caddy-Github](https://github.com/mholt/caddy)
==================================================================================
