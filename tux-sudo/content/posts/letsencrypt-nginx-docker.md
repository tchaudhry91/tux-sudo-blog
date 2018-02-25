---
title: "LetsEncrypt Certificates for Dockerized Nginx"
date: 2017-09-26T16:25:49+05:30
tags: ["docker", "nginx", "webserver", "certbot", "letsencrypt"]
categories: ["Ops"]
project_url: "https://github.com/tchaudhry91/nginx-certbot-docker"
---

I like nginx. I like docker. I like SSL. I also like not paying for SSL certificates.<br>
If you, like me are operating a small time website, you probably don't want to dish out a few hundred dollars for an SSL certificate.Thankfully, [LetsEncrypt](https://letsencrypt.org) provides **free** ssl certificates and is now trusted by most major browsers.<br>
However, integrating [certbot](https://certbot.eff.org/) to automatically configure my nginx inside a docker container has been a pain. In the past, I generated the standalone cert with --cert-only and then linked my docker container to use it but this was a far from usable solution given the horrid automation code I had to write. <br>
Recently, I tried to come up with a more automation friendly solution and ended up with the following:<br>

  + Custom **nginx image** with certbot pre-installed inside the container. <br>

    **Dockerfile**:
    ```
    FROM nginx

    RUN apt-get update && apt-get install -y \
      certbot \
      python-certbot-nginx

    ADD certify.sh /

    RUN chmod +x certify.sh

    ``` 

    **certify.sh**:
    ```
    #!/bin/bash

    certbot --nginx -d $DOMAIN -m $EMAIL --agree-tos -n --redirect 
    ```

    Github-Source: https://github.com/tchaudhry91/nginx-certbot-docker<br>
    DockerHub Image: https://hub.docker.com/r/tchaudhry/nginx-certbot-docker/

  + Then I would simply run the container with the following docker-compose block:
    ```
    version: '3'

    services:
      web:
        image: tchaudhry/nginx-certbot-docker
        container_name: web
        ports:
          - "80:80"
          - "443:443"
        environment:
          - DOMAIN=example.com
          - EMAIL=xyz@example.com
        volumes:
          - letsencrypt:/etc/letsencrypt
          - ./config/nginx_config.conf:/etc/nginx/conf.d/default.conf

    volumes:
      letsencrypt:
    ```
    
    An equivalent `docker run` command would also suffice. But do make note of the _enviroment variables_ and the _letsencrypt volume_. 

      + The _enviroment variables_ are used to provide information about the domain for which the certificate is to be requested.
      + The _letsencrypt_ volume is also essential as this will prevent recertification till the old certificate expires. This can also be a host-mounted directory, however I prefer docker volumes.

  + This image does not do ssl yet. Infact, it can be used as a drop in replacement for the regular nginx image. I can 'certify' the container inside it on demand with:<br>
    `docker exec container_name /certify.sh`
    This command can easily be cron-ed on the base system to re-certify and prevent expired certificates.

**Pitfalls**

+ While this works for now, a true service is most likely going to be load balanced. This approach would require scaling down a single container to ensure the letsencrypt verification passes before certification. The service can then be scaled back up, and given the shared volume, it will work without requiring re-certification.
