---
layout: post
title: How I created my first blog site on Wordpress?
date: 2022-06-29 00:15 -0400
categories: homelab self-hosting
tags: homelab blog proxmox docker monitoring prometheus
img_path: ../../images
---

**UPDATE***: I now use Jekyll for my blog instead of WordPress. This new blog is deployed using Netlify rather than being self-hosted in my homelab. I'm leaving this article as is since it still has alot of useful information and because it was the firt blog post I ever wrote.*

I thought it was only appropriate that my first blog post be about how I created this WordPress site. While I hope to make future blog posts short and sweet, this one is probably going to be a bit wordy as I talk about my my thought process and some of the issues I experienced along the way. I hope to look back on this some day and laugh.

### Hosting

In the past I’ve hosted everything using public cloud providers like AWS or more recently Linode. And I have nothing against those except for the fact that they cost $$. Fortunately, I recently converted a spare PC into a “homelab” server running Proxmox so hosting it myself only made sense. For specifics, here is a brief breakdown of my hardware:

* **CPU**: core i7 9700K
* **RAM**: 32GB DDR4 3200MHz
* **STORAGE**:
  * 256GB M.2 PCIe NVMe SSD (running Proxmox)
  * 1TB 7200RPM SATA (Directory Storage)
  * 2 X 1TB SATA SSD running ZFS mirror for redundancy

### Operating System

This gave me a bit of a headache because I initially used a normal VM running Ubuntu 20.04 but noticed it seem to be using a ton of memory 90%+ at all times. I tried throwing more memory at it but the more I gave it the more it used. Thankfully, I was running the app using docker (more on that next) so I was easily able to move to another setup.

I had read good things about LXC’s and using turnkey-core to run docker and the low resource consumption you got from such a setup. I decided this was what I wanted to do so I set one up and besides my WordPress site loading a bit slower, everything else seemed ok initially including memory usage which was sub 40%. Then I started to run into problems using docker. The docker daemon would hang after issuing commands like `docker-compose down` and these hangs would eventually lead to timeouts. I tried increasing the length of timeouts using `COMPOSE_HTTP_TIMEOUT` but that didn’t seem to do anything. The only thing that did work consistently was restarting the docker deamon with `sudo systemctl restart docker` prior to issuing any command. But this was a pain in the butt to do because the deamon would take a minute or two to complete its restart. I also tried giving the container more memory and that seem to make things better for a bit but never solved my issue entirely.

