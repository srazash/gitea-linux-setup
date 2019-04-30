# GITEA SETUP GUIDE

By RSH, April 2019

## Installation

### Assumptions

This guide is intended to setup Gitea in a very specific way:

1. Gitea will be installed onto Ubuntu Server 19.04.
2. Gitea will be installed from a pre-compiled binary.
3. Gitea will us SQLite rather than a seperate SQL service (PostgreSQL, MySQL, etc.)
4. Gitea will be hosted behind a static IP and with a domain pointing to it.
5. Gitea will be accessed using HTTPS.
6. HTTPS will be protected with a **Let's Encrypt** SSL certificate.

Specifically, I will be setting up my Gitea instance on a hosted virtual server in the cloud but this guide *should* be applicable to other deployments types and you should be able to substitute various parts of this guide with your own methods (if you want to compile Gitea from source instead of using the provided binaries, for example).

### Server Setup

As stated I'll be using **Ubuntu Server 19.04** as the base for this server. As I'm using a VPS I can quickly deploy and access the server, but if you are setting up the server manually ensure you install OpenSSH Server duting the Ubuntu setup.

There are a couple of prerequisite tasks that will need to be performed to make the server ready for Gitea.

**Firstly** (as root) run:

``` bash
# update the server
apt-get update && apt-get upgrade -y
```

**Next** we'll want to check gitis installed. Run:

``` bash
git --version # if git isn't installed, run the command: apt-get install git -y
```

**Next** we'll need to enable and configure the firewall:

``` bash
# enable the firewall
ufw enable
# add service exclusions to the default zone
ufw allow 22/tcp # ssh was already excluded for me but you may need to add the rule
ufw allow 80/tcp
ufw allow 443/tcp
# add port exclusions to the default zone
ufw allow 3000/tcp # default http port of gitea, we'll remove this later
# check the firewall rules for the default zone
ufw status
```

At this stage the server setup is complete and we can move on to the Gitea setup.

### Gitea Setup

**Firstly** we'll need to download Gitea, for this guide we'll be using the latest binary version:

``` bash
# download and rename gitea
wget -O gitea https://dl.gitea.io/gitea/1.8.0/gitea-1.8.0-linux-amd64 # latest version at time of writing is 1.8.0, check on <https://dl.gitea.io/gitea/>.
```

We can validate the gitea binary using GPG to ensure it hasn't been modified:

``` bash
# download the associated asc for the gitea binary
wget -O gitea.asc https://dl.gitea.io/gitea/1.8.0/gitea-1.8.0-linux-amd64.asc
# import the gitea gpg key
gpg --keyserver pgp.mit.edu --recv 7C9E68152594688862D62AF62D9AE806EC1592E2
# verify the file using gpg
gpg --verify gitea.asc gitea
```

**Next** set the Gitea binary as executable before continuing:

``` bash
# make gitea executable
chmod +x gitea
```

**Next** we will need to setup a user for Gitea to use:

``` bash
adduser --system --shell /bin/bash --gecos 'Git Version Control' --group --disabled-password --home /home/git git
```

**Next** we'll create and configure Gitea's directories:

``` bash
mkdir -p /var/lib/gitea/{custom,data,log}
chown -R git:git /var/lib/gitea/
chmod -R 750 /var/lib/gitea/
mkdir /etc/gitea
chown root:git /etc/gitea
chmod 770 /etc/gitea
```

**Next** we'll configure Gitea's working directory:

``` bash
export GITEA_WORK_DIR=/var/lib/gitea/
```

**Next** move the Gitea binary to it's directory:

``` bash
mv gitea /usr/local/bin/gitea # you can als use cp instead of mv to copy the file
```

Gitea is now in a runnable state, but it won't work the way we want - a persistant service - in order to do this we will need to setup a service within systemd.

### Setup Gitea Service

**Firstly** create and edit the gitea.service entry for systemd:

``` bash
nano /etc/systemd/system/gitea.service
```

Copy in the this:

