---
title: "Upgrading Owncloud"
layout: post
date:   2019-01-03 00:00:00 +0100
categories: docker
---

This is the story of how I tried to upgrade my owncloud, got it wrong and then spent a few hours fixing it using Docker. I'm documenting it here to share the solution I came up with.

{% include warning.html content="If you run owncloud on Debian 8 and you have not yet started upgrading your owncloud server, do yourself a favour and make sure to upgrade to owncloud 9.1 _first_, before even thinking of upgrading to Debian 9!
" %}

If however, you did what I did and first upgraded to Debian 9 only to find out that owncloud refuses to upgrade over multiple major versions: read on.

## How it started

As I was figuring out OrgMode (which I hope to write a post about soon) I found out that my owncloud server was getting seriously long in the tooth. In fact, it was lagging so far behind that the latest owncloud app for android refused to connect to it.

Since I had some time on my hands around the holidays I figured I'd upgrade it, which also included upgrading my home server to Debian 9, since I was still running Debian 8.

The plan I had in mind was:

  1. Upgrade my server to Debian 9
  1. Run the owncloud upgrade wizard to upgrade the owncloud database and data folder
  1. Happily sync my stuff

The first step went smoothly enough: I followed the steps documented [here](https://linuxconfig.org/how-to-upgrade-debian-8-jessie-to-debian-9-stretch) and had no problems. Then I called up the owncloud URL, and clicked upgrade button only to be greeted by this:

{% include image.html url="assets/images/owncloud_upgrade.png" description="Exception: Updates between multiple major versions and downgrades are unsupported." %}


It turns out that owncloud will only upgrade one major version at the time. Before the upgrade I was running owncloud 7.0, but Debian 9 came with 10.0.10. This meant I'd need to follow the path 
```
7.0 → 8.0 → 8.1 → 8.2 → 9.0 → 9.1.8 → 10.0.10 
```
to bring my database and data folder up-to-date so my new owncloud install could use it.

## First attempt: Debian packages

After some searching I found [this list](https://owncloud.org/download/older-versions/) of older versions, which helpfully listed both Debian packages and tarballs. I figured I'd just grab the Debian packages for the older versions and run the updates one at a time.

This strategy failed though, since the packages for many of the older versions I needed were only available for Debian 7 and 8. I spent a little time trying to get them installed nonetheless, but gave up since it seemed likely to end with a broken server.

## Second attempt: tarballs

Well, owncloud is basically just a bunch of PHP files, right? So I don't really need the Debian packages, I can just grab the tarballs and run it that way!

This turned out to be correct in theory, except that Debian 9 comes with PHP7 and owncloud didn't start supporting PHP7 until version 8.2. Since I needed to upgrade to 8.0 and 8.1 first and I wasn't willing to try and force PHP5 on my new system, this turned out to be a dead end as well.

## Third attempt: Docker

But wait... running software that requires an older userland on a newer system: that sounds like a perfect usecase for Docker!

### The plan

For this to work, I'd need to run a Docker image that has Apache and owncloud, and then run it in the following way:

  * Shut down Apache
  * Run the Docker container in host-networking mode
  * Volume mount my owncloud data folder
  * Volume mount the MySQL (actually MariaDB socket
  * Volume mount my current owncloud config, so the wizard can use and update it

This looks roughly like this:

```shell
sudo docker run --net=host \
          -v /storage/owncloud/:/storage/owncloud \ 
          -v /var/run/mysqld/mysqld.sock:/var/run/mysqld/mysqld.sock \
          -v /var/www/owncloud/config/config.php:/var/www/owncloud/config/config.php \
          --name owncloud --rm some-owncloud-image
```

Since the Docker container uses the host networking, it will bind to the host's web ports.

{% include note.html content="The original intent behind running the container in host networking mode was that I intended to connect to MySQL over TCP. It turned out that didn't work, so I had to volume-mount the MySQL socket. After this, I could have chosen to only expose the container's web port." %}


### Finding a base-image

First I figured I'd head over to Docker hub and [grab an image](https://hub.docker.com/_/owncloud/), except they only start with owncloud 9. I wasn't ready to give up on this strategy yet though, so I decided if there was no public image I'd make my own.

I found a [very nice example](https://github.com/ulsmith/debian-apache-php) of a minimal Debian image which ran Apache and PHP5, so I used that as a base to expand upon.

### First upgrade: 7.0 to 8.0

The first upgrade would be to 8.0. 

For this I needed 3 files:


`start.sh` Start-script, copied verbatim from the Github example I used
```shell
#!/bin/bash

source /etc/apache2/envvars
exec apache2 -D FOREGROUND
```

`owncloud.conf` Apache config file to expose the owncloud site on `/owncloud`
```apache
<VirtualHost *:80>
Alias /owncloud "/var/www/owncloud/"

<Directory /var/www/owncloud/>
Options +FollowSymlinks
AllowOverride All

<IfModule mod_dav.c>
Dav off
</IfModule>

SetEnv HOME /var/www/owncloud
SetEnv HTTP_HOME /var/www/owncloud

</Directory>
</VirtualHost>
```

`Dockerfile` the Dockerfile to build the image

```docker
FROM debian:8
MAINTAINER Alex Kuiper <alex@nightwhistler.net>

RUN echo 'deb http://download.opensuse.org/repositories/isv:/ownCloud:/community:/8.0/Debian_8.0/ /' > /etc/apt/sources.list.d/isv:ownCloud:community:8.0.list
RUN apt-get update

RUN apt-get install -y apt-utils

## Install base packages
RUN apt-get -yq install \
                apache2 \
                php5 \
                libapache2-mod-php5 \
                curl \
                ca-certificates \
                php5-curl \
                php5-json \
                php5-odbc \
                php5-sqlite \
                php5-mysql \
                php5-mcrypt

RUN /usr/sbin/php5enmod mcrypt && a2enmod rewrite && mkdir /bootstrap

RUN DEBIAN_FRONTEND=noninteractive apt-get -y --force-yes install owncloud

ADD owncloud.conf /etc/apache2/sites-available/owncloud.conf
ADD start.sh /bootstrap/start.sh
RUN a2ensite owncloud
RUN chmod 755 /bootstrap/start.sh && chown -R www-data:www-data /var/www/html

RUN chown www-data:www-data /var/www/owncloud/config -R

EXPOSE 80
ENTRYPOINT ["/bootstrap/start.sh"]
```

{% include note.html content="Since this was a one-off image, I didn't bother to minimize the layers" %}

After that, building the image is a matter of

```shell
docker build . -t owncloud80
```

and then running it with:

```shell
sudo docker run --net=host \
  -v /storage/owncloud/:/storage/owncloud \ 
  -v /var/run/mysqld/mysqld.sock:/var/run/mysqld/mysqld.sock \
  -v /var/www/owncloud/config/config.php:/var/www/owncloud/config/config.php \
  --name owncloud --rm owncloud80
```

I opened up `/owncloud` in my browser, clicked "upgrade" and... success!

Now that this strategy had proven to work, it was a simple matter of rinse and repeat.
For the next steps I'll only show the changes that needed to be made to the Dockerfile.

### From 8.0 to 8.2

This step mostly changes the owncloud package source, and adds `apt-transport-https`

`Dockerfile`
```docker
FROM debian:8
MAINTAINER Alex Kuiper <alex@nightwhistler.net>

RUN apt-get update
RUN apt-get install -y apt-utils apt-transport-https

RUN echo 'deb http://download.owncloud.org/download/repositories/8.2/Debian_8.0/ /' > /etc/apt/sources.list.d/owncloud.list
RUN apt-get update

## Install base packages
...
```

### From 8.2 to 9.0

This step only changes the owncloud package source:

`Dockerfile`
```docker
...
RUN echo 'deb http://download.owncloud.org/download/repositories/9.0/Debian_8.0/ /' > /etc/apt/sources.list.d/owncloud.list
...
```

### From 9.0 to 9.1.8

This step also only changes the owncloud package source:

`Dockerfile`
```docker
...
RUN echo 'deb http://download.owncloud.org/download/repositories/9.1/Debian_8.0/ /' > /etc/apt/sources.list.d/owncloud.list
...
```

### From 9.0 to 10.0.10

This step was different, since now I had arrived at the point where the native Debian owncloud installation should be able to perform the upgrade. So, I turned on the Apache service again, went to the owncloud page and clicked on the "Upgrade" button.

This time I was rewarded with a successful migration... I had an up-to-date owncloud!

## Conclusions

Though this was an educational experience, I could have saved myself a lot of work by doing better research before I had started the upgrade.
Still, since there is a non-zero chance that I'm not the only person this could happen to I figured I'd write up my solution and hopefully save the next person some time.
