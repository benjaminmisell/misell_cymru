---
date: "2018-05-23T09:21:54+01:00"
title: "MQTT on the open internet"
authors: []
tags:
  -
draft: true
---

# The problem

For a project for my company [Fluid Media](https://fluidmedia.wales) I had to signal to a server in another location, behind NAT to do something. Port-forwarding wasn't an option (client felt odd about it) so a dial in option wasn't going to work. Now I could have written my own solution, but thats almost never a good idea. So I went out to find something that already did what I want.

So as I was feeling lazy on the day, I asked twitter. The super amazing [@KevinHoffman](https://twitter.com/KevinHoffman), came to the rescue with the suggestion of [NATS](https://nats.io).

<blockquote class="twitter-tweet" data-cards="hidden" data-partner="tweetdeck"><p lang="en" dir="ltr">NATS (<a href="https://t.co/8xI9WHR3Ph">https://t.co/8xI9WHR3Ph</a>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Unfortunately NATS didn't really have the authentication/authorization that I needed. Users and permissions could be defined but only in a static config file that required a server reboot to apply. Not ideal. After some more talking over DM the idea of MQTT came up. I'm surprised that I didn't think of it myself as I use it for all my home automation.

# Which server to use

For all my home automation I use Mosquitto, and I wanted to use it here as I understand how it works and it seems to be a well written implementation. MQTT does have authentication built into the protocol, but not authorization. Mosquitto simply gets around this by disconnecting the client if it does something it can't. 

​On it's own Mosquitto can't handle auth, it requires a plugin to be written, in C. I didn't fancy that. Luckily such a plugin already exists, in the form of [mosquitto-auth-plugin](https://github.com/jpmens/mosquitto-auth-plug) made by [jpmens](https://github.com/jpmens) on Github. It supports all kinds of stuff from MySQL to Mongo to LDAP, but the only one of intrest to me is HTTP/JWT. This allows me to have the most flexibility as I can write a micro-service in my favorite language (GoLang) to handle auth.

# How to run mosquitto?

Next I actually have to run the server. As my company user kubernetes it has to be in a docker container. So as this requires a build but I don't want a massive container with all the build tools, a multi stage container is perfect for this. This allows you to put many `FROM` in the Dockerfile and copy from previous stages by using:

```dockerfile	
COPY --from=0 foo bar
```

