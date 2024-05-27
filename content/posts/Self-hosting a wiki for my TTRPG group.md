---
title: 'Self Hosting a Wiki for My TTRPG Group'
date: 2024-05-27T17:58:22-04:00
draft: false
tags:
  - wikijs
  - docker
  - cloudflare
---

# Self-hosting a wiki for my TTRPG group
I've been a part of an ongoing series of TTRPG (Tabletop Roleplaying Games) for a few years now. I'm so fortunate to have a group of friends that are committed enough to keep a game going for this long, and we even play in our office which is pretty cool. 

Of course there's a lot that goes into making these games happen. One thing we do to make this easier is sharing the responsibilities for running the game between all of us. Those kinds of things include ordering food, playing music, running the game board display, and taking notes. That last responsibility is mine.

For the longest time, I was taking these notes in [Obsidian](https://obsidian.md/) and backing them up to a public [Github repository](https://github.com/Luke-Rand/Blackstar-Campaign-Notes). I foolishly thought that instructing people to visit my Github profile would be a sufficient way to share these notes. As it turned out, that was a terrible experience.

So because of this I set out to find the best way to host my notes, and maybe even enable my fellow party members to make contributions as well.

## My requirements
Need to have:
- Publishing to the public internet
- Ability for other (authorized!) people to make contributions
- Ability to drop my existing notes in place from Obsidian
- Self-hosted
- Secure

Nice to have:
- Markdown editor
- Good design
- Docker container
- Open-source with good community

## Options for hosting notes
### Github
I've tested this option, and it failed. You really need to be a technical person to understand how to effectively view markdown files on Github. And even then, you really should also have a good working knowledge of how Github works as well. Markdown files straight from Obsidian aren't always formatted correctly too. This option is out.

>![](https://i.giphy.com/WRQBXSCnEFJIuxktnw.webp)
>*Trying to figure out what happened last session but I have to use Github*

### Hugo
![Hugo](https://raw.githubusercontent.com/gohugoio/gohugoioTheme/master/static/images/hugo-logo-wide.svg?sanitize=true)
I really wanted this option to work. As you may know, this blog is made using Hugo (find out more about that [here](Why I moved my blog from Wordpress to Hugo.md)). The obvious downside to using Hugo is that I would be the only one able to publish the campaign notes. And even worse still, the process for making these notes available would go like this:
1. Take notes in Obsidian
2. Copy notes to Hugo campaign notes directory
3. Commit changes for the hugo site
4. Github Actions runs automatically, publishing the site after about ~2-3 minutes
5. Notes are available on the Github Pages site

That's just not going to work.

### Wiki.js
![Wiki.js](https://camo.githubusercontent.com/522006c76f01f78adaa031157f6a1fbc5264b2a9dfc0b454c562006cf6047e90/68747470733a2f2f7374617469632e7265717561726b732e696f2f6c6f676f2f77696b696a732d66756c6c2d6461726b7468656d652e737667)
I knew from the jump that a wiki of some sort would be my best bet. With a wiki service, you can reasonably expect it to support multiple maintainers, markdown support, and a interconnected set up pages with backlinks. But the best part is that my users (party members) don't need to worry about any of that! At a maximum, all they need to do is visit the site and read any page to get value from it.

I've used products like [TiddlyWiki](https://tiddlywiki.com/) in the past. That worked well for the simple application I was using it for at the time. Since then though, many wiki services have become available. In my search, I eventually found [Wiki.js by Requarks](https://github.com/requarks/wiki). This product checked all of the need-to-haves as well as nice-to-haves that I had on my list. I could self-host Wiki.js in my homelab as a Docker container. It even supports Markdown and has a very robust set of features for managing users.

Let's get started!

## Hosting Wiki.js
### Docker
I started by looking on [Dockerhub](https://hub.docker.com) for a community image to use for my deployment. I was happy to see that there was an [actively maintained image](https://hub.docker.com/r/linuxserver/wikijs) from linuxserver.io. A lot of other services in my homelab are run on images provided by linuxserver.io so I trusted that their Wiki.js image would be just as reliable.

I started by putting together a `dockercompose.yml` file for the `lscr.io/linuxserver/wikijs:latest` image. I kept most of their sample docker compose file the same, except for the following changes:
- Changing `TZ` to my local timezone 
- Mapping my local persistent storage
- Adding my custom `cloudflared` docker network for communicating with my Cloudflare Tunnel container (more on that below)

![Portainer](/images/wikijs/portainer.png)

For now, I am deploying Wiki.js using a sqlite database. It's important to note that when Wiki.js 3.0 eventually comes out, it will require PostgreSQL. Because of this, my current deployment is not future-proof, but it does at least deliver a minimum viable product.

I then opened Portainer on my dedicated Docker host for external-facing services. This host is on a network segment that can't make outbound connections to other hosts on my local network. From there, I used the Stacks feature of Portainer to deploy my docker compose file. I'm using Stacks because it's handy for deploying an entire suite of microservices from a single compose file. Eventually, I plan to deploy a dedicated PostgreSQL database for this wiki, as well as a dedicated Elasticsearch services. Keep an eye out for those blog posts in the future!

After copy/pasting my docker compose file into the Portainer Stacks editor and clicking deploy, the site went up almost instantly. **Success!** I did a few little edits to make sure this was going to work for me. When I was comfortable with going forward, I moved on to the next steps.

### Cloudflare
Being that this is an externally facing web service, I wanted to minimize my attack surface as much as possible. No RPG wiki is worth the security of my home network. That meant that I definitely wasn't going to be doing any port forwarding for this project. I opted instead to use Cloudflare's Zero Trust networking service: [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/).

Cloudflare Tunnel allows me to host services on my local network without exposing that service directly to the internet. Everything going to my web server is proxied through Cloudflare first. For a while now, I've had a Cloudflare Tunnel node in the same network segment as my hosts for external services. You can see how this is done in the diagram below.
![Cloudflare Tunnel diagram](https://cf-assets.www.cloudflare.com/slt3lc6tev37/5uLXEYIlWL2EyMuugX2H4U/916ebdad56ed34cdb3161bd471e95ec5/argo-tunnel-diagram.svg)

I started by going to Cloudflare's Zero Trust portal, where my existing Cloudflared node is set up. I then used the `Create a tunnel` button to input the specifics of my internal web server. Here, I input the hostname of my Docker server host, as well as the port that Wiki.js is being hosted on. Then, I assigned the route that should be used to access this service. For me, I will be using rpgwiki.lukerand.com so I put that in. This automatically creates a DNS entry for my lukerand.com domain. Almost instantly, you could visit the Wiki.js instance by going to [rpgwiki.lukerand.com](https://rpgwiki.lukerand.com). **Double success!**

![My Wiki.js home page](/images/wikijs/homepage.png)

## What's next
### Backups
Right now, I'm using Wiki.js's local filesystem backup feature. Eventually, I think I'll want to look into the S3 storage option. That's a technology that I want to learn more about so I think this application would be a great excuse to do that.
![Storage Settings](/images/wikijs/storageSettings.png)

### Clustering
Right now, Wiki.js is being hosted on a single Docker server. This is clearly more than enough for the kind of traffic I'm serving, but this is another example of a technology I want to learn. So why not give that a shot at some point?

### Elasticsearch
Wiki.js supports a number of "Search Engine" products in the admin settings. One of these is Elasticsearch. We are currently exploring the application of Elasticsearch at my work, so I see this as a great way to get familiar with that technology as self-study.
![Search settings](/images/wikijs/searchSettings.png)

### PostgreSQL
On Requark's blog for Wiki.js, they have [an article](https://beta.js.wiki/blog/2021-wiki-js-3-going-full-postgresql) about requiring PostgreSQL in version 3 of Wiki.js. That means that my current use of a sqlite database eventually won't be supported.