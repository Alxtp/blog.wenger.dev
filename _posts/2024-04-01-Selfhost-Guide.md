---
title: "Automated Hosting of Static Websites with Docker"
date: 2024-04-01 15:15:00 +0200 
categories: [Self-Hosting]
tags: [docker, caddy, github, nginx-proxy-manager, cloudflare, portainer]
img_path: /selfhost/
---

In this guide I will show you an easy, lightweight and secure way to host your own website at home without showing your IP to the whole world.
The solution consists of the following docker images:

| Docker Image      | Description |
| ----------- | ----------- |
| portainer | Management tool for docker |
| caddy      | Webserver to host the static website |
| nginx proxy manager   | Easy to use proxy with automatic SSL support |
| watchtower | Will automatically update all your container images |
| cloudflare-ddns | Used to automatically update the public dns pointer to your public IP |

Here is an overview of the architecture:

![Light mode only](architecture_light.png){: .light }
![Dark mode only](architecture_dark.png){: .dark }

## Prerequisites
- Any device to run docker on
- Own a Domain or be ready to buy one
- A GitHub Repo with a ready to use static website (at least a `index.html`)

## Docker
1. First install docker on your device by following the [offical manual](https://docs.docker.com/engine/install/).
2. Install [Portainer](https://docs.portainer.io/start/install-ce) to make your life simpler (you dont need to!)
3. Access Portainer: `https://your_ip:9443/`

Now that docker is prepared you can create all the required containers. I usualy use Stacks inside Portainer but you can do it in you own way. The doc refers only to Stacks.

## Caddy
In order to "host" my website files inside a GitHub repo and the webserver pulls changes from there I used the [caddy-git](https://github.com/greenpau/caddy-git) module from [Greenpau](https://github.com/greenpau) and created my own image: https://hub.docker.com/r/greedtp/caddy.

> The creation of the custom image is done here: https://github.com/GreedTP/caddy
{: .prompt-tip }

The image can be used as follows:
```yaml
version: "3.9"

services:
  caddy:
    image: greedtp/caddy:latest
    restart: unless-stopped
    volumes:
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - REPO_NAME=wenger.dev
      - REPO_URL=https://github.com/GreedTP/wenger.dev.git

volumes:
  caddy_data:
  caddy_config:

networks:
  default:
    external: true
    name: dmz
```
> Note that we use a dedicated network "dmz". With this it is not required to publish any ports on the containers (except for the proxy). Instead, we can proxy the traffic internaly.
{: .prompt-info }

## Nginx Proxy Manager
Create the NPM image:
```yaml
version: '3.9'
services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '81:81' # Admin Web Port
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt

networks:
  default:
    external: true
    name: dmz
```
Now you can access the Web interface and create the required proxy host:
1. Login to NPM: `https://your_ip:81/`
2. Define username and password
3. Under proxy hosts, create a new entry with your site url which points the name of your created caddy container to port 80:

![NPM Proxy Host](npm01.png)

> The SSL option will be covered later
{: .prompt-info }

Now your website should already be available under: `http://your_ip`

> When you commited changes to your website repo, you can pull the latest changes to caddy by just opening: `http://your_ip/update`
{: .prompt-tip }


## Cloudflare
To make your website available to the public and hide your public IP, you can use Cloudflare as a reverse proxy.

1. Sign up on [Cloudflare](https://www.cloudflare.com/)
2. Either [register a new domain](https://www.cloudflare.com/products/registrar/) or [add your existing domain to cloudflare](https://developers.cloudflare.com/fundamentals/setup/manage-domains/add-site/)
3. Create a new DNS A record which points to your public IP. Make sure that Proxy status is "Proxied"!

![Cloudflare DNS Management](cloudflare.png)

## Port forward and SSL Certificate
On your router, forward ports 80 and 443 to your device IP. You can then request a new certificate for your domain from the SSL tab of your proxy host:

![SSL Certificate](npm02.png)

> If this doesnt work, use the DNS challange option to create a new SSL certificate
{: .prompt-tip }

Congrat! Your site is now public available. But maybe not for long as your public IP might change at some time. To solve this problem I used cloudflare-ddns.

## Cloudflare-DDNS
This container will automatically adjust the A record to point to your public IP if it changes.
Create the config file by following the instructions here: https://github.com/timothymiller/cloudflare-ddns

When the `config.json` file is prepared you can create the docker image:
```yaml
version: '3.9'
services:
  cloudflare-ddns:
    image: timothyjmiller/cloudflare-ddns:latest
    security_opt:
      - no-new-privileges:true
    network_mode: 'host'
    volumes:
      - /data/cloudflare-ddns/config.json:/config.json
    restart: unless-stopped
```

> Make sure the path for the volume matches the location of your `config.json` file.
{: .prompt-tip }

## Watchtower
Finally create the Watchtower image which will update all the other images automatically in your docker:
```yaml
version: "3.9"
services:
  watchtower:
    image: containrrr/watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

# Conclusion
I have shown you a modern way to easily and securely (I would say) host your own website from home.

However, this can only be the foundation of your self-hosted infrastructure. Now you can host as many services as you want; all you need to do is deploy a container, create a new entry in NPM, and create a new CNAME entry in Cloudflare.