So what does one do in this situation, he scours the internet for advice. At some point during my travels I stumbled upon this article [Linux ate my ram](https://www.linuxatemyram.com/) which really put my initial memory concern into perspective. I didn’t mention this before, but one thing that confused me early on was the 90% memory consumption being reported by the Proxmox UI verse the usage being reported by `htop` and the New Relic agent I had installed. However, it all made sense to me after reading that article.

It was at this time that I decided to go back to using a VM running Ubuntu 20.04 and so I did. The Proxmox UI still shows ~90% memory usage, however running `htop` or `free -h` shows my memory available is actually closer to 50%:

#### Proxmox UI vs Actual

![proxmox](proxmox-ui-storage.png){: width="800" height="800" }{: .left }

### Deployment

As mentioned earlier, my entire stack including wordpress, database, webserver, and monitoring (more on that next) is deployed using one docker-compose file. I’ve since read that I could break this up into separate files but I like having only one file to edit when working from the terminal. I simply use the `depends_on` parameter to control the order in which things get built and run. If you’re interested in seeing how I have everything configured then you can refer to my git repo [here](https://github.com/timmyb824/wordpress-blog). I took a lot of inspiration from, and want to give credit to, this Digital Ocean article [How to install wordpress with docker-compose](https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-docker-compose). I obviously had to make changes to fit my needs but this got me up and running with very little time and effort.

To provide a but more insight into my stack, here is a breakdown of each service I’m running:

* **[db](https://www.mysql.com/)** – mysql database which is required by wordpress. You’ll see I use my own configuration file to improve the performance of mysql.
* **[wordpress](https://wordpress.com/)** – responsible for the frontend you see here.
* **[webserver](https://www.nginx.com/)** – NGINX webserver to handle connections and serve the website over ports 80/443.
* [**certbot**](https://certbot.eff.org/) – provision SSL certificates
* **[prometheus](https://prometheus.io/docs/prometheus/latest/installation/)** – metrics server for monitoring
* **[node_exporter](https://github.com/prometheus/node_exporter)** – works with Prometheus to provide host metrics
* **[cadvisor](https://github.com/google/cadvisor)** – works with Prometheus to provide container metrics
* **[mysqld_exporter](https://github.com/prometheus/mysqld_exporter)** – works with Prometheus to provide mysql db metrics

All services are deployed to the same network, which is one of the benefits of using docker-compose. It makes communication between the containers much easier. Docker volumes are created where necessary to ensure data persist on the host. This was crucial when it came time to move to a new server. I was able to create extracts of the volume data and unpack it on the new server. The following articles helped me a lot when it came time to do this: [copying a docker volume from one host to another](https://rakhesh.com/linux-bsd/copying-a-docker-volume-from-one-host-to-another/) and [how to copy letsencrypt account including all certificates to a new server](https://blog.adriaan.io/how-to-copy-letsencrypt-account-including-all-certificates-to-a-new-server.html).

The last thing I want to mention in this section is a tool I used to help manage my docker environment. I use to use Portainer, and I like that tool a lot, except that I prefer to be a bit more hands-on over a GUI. I found this nice little replacement tool called [Lazy Docker](https://github.com/jesseduffield/lazydocker) that gives you insights into running containers, logs, images, and volumes all without ever having to leave the terminal. Now, whenever I set up a new server I add this as one of my installed packages.

### Monitoring

I come from an SRE/DevOps/Problem Management background so it would be uncharacteristic of me to set all this up without any monitoring to know how its performing. Initially, I installed [New Relic](https://newrelic.com/) and while they have a generous free plan (100gb data ingest/month), its also easy to exceed that if you start adding things like logs and process monitoring. I was also already very familiar with New Relic having used it heavily in my professional life for several years. I thought this could be a good opportunity for me to get my hands dirty in something new so I looked for an open-source solution and landed on Prometheus.

I’d already heard of Prometheus but usually in the context of monitoring Kubernetes environments. And while it does an awesome job of monitoring clusters, it can also be used to monitor small scale environments like my own. My Prometheus configuration can be found on my git repo linked above, but two key items are missing that I want to mention here. The first is the means by which I am visualizing these metrics. I am using a self-hosted Grafana instance running on a different server within my Proxmox environment. The reason for this is because on the same server, I am also running a centralized “Prometheus” instance which collects metrics from my different Prometheus instances using Prometheus federation. Before I discovered federation, I would install Prometheus on a of server using docker and than add it as a new data source in Grafana. This wasn’t really scalable and made it hard to facet data across different instances on a single dashboard. In some cases, I’d have the same dashboard duplicated for different servers which made no sense. Through the use of Prometheus federation and labels, I’m able to collect the metrics from my different servers under one data source in Grafana. Then I can have one dashboard that is filterable by the different instances using dashboard variables. Here are a few screenshots showing examples of Prometheus federation and Grafana.

#### Prometheus config file

![prometheus](prometheus.png){: width="800" height="500" }{: .left }

#### Grafana dashboard

![grafana](grafana.png)

I’m also using federation with [alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) so I can be alerted when certain metrics reach certain thresholds. These alerts are sent to my private slack workspace so I handle them accordingly. Additionally, I recently set up logging using Grafana [Loki](https://grafana.com/oss/loki/) but only on one server. I might go more in depth into my monitoring setup in a separate post but thought it was important to mention as part of this post as well.

### Closing

All told it probably took me 2-3 weeks to get my WordPress site into a “stable” state or rather a state where I felt comfortable enough to start making posts. I realized it doesn’t take a lot of compute resources to run this kind of site either, so I’d encourage anyone thinking about doing this to just go for it. I think even a rasberrypi could handle this. Now, there is one item I purposely left out and that’s how I setup my home networking to allow internet traffic to my home server and this website. I didn’t find that process super difficult once I figured out what I was doing, but I think the topic might make for a good second post so be on the lookout for that if your interested.
