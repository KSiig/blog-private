---
layout: post
title:  "Docker & LetsEncrypt DNS Validation"
date:   2019-03-23 12:00:00 +0200
categories: docker security
image: https://i.imgur.com/N7ArAH5.png
ascent_image:
  background: url('https://i.imgur.com/N7ArAH5.png') center/cover
  overlay: false
invert_sidebar: false
---

When I first wanted to get into servers, one of the first things I knew was that I needed an SSL certificate. After some research, I found out that EFF had created LetsEncrypt, where you can get a free cert! So I followed some guides on how to set it up, but it caused me a lot of trouble, specifically because I wanted to have my servers behind a reverse proxy.

It was a day of glory, once I figured out that you could do something called DNS validation, and just how much it eases the whole process. And it only got better when i found the container made by [linuxserver.io](https://hub.docker.com/r/linuxserver/letsencrypt). In this article you will get a brief introduction into how HTTP and DNS validation works, and just how to set it up.

# Why Use DNS Validation?

## How HTTP Validation Works

If you’ve ever tried getting a LetsEncrypt certificate via HTTP validation, and your servers are hidden behind a reverse proxy, you know how annoying that can be. If you haven’t tried getting a cert via http validation before, here’s what happens:

You use something like Certbot to get the certificate. What it will do is send a request to the LetsEncrypt servers with various information, including the domain you are requesting a cert for, let’s say *example.com.* Certbot will also place a file on your system.

The LetsEncrypt servers will then send a request to *example.com*, looking for the file that Certbot has placed. If it finds the file: great! That must mean you own the domain that you are requesting a cert for, and will be granted the certificate.

When your servers are placed behind a reverse proxy, you will have to add some configuration to it, as to make sure the requests from LetsEncrypt, hit the correct place. Additionally you will also be exposing more to the world wide web than needed. This is where DNS validation shines.

## How DNS Validation Works

When you set up Certbot with DNS validation, the LetsEncrypt server will only check your DNS, it won’t send a request to the server being hosted on that domain. What this means, is that when you are doing this type of validation, you will be asked to enter some records in your DNS.

Continue the process and LetsEncrypt will check that the previously mentioned records exist. If they exist: great! That must mean you own the domain. I should mention the obvious, that these records are unique to every domain, hence why it ensures ownership.

As mentioned; LetsEncrypt never sends a request to the server, only the DNS. This means that you don’t need to setup some entry in your proxy or anything, and in reality you don’t even have to do this process on the server that needs the cert. You can do it on basically any computer. This wouldn’t make a lot of sense, it’s just a testament to the flexibility of DNS Validation.

Luckily there are multiple containers which automates this entire process, however I’m a firm believer in understanding what’s happening, even though you’re not directly doing it. Now without further ado, let’s get started!

# Prerequisites

## Domain

You should have a domain (here we will be using *example.com*), and have it set up with a DNS provider supported by [Certbot](https://certbot.eff.org/). Here is a list of valid providers:
- cloudflare
- cloudxns
- digitalocean
- dnsimple
- dnsmadeeasy
- google
- luadns
- nsone
- ovh
- rfc2136
- route53

## Docker & Docker-Compose

You should have Docker version 17.12.0+, and Compose version 1.21.0+.

# Setting Up Compose File

## What to Do

```yaml
version: '3'
services:
  letsencrypt:
    image: linuxserver/letsencrypt
    container_name: letsencrypt
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - URL=example.com
      - SUBDOMAINS=wildcard,
      - VALIDATION=dns
      - DNSPLUGIN=<dns-provider>
      - EMAIL=<e-mail>
      - DHLEVEL=4096
    volumes:
      - </path/to/appdata/config>:/config
    restart: unless-stopped
```

This is like any other compose file; what you should focus on here, are the environment variables, and the volume mount. It is important than the config folder does not exist on your host yet. This will be explained later.

The `PUID` and `PGID` should be set to whoever owns the folder where you are mounting the volume. Change `URL` to your domain, and the `DNSPLUGIN` to your DNS provider (i.e. cloudflare). It should written, as it has in the list above. Now the only thing remaining is to change `EMAIL`, and you’re set.

You may also notice that `SUBDOMAINS` is set to `wildcard`. This means that the certificate will work on all your subdomains. In most cases this is preferred, but if you want to name specific subdomains, type them in a comma separated, no spaces list.

## Why it Works

If you haven’t yet worked with containers made by linuxserver.io, you are really missing out, and should look into their work. Most of their containers mount to `/config`, which is where they, funnily enough, store all of their configuration. This is also where your certificates are going to end up.

Another great thing about their containers, is the ability to set what user the container should act as. Mostly your `id` will be 1000, however you can check it by running the id command in your terminal. What this means in practical terms, is that the folders and files the container generates, have the owner and group, defined by those two environment variables.

# Running Container First Time

## What to Do

Now it is time to run the container for the first time. This will do some initialisation, which is crucial for the next part to work. Make sure you are in the directory containing the `docker-compose.yaml` file, and then run `docker-compose up -d`.

Bonus tip: If you are going to be using the compose file a lot, I recommend setting an alias to it. Personally i use `alias dcp='docker-compose -f /path/to/docker-compose.yaml'`.

## Why it Works

Remember the mention earlier, about how it is important that the config folder should not yet exist on your host? Well here’s the reason why: When you volume mount to a folder that doesn’t yet exist, docker will automatically create that folder, and populate it with the contents of the container; in this case `/config`. Sounds simple enough, and it is. There’s just a simple pitfall, that I’ve fallen in multiple times myself:

The only thing you’re telling Docker, is that it should mount a specific folder. You’re not telling it to check the contents, or anything of the sorts. This means, that if you mount `/path/to/your/config:/config`, and an empty `/config` folder exists on your host, the folder inside your container will be empty too, creating a whole world of problems. And yes once you know it, it is incredibly obvious, but it may cause confusion for some. Myself included.

# Almost at the End

## What to Do

You should now go into `/path/to/your/config/dns-conf`, where you will find a bunch of different `.ini` files. Find the one that matches your DNS provider, and open it with your favorite text editor. Fill in the required values, and save the file. From here a simple `dcp restart letsencrypt` should be enough. If everything works correctly, you can find your certificate in different file formats in `/path/to/your/config/etc/letsencrypt/live/example.com/`.

And there you have it. A free certificate for your domain, which will renew itself. One thing to note, is that you will receive a mail after 60 days, saying your cert is close to expiring. You do not have to worry about that. It’s simply because of when the container decides to renew your certificate.

## Troubleshooting

I have experienced some times when a simple restart of the container isn’t enough. In that case you will have to run `docker stop letsencrypt && docker rm letsencrypt`. DO NOT remove the config folder, only remove the container. Now just run `dcp up -d` and it should work for you.
