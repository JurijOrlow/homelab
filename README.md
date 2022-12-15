# Full homelab config tutorial
## Bitwarden, Nextcloud and Heimdall hosted on premise

*Version for ARM processors, like RaspberryPi or Oracle Cloud A1 architecture*

---

This repository contains docker files and instructions needed to run your very own on premise homelab, complete with cloud-accessible Bitwarden password manager, cloud-accessible Nextcloud storage and office suite, and Heimdall for providing nice and coherent dashboard for your lab. Everything secured by SSL connections - certificates generated via Letsencrypt.

---

**IMPORTANT NOTICE**

Although it's the setup I personally use and am pretty confident with it (SSL secured connections, proxy and no needlessly opened ports) **I DO NOT HOLD ANY RESPONSIBILITY ABOUT YOUR DATA SECURITY**! Everything you do here is done at your own risk!

---

With that out of the way, first we will need suitable machine to run our homelab on. You can do it on your hardware, in this case recommended specs are:

- CPU: Decent CPU with at least 4 cores (something like i5 4th Gen would be suitable)
- RAM: At least 8 GB, the more the better
- Disk: SSD would be best, about 20-30 GB for system + as much as you want for your Nextcloud storage
- Network: the faster, the better, 1 Gbps cable connection would be ideal
- OS: I recommend Ubuntu Server, but pic your poison

I personally didn't run this on hardware, so the part about static public IP, NAT forwarding and mouting your physical drives for Docker and Nextcloud to use you have to figure yourself. I used Oracle Cloud Always Free Tier instance. Specs I chose for it:

- CPU: Ampere 2 CPU units
- RAM: 12 GB
- Disk: 120 GB boot volume
- Network: 2 Gbps link
- OS: Ubuntu Server 22.04 (latest LTS at the moment of writing this)

With that everything is fast and snappy, uploading files I am able to reach up to 200-300 Mbps upload via Wi-Fi. Also this configuration can run 24/7 and is good for Always Free Tier, and leaves 2 CPU, 12 GB RAM and 80 GB of storage for your other instances.

---

## Prerequisites
0. Basic knowledge about using Linux command line

Without it you won't be able to do this. I will try to hold your hand, but will use a lot of shortcuts and skip obvious things.

**NOTE FOR ORACLE CLOUD!**

Before anything you need to configure your VCN and subnet! Without it you won't be able to do anything. I won't be explaining it here, as it's easy, you do it in "Networking" section of your Oracle Cloud dashboard.

1. Updating system

It will prevent you from frustration late:

```bash
apt update && apt upgrade -y
```

Then reboot. Of course, if you're using something else than Ubuntu - figure that out.

2. Changing password

This step is CRUCIAL if you are on Oracle Cloud. You need to set up password for user `ubuntu`. Why is that important? Becouse if you screw something up with ports, firewall or ssh daemon and won't be able to access server via SSH you could still access it via online console. Without it, you would have to delete whole instance and start over. So after connecting with SSH simply run:

```bash
sudo passwd ubuntu
```

Also important - all of things here will be done on root acount, so:

```bash
sudo su
```


3. Preparing firewall

First of all, you will need to pick 4 ports - SSH, Heimdall, Bitwarden and Nextcloud. They need to be outside of well-known TCP ports. For more info go [here](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers). Next you have to edit `/etc/iptables/rules.v4` to add these lines:

```bash
-A INPUT -p tcp -m state --state NEW -m tcp --dport <ssh-port> -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport <bitwarden-port> -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport <nextcloud-port> -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport <heimdall-port> -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
```

Then simply:

```bash
iptables-restore < /etc/iptables/rules.v4
```

You will also need to open these ports for mask `0.0.0.0/0` in Security List of your subnet in Oracle Cloud dashboard. It's easy, you can do it. If you're using something else, or do it on hardware - keep that in mind, these ports have to be available from outside.

4. Changing SSH port

First order of business is changing default SSH port from 22 to something else. To do this in OS you can either edit `/etc/ssh/sshd_config` file or create new file, like `/etc/ssh/sshd_config.d/new.conf` (name is not important, but it must have `.conf` extension). there you edit/add this line:

```bash
Port <ssh-port>
```

Then you have to restart ssh daemon:

```bash
systemctl restart sshd
```

**VERY IMPORTANT!**

BEFORE YOU DO ANYTHING ELSE, OPEN SECOND SSH CONNECTION WITH THIS NEW PORT AND MAKE SURE YOU CAN CONNECT! IF NOT, SOMETHING'S WRONG WITH EITHER FIREWALL ON INSTANCE OR ON YOUR PROVIDER (ORACLE, SHELLS, LINODE OR YOUR ROUTER). IF YOU CAN'T ACCESS, GO BACK TO PREVIOUS CONSOLE AND DELETE THAT LINE WITH PORT, RESTART DAEMON AND FIGURE OUT WHAT'S WRONG!

