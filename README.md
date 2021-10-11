# LXC-OperationPrivacy-VOIP
Notes on deploying https://github.com/0perationPrivacy/VoIP in a LXC container (interim to a Docker deployment)

This is **not** a fork of OperationPrivacy/VOIP.
This is a HOW-TO for install that repo in a container using LXC (https://linuxcontainers.org/lxc/introduction/).

## Context
I enjoy Michael Bazzell's OSINT podcast and learnt about the self-hosted sms portal app developed by @OperationPrivacy following Michael's nagging.
But ... without criticism ... I did not feel that the deployment option described https://inteltechniques.com/sms.html was suitable.
1. deploying to Heroku is not great for me (I am a committed self-hoster and the free Heroku can lead to a bigger paid account)
2. as a committed self-hoster, I was not keen on hosting the db element on mongodb.com

Most of my self-hosting is via Docker.  But there is no Diocker available (yet) for the app.
So I decided to go a different route to achieve containerisation : LXC.
It's not as well known or as widely used as Docker, but it is in many ways easier.
And in all likelihood you may well have LXC alrready installed if you're running Ubuntu 20.04.

So I documented process of deploying @OperationPrivacy's app in a LXC container, and hope it helps others.

DISCLAIMER : I am not a professional developer, and this guide is pitched at someone (me) needing a step-by-step process.

IMPORTANT : this is **not** a substitute for the instructions at https://github.com/0perationPrivacy/voip/wiki, more of a supplement.

- prepare LXC container
- install mongodb
- secure mongodb
- clone repo
- configure repo
- configure Telnyx
- run it

## run as ...
If you are logged in to your Linux box as a normal user, which you should be, you may need to prefix commands with `sudo`
YMMV (your mileage may vary).

## prepare LXC container
This is not a tutorial on LXC.  For that, see https://linuxcontainers.org/lxc/introduction/.
These are the basic steps you need to take

`lxc --version` : confirms that LXC is installed on your Linux instance.

`lxc launch ubuntu:20.04 <containername>` : creates a new Ubuntu 20.04 container with your chosen name.  First time may take a few minutes to download the OS image.  
`lxc exec <containername> -- /bin/bash` : enter the container as root (similar to normal Linux su <user>.  Note shell prompt should change.
 
`hostname` : should confirm you're in the right place
  
`apt-get update && apt-get upgrade -y` : ensures latest packages 

`apt autoremove -y` : tidy up
  