``` ini
[Unit]
Description=Gitea (Git with a cup of tea)
After=syslog.target
After=network.target
#Requires=mysql.service
#Requires=mariadb.service
#Requires=postgresql.service
#Requires=memcached.service
#Requires=redis.service

[Service]
# Modify these two values and uncomment them if you have
# repos with lots of files and get an HTTP error 500 because
# of that
###
#LimitMEMLOCK=infinity
#LimitNOFILE=65535
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web -c /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
# If you want to bind Gitea to a port below 1024 uncomment
# the two values below
###
#CapabilityBoundingSet=CAP_NET_BIND_SERVICE
#AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Save (CTRL + O) and exit (CTRL + X) nano. The Gitea service is now defined within systemd, in order to ensue the service runs when the server starts we'll need to enable the service:

``` bash
# mark the service as enabled
systemctl enable gitea
# manually start the service, this saves us needing to restart the whole server
systemctl start gitea
```

The service is now configured and active, you will be able to access Gitea via:

- http://***local-ip***:3000/
- http://***public-ip***:3000/
- http://***fully-qualified-domain-name***:3000/

Select **Register** from the landing page, this will start the Gitea configuration wizard. Based on the work we have done so far this should be completed with **SQLite3 selected as the database type**.

Most option are safe to leave as their defaults, but the following should be reviewed:

- SSH Server Domain (set to your domain or IP address)
- Gitea Base URL (set to your domain or IP address)
- Enable OpenID Sign-In (recommend disable, will also disable OpenID registrations)

Once done we will want to lock down the configuration file so no further changes can be made from Gitea:

``` bash
chmod 750 /etc/gitea
chmod 644 /etc/gitea/app.ini
```

We can still make changes from the server using nano, which we will be doing shortly.

We now have a fully operational Gitea server. However, it's still not in an optimal state. We're running on HTTP - an insecure protocol, to resolve this we will configure Gitea to run on HTTPS and automatically secure itself using Let's Encrypt.

By default we won't be able to bind ports 80 (needed to issue the Let's Encrypt certificate) or port 443 to Gitea.

On Linux, ports below 1000 are considered **privileged** and can only bind to a process executed by the root user, in order to allow Gitea to bind to port 80 and 443 run the following command:

``` bash
setcap 'cap_net_bind_service=+ep' /usr/local/bin/gitea # this will need to be re-run each time Gitea is upgraded
```

Now we'll make some changes to the **app.ini** file to enable HTTPS and Let's Encrypt certification:

``` bash
nano /etc/gitea/app.ini
```

We can see various sections, look for the one starting with ```[server]```. We're going to want to add a few lines to the top of this section and modify some existing settings. Add the following lines as shown below:

``` ini
PROTOCOL=https
ENABLE_LETSENCRYPT=true
LETSENCRYPT_ACCEPTTOS=true
LETSENCRYPT_DIRECTORY=https
LETSENCRYPT_EMAIL=<your@email.address>
```

We're also going to change two other existing lines, look for (they should be next to each other):

``` ini
HTTP_PORT   = 3000
ROOT_URL    = http://<server-address>:3000/
```

And change them to:

``` ini
HTTP_PORT   = 443
ROOT_URL    = https://<server-address>/
```
Changing the HTTP_PORT to '443' and changing the ROOT_URL to start with 'HTTPS://' instead of 'HTTP://' and removing the port number from the address, including the colon ':3000'.

Save (CTRL + O) and exit (CTRL + X) nano. Restart the Gitea service:

``` bash
systemctl restart gitea
```

We'll now be able to access Gitea using HTTPS, and we should see a 🔒 **green padlock** icon, indicating the connection is secured using a Let's Encrypt SSL certificate.

We're pretty much done now. Just one last job, a bit of housecleaning and security tightening. Remove port 3000 from the firewall rules:

``` bash
ufw delete allow 3000/tcp
```

Confirm that only ports 22, 80 and 443 are open with:

``` bash
ufw status
```

These are required for SSH, Let's Encrypt certificate issuance and access Gitea using HTTPS, respectively.

It may also be worth removing any leftover files you downloaded from the home (~) directory.

Once done, you are ready to go!