If you really f up, and completly lose access to server, you can always use cloud console on Oracle, using password we set in 1st step.

5. Installing Docker

I won't be going into details here, Docker has great instruction on their website, so I will just point you to [THAT](https://docs.docker.com/engine/install/).

6. Configuring domain

Here comes the part you have to pay a little. You need to buy a domain to use this all and access from internet. From my search in Poland cheapest domains offers [home.pl](https://home.pl/domeny/) - at about 5-6 PLN per year (then it gets expensive, but you can always buy another one right?).

When you have your domain, you have to create DNS A records for your services. I think the best scheme is:

- `<your.domain>` for Heimdall
- `bitwarden.<your.domain>` for Bitwarden
- `nextcloud.<your.domain>` for Nextcloud

But you do you, nobody prevents you from having your password manager under `dupajasia.<your.domain>`. DNS A records all need to point at your server's public IP address.

---

## Installing services

Here comes this repository. Just clone it under `/opt/`:

```bash
cd /opt
git clone <repo-url>
```

Now you have `docker` directory and in that four directories:

- `bitwarden` for Bitwarden docker files
- `nextcloud` for Nextcloud
- `heimdall` for Heimdall
- `caddy` for Caddy, which serves as a reverse proxy and SSL provider

For the rest of tutorial I will be refering to this scheme, where all files are under `/opt/docker/`. 

1. Caddy

[Caddy GitHub](https://github.com/caddyserver/caddy)

First let's start with Caddy. It's a nifty, little container, that serves mainly as our reverse proxy. By default when you type domain name in the browser it points to HTTP/HTTPS ports - 80 and 443. Since we can't have more than one service on these ports (and we will have 3 services) we will be using ports you set up in step 1. And normally to access services you would need to specify this port in browser, like `https://bitwarden.<your.domain>:<bitwarden-port>/`. Not very elegant. Reverse proxy takes 80 and 443 requests from specified domains and reroutes them to port we need, so when we just type `https://bitwarden.<your.domain>` (and it directs us to port 443), we actually connect to the port we specify in config.

As an added bonus, Caddy also generates and auto regenerates SSL certs from Let's Encrypt for your domains, so we can use secure SSL connections (and for example Bitwarden needs it to work with browser extension or mobile app).

To config this you need to edit `/opt/docker/caddy/cddata/Caddyfile` to provide domains and ports you configured earlier. Simply replace <\*-domain> with respective domain name for each service, and <\*-port> with respective port for each service. It's not rocket science.

```bash
https://<heimdall-domain>:443 {
	reverse_proxy localhost:<heimdall-port>
}
```


2. Heimdall

[Heimdall GitHub](https://github.com/linuxserver/Heimdall)

Heimdall is also a "quality of life" type of thing. It provides us with customizable dashboard, where we can pin our applications, like a starting page of a browser, for many, many services, that you would want to use in the future. Added bonus above typical starting page (becouse you can also just pin your services addresses to your Chrome/Firefox starting page) is that it integrates with services. For example, for Nextcloud it can show available or used space, right there on the dashboard.

The only change you have to make here is edit `/opt/docker/heimdall/docker-compose.yml` and replace `<heimdall-port>` in "ports" section with the port you specified in step 1.

```bash
ports:
	- <heimdall-port>:80
```


3. Nextcloud

[Nextcloud AIO GitHub](https://github.com/nextcloud/all-in-one)

Nextcloud is too big to explain here, so just read about it [HERE](https://nextcloud.com/). We will be using Nextcloud AIO image, which provides us with easy and fast process of setting everything. If you want to use full manual version, find it on [GitHub](https://github.com/nextcloud/server). And good luck.

Only config we have to make is change port on which Apache container serving our actual drive will be run. To change this we need to open `/opt/docker/nextcloud/docker-compose.yml` and replace `<nextcloud-port>` in enviroments of Docker file to port we specified earlier.

```bash
environment:
	- APACHE_PORT=<nextcloud-port>
```

4. Bitwarden

[Bitwarden GitHub](https://github.com/bitwarden/server)

Bitwarden is also big, so read about it [HERE](https://bitwarden.com/). We'll be using docker-unified method, since it's the only one that works both on x86_64 and ARM processors. Important note here - docker-unified is still in beta, so minor bugs can occur. Should be good, but no guarantee. If you're using x86_64 architecture, you probably should use "classic" method.

Here we have the most to configure. First let's hop to Docker file - `/opt/docker/bitwarden/docker-compose.yml`. We will be using MariaDB database for our passwords, with password protected user and random root password. We need to edit two lines:

- first replace `<bitwarden-port>` in "ports" section with specified earlier port

```bash
ports:
	- "<bitwarden-port>:8080"
```

- next change `<db-password>` with strong, complicated, random password. Don't worry, you won't have to remember it, just paste it in other config file, so I recommend 32 characters random generated password

```bash
MARIADB_PASSWORD: "<db-password>"
```

Write, quit. Open env file - `/opt/docker/bitwarden/settings.env` and change following lines:

```bash
BW_DOMAIN=<bitwarden-domain>
.
.
.
BW_DB_PASSWORD=<db-password>
```

I'm pretty sure you know what to put there. Next you need to go to https://bitwarden.com/host/, provide your e-mail address and get an installation ID and installation key. These are random and e-mail connected. Why they require those? Read the docs ;)

After you have ID and key, you need to put them in `settings.env` file:

```bash
BW_INSTALLATION_ID=<id>
BW_INSTALLATION_KEY=<key>
```

Last step is to configure SMTP e-mail. Yes, you can install another service on your homelab, that will provide you with nice emails, ending with `@<your.domain>` but I didn't do that. Maybe in some later point in the future I will, but now I don't feel the need - I will update repo and this tutorial if that ever happens. I just used my GMail account to send me e-mails as me. If you're using something else, you need to find these configs by yourself.

- SMTP Host: smtp.gmail.com
- SMTP Port: 587
- SMTP Username: your username
- SMTP Password: your password (if using 2FA you need to go to your Google Account settings and generate an Application Password)

You can also create separate e-mail address for Bitwarden (and Nextcloud) with some provider - gmail, yahoo - but I think it's a bit overkill for a homelab. Your private e-mail is sufficient. Of course you change it in `/opt/docker/bitwarden/settings.env`:

```bash
# Mail
globalSettings__mail__replyToEmail=noreply@$BW_DOMAIN
globalSettings__mail__smtp__host=
globalSettings__mail__smtp__port=
#globalSettings__mail__smtp__ssl=false
globalSettings__mail__smtp__username=
globalSettings__mail__smtp__password=
```

After that our services are 95% configured.

---

## Starting and managing Docker services

First of all - useful commands:

```bash
docker compose up -d #start Docker compose file
docker compose down #stop Docker compose file
docker ps #list running containers
docker system prune -a #clean up - remove everything not connected to running containers INCLUDING stopped containers
docker volume prune #clean up Docker volumes not connected to any running containers
```

Docker compose must be always run from directory where `docker-compose.yml` is located. You can change any setting and play with it, but after every change you have to stop container with `docker compose down` and start it back again.

So to start our services we need to go to every directory and start them. First launch will be slow, becouse Docker needs to pull images and create volumes:

```bash
cd /opt/docker/heimdall
docker compose up -d
cd /opt/docker/caddt
docker compose up -d
cd /opt/docker/bitwarden
docker compose up -d
cd /opt/docker/nextcloud
docker compose up -d
```

---

## Further configuraion - Nextcloud

One last step is to configure Nextcloud. Since we used Nextcloud AIO, we need to actually use it to launch "normal" Nextcloud. You need to head to `https://<your-nextcloud-domain>:8080/`. You should see huge Nextcloud logo and box with password consisted of several words separated with space. **KEEP THIS PASSWORD, AS THERE'S NO WAY TO RECOVER IT!** After you copy it, you're redirected to login page, where you paste that password. And then you're in AIO control center. 

First you need to paste your Nextcloud domain - if you're following my example `nextcloud.<your-domain>` and click Save. It should check it and point you to next page, where you actually manage Nextcloud containers. Below you can configure your timezone, check what Nextcloud features you want and click "Start containers". It will take about 5-10 minutes, and when all dots are green you are given credentials to Nextcloud admin account. **SAVE THIS PASSWORD, AS BEFORE YOU CONFIGURE SMTP IN NEXTCLOUD THERE'S NO WAY TO RECOVER IT!**

Under credentials you have link to your Nextcloud instance. Go there, log in, and open Preferences. Under Personal > Personal Info you have to fill at least your e-mail address, then under Administraion Settings > General Settings you need to configure mail server. Settings are exactly the same as in Bitwarden. Authentication method - Login, Authentication required. After that you can click "Send e-mail" below to test. If e-mail reaches you - you're done!

---

## Final product

If you done everything correctly (and I didn't mess up in this tutorial) now you should be able to just open your browser, type `https://<service>.<your-domain>` and go to your desired service. All that was left to do is add your apps to Heimdall main dashboard, create Bitwarden accout and upload some files to Nextcloud.

One final request - please don't create Issues asking me to help you with setting up. Better go to each project's respective GitHub repo and ask there, you will get answer from software creators. I am just some guy, who after 2 weeks managed to make this and wanted to share with you.

---

## ALL USED CODE IS OWNED BY ITS RESPECTIVE CREATOR. I DIDN'T MAKE, CREATE ANY OF SOFTWARE USED HERE. LINKS TO EACH PROJECT'S RESPECTIVE GITHUB REPO IS PROVIDED.
