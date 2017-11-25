+++
title = "Setting Up A Multipurpose Server"
date = 2017-07-13T19:31:05-04:00
draft = false
+++

Since this is the first post on my blog, I figured it would be appropriate to explain how I set up the server that runs it. The server is just a cheap VPS provided by OVH, and has minimal specifications. For about $5 a month, I get a VPS with 2 GB of RAM, 1 virtual core, and 10 GB of storage. While this isn't a powerful server by any means, I figured that it would be enough to run everything I wanted (which turned out to be the case).

After purchasing and securing the server, I got to work installing software on it. I wanted the server to be capable of the following:

* hosting a personal website, with the option of hosting multiple websites
* running a VPN
* hosting a private git server with web interface

To be able to host and manage these services easily, I chose to use Docker containers. Essentially each service I want to run on the server goes inside its own Docker container, making it easy to control which services are running, and isolating those services from each other. I was able to find existing images/Dockerfiles for the services I wanted to run, so the setup and configuration of the services was a breeze. For my personal website (made using Hugo), I used the [publysher/hugo](https://hub.docker.com/r/publysher/hugo/) image running as a stand-alone server using the default Hugo server. While it's fairly easy to set up a server to host git repositories, I also wanted a lightweight web interface that could provide functionality similar to that of Github. The [gogs/gogs](https://hub.docker.com/r/gogs/gogs/) image provides the gogs self-hosted git application, and seemed to have all of the features that I wanted. The VPN was the most difficult one to set up because it (as expected) required a lot of configuration. The [kylemanna/openvpn](https://hub.docker.com/r/kylemanna/openvpn/) image seemed to be the most popular and supported OpenVPN Docker image, and I followed the instructions on [this blog](http://samwize.com/2016/09/10/setup-your-own-vpn-with-docker-openvpn-and-digital-ocean/) to get it all set up.

I also needed a way to host multiple websites, my personal website and the gogs web interface included. I found that using the [jwilder/nginx-proxy](https://hub.docker.com/r/jwilder/nginx-proxy/) container is a really nice way to accomplish this, especially since all of my services are running inside containers. The nginx-proxy container watches for other containers to start up, and checks to see if that container is run with the "VIRTUAL\_HOST" environment variable set. The proxy then forwards all incoming requests to whatever port is exposed on the newly started container. As an example, I run the Hugo container starting like:

```bash
docker run -e VIRTUAL_HOST=jordan-hatcher.com,www.jordan-hatcher.com  ...
```

and the Gogs container starting like:

```bash
docker run -e VIRTUAL_HOST=git.jordan-hatcher.com  ...
```

Automatically, the nginx-proxy container knows to send requests to jordan-hatcher.com (or www.jordan-hatcher.com) to the Hugo container, and requests to git.jordan-hatcher.com to the gogs container, without requiring any additional user configuration. If I wanted to add a new website (say, new-containerized-site.com), all I need to do is run it in a new container with:

```bash
docker run -e VIRTUAL_HOST=new-containerized-site.com
```

and everything is ready to go!

Here's a little diagram for a visual idea of what's going on:

![](/img/setting-up-a-multipurpose-server/container-diagram.jpg)

Now to check out how the server holds up running all of these services. Nothing has failed catastrophically so far, so the server should be fine, but I ran a few tools to get some actual data. The total idle CPU usage is around 0.3%, and from the VPS web interface, the peak load is still well below 20%. To check memory usage, I used the free command, which showed that almost 1.5 GB out of the total 2 GB was being used! After looking over the output more carefully though, about 1.2 GB of that is being used for cache/buffer, so the actual amount being used is more like 300 MB, which is much more reassuring. In fact, this is shown when I log in to the server, showing 10% of memory being used. As for disk space, storing the Docker images takes up a lot of space, and about 20% of the 10 GB of storage is used already. I'm not super worried about it yet, but it's something to keep in mind if I add more services.

That's about all I've done with the server so far, I'll post updates as I make any changes to it! 
