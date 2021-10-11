# LXC-OperationPrivacy-VOIP
Notes on deploying https://github.com/0perationPrivacy/VoIP in a LXC container (interim to a Docker deployment)

This is **NOT** a fork of OperationPrivacy/VOIP.
This is a HOW-TO for installing that repo in a container using LXC (https://linuxcontainers.org/lxc/introduction/).

---
## Context
I enjoy Michael Bazzell's OSINT podcast and learnt about the self-hosted sms portal app developed by @OperationPrivacy following Michael's nagging.
But ... without criticism ... I did not feel that the deployment option described in https://inteltechniques.com/sms.html was suitable.
1. deploying to Heroku is not great for me (I am a committed self-hoster and the free Heroku can lead to a bigger paid account)
2. as a committed self-hoster, I was not keen on hosting the db element on mongodb.com

Most of my self-hosting is via Docker.  But there is no Docker available (yet) for the app.
So I decided to go a different route to achieve containerisation : LXC.

It's not as well known or as widely used as Docker, but it is in many ways easier.
And in all likelihood you may well have LXC already installed if you're running Ubuntu 20.04.

So I documented my process of deploying @OperationPrivacy's app in a LXC container, and hope it helps others.

DISCLAIMER : I am not a professional developer, and this guide is pitched at someone (me) needing a step-by-step process.

IMPORTANT : this is **NOT** a substitute for the instructions at https://github.com/0perationPrivacy/voip/wiki, more of a supplement.

- prepare LXC container
- install mongodb
- secure mongodb
- clone repo & build
- configure repo
- configure Telnyx
- run it

---
## run as ...
If you are logged in to your Linux box as a normal user, which you should be, you may need to prefix commands with `sudo`.

YMMV (your mileage may vary).

---
## prepare LXC container
This is not a tutorial on LXC.  For that, see https://linuxcontainers.org/lxc/introduction/.

These are the basic steps you need to take to fire up a container.

`lxc --version` : confirms that LXC is installed on your Linux instance.

`lxc launch ubuntu:20.04 <containername>` : creates a new Ubuntu 20.04 container with your chosen name.  First time may take a few minutes to download the OS image.  

`lxc exec <containername> -- /bin/bash` : enter the container as root (similar to normal Linux su <user>).  Note shell prompt should change.
 
`hostname` : should confirm you're in the right place
  
`apt-get update && apt-get upgrade -y` : ensures latest packages 

`apt autoremove -y` : tidy up
  
---
## install latest mongodb
Ubuntu has an old version of mongo db (v3.x).  You probably want the latest version.
For full details, please read the excellent guide at https://www.digitalocean.com/community/tutorials/how-to-install-mongodb-on-ubuntu-20-04
 
`curl -fsSL https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -`
 
`echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list`
 
`apt-get update -y`
 
`apt install mongodb-org -y`
 
`systemctl start mongod.service`

`systemctl status mongod`
 
That should show a running mongo instance.  If not, check the detailed guide linked above.
 
`systemctl enable mongod`
 
`mongo --eval 'db.runCommand({ connectionStatus: 1 })'`
 
`systemctl status mongod`

---
## secure mongodb
You probably want a secure mongo db.
The following is an abstract of excellent guide at https://www.digitalocean.com/community/tutorials/how-to-secure-mongodb-on-ubuntu-20-04

 `mongo` : enters interactive shell - enter these commands line by line
  ```
	show dbs
	use admin
	db.createUser(
	{
	user: "mongoadmin",
	pwd: passwordPrompt(),
	roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
	}
	)
 ```
closing bracket complete the command

`nano /etc/mongod.conf`. : <blatant plug : use `ne` editor, it's much nicer, `apt install ne -y`>

	-->	uncomment line `#security:`

	-->	add line `authorization: enabled`
 
`systemctl restart mongod`
 
`systemctl status mongod`
 
`mongo -u mongoadmin -p --authenticationDatabase admin`
 
`show dbs` : confirms you can see databases currently installed (nothing much) 

 `exit`

---
## clone VoIP repo & build
Now we can get going ...

`git clone https://github.com/0perationPrivacy/VoIP/`

`cd VoIP`
 
`apt install nodejs -y`
 
`apt install npm -y`
 
`npm install`

---
## configure repo
The following is not particularly clear (at time of writing anyway) from the official guides.
	
And it is the magic which makes it all work (well, webhook in Telnyx also see below). 

`nano .env` or use your favourite editor (cough : `ne`)

ensure that the following is set up (not specified in the rep sample .env) : you can leave the other lines
```
DB = mongodb://<db_admin_name>:<db_password>@localhost/admin
BASE_URL = https://<domain>.<tld>/
```

NB : as per Troubleshooting, ensure the `BASE_URL` has protocol (hhtps) not just url, and ensure it has trailing `/`

---> Now `exit` from your container back to host VPS <---

---
## nginx reverse proxy
- NB : this is done in your host VPS : check you are not still in your container
- set up your DNS to point your domain or subdomain to your VPS
- your container is bridged auto-magically when the container is built and launched
- but that's only fine for outbound traffic : you need inbound traffic (doh!) as well
- your installation may be different but in my case I need `nginx` running on the VPS and then a `nginx` conf file to direct traffic to the container
- do `lxc list` to get the local ip address of your container : you need it for the nginx conf file
- create the conf file :  `nano /etc/nginx/sites-available/<domain>.<tld>`
- sample nginx conf file below pre-SSL enabling
- enable it : `ln -s /etc/nginx/sites-available/<domain>.<tld> /etc/nginx/sites-enabled/<domain>.<tld> `
- check config : `nginx -t`
- reload config : `systemctl reload nginx`
- do your certbot dance to get https cert (if your `.env` is set up with https in BASE_URL

sample nginx conf file as BEFORE you do your certbot dance to get https enabled :
```
server {
       listen 80;
       listen [::]:80;

       server_name <domain>.<tld>;

       location / {
                proxy_pass http://<conainter_ip>:3000;
                proxy_http_version 1.1;
                proxy_cache_bypass $http_upgrade;
        # Proxy headers
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        "upgrade";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  $server_port;
       }
}
```

---
## configure Telnyx
Sorry, I don't know about Twilio.  You're on your opwn there, but it is probably similar.
	
Telnyx is much easier anyway (and cheaper).

Check out Michael's guide for instructions on opening a Telnyx account (https://inteltechniques.com/sms.html).

But note the following :
- you need to buy a number (doh!) and it should be SMS enabled : pick your region
- you need to Get at least L1 verified to use it, and you may well need L2 verification if you're not in US
- L2 verification needs 24-48 hours, but if you leave it say 12 hours and talk by chat very nicely to support, you may be lucky and they do it then & there.
- you need to create a messaging profile and specify a web hook in the format `https://<domain.tld>/webhooks` 
- ensure the mesaging profile is set to APIV2 (see also Troubleshooting notes in OperationPrivacy/VoIP wiki
- you need to attach your number to the messaging profile (in the Numbers section of Telnyx dashboard)

---
## run it !
You should be all set to fire it up.

Go back into your container : `lxc exec <containername> -- /bin/bash`

`cd VoIP`
	
`node app.js &`

Then visit your installation URL, sign up for an account (your own), and add a Profile.

You will need the API key from your Telnyx account, and it should show your number(s) available.
	
Test it.

If you need to stop the app, the brutal way is `killall node`.  Because you're running in a LXC container, you should not affacted anything else.

If you leave the termoinal running, you can see status messages when you interact with the browser.

If you ensure you yse the trailing `&`, you can exit from the container and your VPS terminal, and the app should continue running.

---
## notes 
- I haven't used the app for voice, so no knowledge here
- I haven't tested all aspects of the app (e.g. attachments)
- Telnyx has rate limits to avoid sending abuse
- **the big one** : config changes in the app, e.g. the `.env` file, need the profile in the browser to be deleted and re-created.  This will lead to past messages being lost.  Save them somehow if you need them.
- I haven't tested any keepalive measure to ensure the app stays running (check out notes at bottom of Michael's instructions).  I am not convinced they are needed but hey, what do I know.

I hope this is helpful.
Let me know any corrections, omissions or questions.
                                    
