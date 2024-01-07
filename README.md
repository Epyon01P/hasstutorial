# hasstutorial
[![CC BY 4.0][cc-by-shield]][cc-by]

How to install Home Assistant as a stand-alone Python application on a Raspberry PI, or any other device running a Debian-based Linux distro, and make it securely accessible from the internet through Nginx.

## About
This guide will show you how to install the [Home Assistant](https://www.home-assistant.io/) home automation software as a stand-alone Python application on a device running a Debian-based Linux distro, like Raspberry Pi OS, and how make it accessible from the internet in a (pretty) safe way, e.g. to use with the Home Assistant companion app even when you are away from home. It will also show you how to install other useful tools and how to maintain and update your Home Assistant.

The instructions below are mainly geared towards Raspberry Pi users, but are suitable for any Debian-based Linux distribution, e.g. Ubuntu.

## Table of contents
[Why run Home Assistant standalone?](#why-run-home-assistant-standalone)

[Setting up your Raspberry Pi](#setting-up-your-raspberry-pi)
- [Flashing the OS image](#flashing-the-os-image)
- [Accessing your Pi over SSH](#accessing-your-pi-over-ssh)

[Installing Home Assistant](#installing-home-assistant)
- [Installing Home Assistant Core]
Configuring Home Assistant as a service	7
Updating Home Assistant	8
Installing mosquitto	9
Making your Home Assistant remotely accessible	11
Installing Nginx	11
Getting a dynamic DNS	13
Setting up your first site	15
Securing your site with a SSL/TLS certificate	17
Redirecting your domain to Home Assistant	18
Upgrading Python	20

## Why run Home Assistant standalone?
Why go through all the hassle of using Home Assistant as a standalone application when there are prebuilt images like Home Assistant OS?

Well, using Home Assistant OS means dedicating your entire shiny new Raspberry Pi to run Home Assistant, and Home Assistant only. While Home Assistant, in essence, is just one Python application. 

Perhaps you want to use your Pi for other stuff as well? Using it as a NAS, running a media server, hosting a website, using the Pi-hole advertisement and Internet tracker blocking application, running a NVR, an always-on torrent client? You name it. When running a dedicated OS like Home Assistant OS, you will need at least one other Raspberry Pi or similar device for that. And sure, you could run Home Assistant in a docker container, but that isn’t really (power)efficient, especially on a Pi. 

When you run Home Assistant as a standalone Python application, you have your hands free to do all these other things with your Pi, and then some. And you might pick up some valuable Linux skills along the way.

## Setting up your Raspberry Pi
### Flashing the OS image
This part of the guide is written for a model B Raspberry Pi 3 or 4, but the 5 will work too (even better). I do not recommend older generations of Pi, nor the Zero variants. It might work, but YMMV.

First, Install Raspberry Pi OS using [the Raspberry Pi Imager](https://www.raspberrypi.com/software/). You will need an empty micro-SD card of at least 16GB, possibly a micro-SD tot SD card adapter, and the Raspberry Pi Imager tool. Put the SD card into your laptop or computer.

Install the imager tool and run it. If you are using a Raspberry Pi 4 or 5, select it in the Raspberry Pi Device dropdown. If you are using a Raspberry Pi 3, select `No Filtering`. 

You now have a few options to select a suitable Operating System.

- If you have a Pi 4 or 5 with more than 4GB of RAM, Raspberry Pi OS (64-bit) is recommended.
- If you have Pi 4 or 5 with 4GB or less of RAM, Raspberry P OS (32-bit) might be more suited.
- If you have a Pi 3, you must use Raspberry P OS (32-bit).

A 64-bit OS is required to make proper use of RAM exceeding 4GB in size, and of the 64-bit capabilities of the Pi 4 and 5. The downside is possible compatibility issues with some software. In general, I would tend to go with the 64-bit OS. 
If you have a Pi 3, you must use the 32-bit OS, as the Pi 3 doesn’t have 64-bit capabilities.

This guide has been written based on a 32-bit OS.

Select the drive letter of your SD card as the Storage target. Click next.

When prompted to apply custom settings, click `Edit Settings`. In the following dialog box, you can configure various settings which will be flashed to the card. We will need the following configuration:

Under **General**, set *hostname* to `homeassistant`, or any other name you like (e.g. `homeserver`). This is the “name” your Pi will get on your local network and, if your router properly supports mDNS, you can use to access the Pi without having to find its IP address.

Also set a custom username and password. Make sure the password is strong enough yet easy to remember. Do **not** use `pi`, `raspberry`, `root`, `admin` or any combination of these for the username or password.

Configure your local time zone and keyboard layout under locale settings. We will be using a remote terminal in this guide, but you might want to hookup a screen and keyboard to your Pi later.

Under **Services**, enable SSH, using password authentication. We will use this for remotely accessing the Pi.

Optionally, you can also make the Pi connect to your local network over WiFi by configuring the wireless LAN settings. I recommend a wired Ethernet connection though.

When you’re done, click `Save`, then `Yes`. If prompted to erase all existing data on the card, click `Yes`. Now grab a cup of coffee while the imager writes the Raspberry Pi OS to disk.

When the SD card has been verified by the imager, ignore any Windows prompts to format the card again, but remove it from you PC or laptop and put it into the micro-SD slot of the Pi. Remember, the coppery pads of the card need to face the Pi’s mainboard.

If you are not using the WiFi connection (recommended), connect the Pi to your router with an Ethernet cable. Plug in a decent power supply (5V 2A is recommended for the Pi 4) and let the Pi boot up.

You might need to press the power button on a Pi 5.

### Accessing your Pi over SSH
Return to your PC or laptop. We will be controlling the Pi headless, which means without a keyboard and screen attached to it. Rather we will be using Secure Shell, or SSH, to control the Pi over the local network. SSH allows us to open a command-line shell, which is a text-based interface, to the Linux-based operating system running on the Pi. 

Very soon now, you will feel like a real hacker.

In this guide, we will be using the venerable [PuTTY SSH client](https://www.putty.org/). You will probably want the 64-bit x86 Windows Installer. Install and run it.

Now comes the trickiest part of this guide, which you unfortunately will have to figure out yourself. To access your Pi, you will need to know its IP address on the local network. Each device on your local network has an IP address, but finding a list of these addresses requires you to login to the web interface of your router. This interface can be accessed either through it’s gateway IP (mostly `192.168.1.1`), or often through a portal of your ISP. 

Either way, you should eventually find a list of IP addresses active on your local network, hopefully with one of them showing the `homeassistant` moniker, or whatever you put into the hostname field of the imager tool. Otherwise, you will need to try them all. If your router supports mDNS, you could also try the `homeassistant.local` hostname.

Open up Putty. Under *Host Name (or IP address)*, type in the configured hostname (e.g. `homeassistant.local`) or IP address of your Pi on your local network. Make sure the Port is set to **22**. Click *Open*. If the connection times out, the Pi hasn't finished booting yet, or you used the wrong IP address.

When prompted to accept a new host key, click *Accept*. Now login with the credentials (username & password) you configured in the imager.

Congratulations! You are now talking with your Pi over the network. Time to get down and dirty.

From now on, we will be using text-based commands to talk to the Pi. You can just type or copy/paste these commands into the terminal (it is called a ‘command line interface’ (CLI) for a reason) followed by an Enter. To paste text or commands copied from your laptop or PC in a PuTTY shell, just right click. Also, selecting text by highlighting it in PuTTY copies it to the clipboard.

## Installing Home Assistant
First, we will update the Pi so it has all the latest software. Fetch the update list with

`sudo apt-get update`

Next, install the updates with

`sudo apt-get upgrade`

Wait for that to complete, answer any prompts with 'y' + Enter. Your system is now up-to-date. Reboot your Pi just to be sure.

`sudo reboot`

If you are using PuTTY, click OK on the dialog box warning you of a sudden disconnect. 

Wait a minute or two, then right-click the PuTTY (inactive) title bar and select Restart Session, or follow the steps above to initiate a new connection. 

Login with your username and password. You are now set to install additional software!

### Installing Home Assistant Core
In essence, Home Assistant is just a Python application. That means there are multiple ways to get it up and running on a target device. Most people use Home Assistant OS (HAOS), which is a complete Linux-based operating system containing Home Assistant and supporting tools. You just flash it to an SD card, like we did before, and poof you’ve got Home Assistant in your home. 

HAOS, however, runs Home Assistant and only that. That means you’re dedicating an entire Raspberry Pi just to run a Python application. If you want to do other things, like hosting media files, a web server, an NVR etc. you need to buy another Pi and feed it power, too.

In this guide we are going to install **Home Assistant Core**, which is the core Python application that is Home Assistant. You can find an overview of the differences between every type of installation [here](https://www.home-assistant.io/installation/#compare-installation-methods). 

The main difference is that Home Assistant Core has no supervisor, meaning you will have to update Home Assistant manually. The chart also says you can’t have add-ons or configuration restore, but that is only half true. This guide will show you how to do this on Home Assistant Core too.

To install Home Assistant Core, we need Python 3.11 or higher. Chances are your Raspberry Pi OS came with an older version. To find out which version you have, use the command

`python -V`

If this yields a Python version equal or higher to 3.11, you’re good to go. If not, see Upgrading Python.

We are now ready to install Home Assistant. First, we’ll download some additional tools which are required to install Home Assistant.

`sudo apt-get install libjpeg-dev zlib1g cmake libopenblas-dev`

Instead of installing Home Assistant under our own user account, we are going to create a new user exclusively for Home Assistant. This way, Home Assistant lives in a different location, safe from you accidentally deleting your home folder or creating other unwarranted mayhem which might impact its performance.

Add the new user

`sudo useradd -rm homeassistant -G dialout,gpio,i2c`

We will install the Home Assistant Core Python application in a virtual environment, or *venv*. A venv is kind of like an isolated copy of an existing Python installation, where you can mess around with installing additional libraries or downgrading packages without botching the rest of the system. If you really end up turning the venv into a hot mess, you can delete it and start over fresh. It’s not a virtual machine, but what Python’s concerned, it’s pretty close.

Go to the directory where the venv will live and create it

`cd /srv`

`sudo mkdir homeassistant`

Change the ownership of the directory to our homeassistant user

`sudo chown homeassistant:homeassistant homeassistant`

Switch our user account to the homeassistant user. You might notice the name of our user in the terminal will change.

`sudo -u homeassistant -H -s`

In order install some of the Home Assistant dependencies, we will need the Rust compiler. 

> Note: even if you already installed the Rust compiler to upgrade Python, you need to install it again for the homeassistant user as the rustup install script works on a per-user basis.

`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

This will download and setup Rust. Normally, the installation script should detect and set the default install options automatically. 

For a Raspbery Pi 3 these are:

```
  default host triple: armv7-unknown-linux-gnueabihf
  default toolchain: stable (default)
  profile: default
  modify PATH variable: yes
```
  
The default host value will depend on the type of device you are using. The other values should be kept as above. 

If the values are correct, select option 1 to install Rust. If not, select option 2 and make the necessary changes. Then install Rust with these changes applied.

When the Rust compiler has been installed, reload the shell interface with

`source "$HOME/.cargo/env"`

Verify Rust is installed with 

`rustc -V`

This should display a fairly recent version of the Rust compiler.

Enter the directory and create the venv

`cd /srv/homeassistant`

`python -m venv .`

This might take a minute or so while the isolated Python environment is getting setup. When it’s done, activate the venv

`source bin/activate`

Next, we need an additional install tool, wheel, which we install through the pip Python package manager.

`python -m pip install wheel`

While we’re at it, lets upgrade pip, too.

`python -m pip install --upgrade pip`

After this, you’re ready to install Home Assistant!

`pip install homeassistant`

The pip package manager will download and install all necessary components to run Home Assistant, resulting in the installation of our favourite smart home software itself. Grab yourself another coffee, this might take a while.

When the installation is finished, start Home Assistant with

`hass`

The first time you run Home Assistant, it will create a default configuration which you can edit later. It also will take a while to setup and start all required integrations. You might receive a waiting for integration to finish warning.

After a few minutes you should be able to access Home Assistant at the IP address of your Pi, followed by :8123, e.g. `192.168.1.209:8123`, or at the configured mDNS hostname, e.g. `homeassistant.local:8123`. 

Congratulations! You can now start you journey into the smart home! Take some time to set up your Home Assistant account before moving on the next steps.

### Configuring Home Assistant as a service
In the previous step, we started Home Assistant manually. If you would close the PuTTY terminal, the Python application which is Home Assistant will quit. 

We of course want Home Assistant to be running all the time. To do this, close your current PuTTY sessions, open a new one and log back in to your Pi. 

We will add a new entry to *systemd*, the system and service manager of the Linux operating system, to start Home Assistant on boot and make sure it keeps running.

First, create an entry for our new service.

`sudo nano /etc/systemd/system/home-assistant@homeassistant.service`

In the nano text editor, paste this file (remember: to paste text in PuTTY, just right-click in the terminal).
```
[Unit]
Description=Home Assistant
After=network-online.target

[Service]
Type=simple
User=%i
ExecStart=/srv/homeassistant/bin/hass -c "/home/%i/.homeassistant"

[Install]
WantedBy=multi-user.target
```
Close nano and save its contents with Control + X, followed by Y and enter. 

Reload the changes to systemd.

`sudo systemctl daemon-reload`

Start our brand new homeassistant service with

`sudo systemctl start home-assistant@homeassistant.service`

Make sure it is enabled on the next boot

`sudo systemctl enable home-assistant@homeassistant.service`

Check the status of the service

`sudo systemctl status home-assistant@homeassistant`

You should now be able to access Home Assistant again on the :8123 port.

To be sure Home Assistant is running as a service now, reboot the Pi with 

`sudo reboot`

Wait a minute or two, and log back in to the Pi. Check the status of the service again

`sudo systemctl status home-assistant@homeassistant`

If all went well, you should be able to access Home Assistant consistently through reboots of your Pi now.

I recommend you now take some time to setup and get familiar with Home Assistant. This is not part of this tutorial, so check out the official [Home Assistant docs](https://www.home-assistant.io/getting-started/onboarding/).


### Updating Home Assistant
Every month Home Assistant gets a new release with new features and bugfixes. When running Home Assistant as a standalone application, you have to maintain your installations yourself. 

It’s certainly not required to update every month, and often you’re even better off ignoring major releases (ending in a .0) and waiting for a so-called point release containing bug fixes.

Updating Home Assistant is quite easy. Login in to your Pi. If it has been a while since your previous login, it might be a good time to update your other software as well.

`sudo apt-get update`

`sudo apt-get -y upgrade`

`sudo apt autoremove`

Now everything is up to date, stop the Home Assistant service.

`sudo systemctl stop home-assistant@homeassistant.service`

Change to the homeassistant service.

`sudo su -s /bin/bash homeassistant`

Enter the venv where Home Assistant lives in.

`source /srv/homeassistant/bin/activate`

Upgrade the Home Assistant installation

`pip3 install --upgrade homeassistant`

Exit the venv and switch back to your normal user.

`exit`

Start our Home Assistant service again.

`sudo systemctl start home-assistant@homeassistant.service`

After a minute or two, Home Assistant will be back with the updated version.

### Installing custom integrations
Coming soon.

## Installing mosquitto
If there is one basic tool which every Home Assistant installation needs, it’s the Mosquitto MQTT broker. 

MQTT (Message Queuing Telemetry Transport) is a lightweight, publish-subscribe network protocol that efficiently transports small messages between devices. It’s very popular within the home automation world because of its simplicity yet versatility. 

An MQTT broker like Mosquitto functions like a wall of mailboxes, where each mailbox is called a topic. An MQTT client, like an IoT temperature sensor, can publish a message to this mailbox/topic, e.g. containing the latest temperature reading.

Another client, e.g. Home Assistant, can then check this mailbox for the latest message, in this case the temperature reading and. Furthermore, clients can request additional mailboxes/topics on the go, and notify other clients of their creation. 

This all happens with minimal overhead, which makes MQTT a very interesting communication system.

Let’s install the Mosquitto broker. First, lets make sure we have all the latest updates.

`sudo apt update`

`sudo apt upgrade`

Next, install the Mosquitto broker. We will also install the Mosuitto client to test the functionality.

`sudo apt install mosquitto mosquitto-clients`

To configure Mosquitto we ill need to modify its configuration file.

First, delete the default configuration file

`sudo rm -f /etc/mosquitto/mosquitto.conf`

Next, create a new configuration file

`sudo nano /etc/mosquitto/mosquitto.conf`

Paste in the following lines
```
persistence true
persistence_location /var/lib/mosquitto/
log_dest file /var/log/mosquitto/mosquitto.log
include_dir /etc/mosquitto/conf.d
```

Save and exit.

Next, navigate to the Mosquitto config directory

`cd /etc/mosquitto/conf.d`

Create a user config file for the default user:

`sudo nano default.conf`

Paste in the following configuration
```
allow_anonymous true
listener 1883
```

Save the file and exit the text editor (Ctrl + X, then Y, and Enter).

By default, we are going to allow MQTT clients without authentication, which is common on local networks. If you want to add authentication, you might want to investigate the *mosquitto_passwd* tool.

After making changes to the configuration, restart the Mosquitto service to apply them:

`sudo systemctl restart mosquitto`

Make sure the mosquitto service runs after each reboot

`sudo systemctl enable mosquitto`

Check the status of the Mosquitto service to ensure it's running correctly:

`sudo systemctl status mosquito`

Now lets test if Mosquitto is functioning correctly. Start up an additional PuTTY session to your Pi (protip: left-click the title bar and select Duplicate Session), login with your credentials, and subscribe to the topic test:

`mosquitto_sub -v -t test`

This terminal will now be listening to any incoming messages on the test topic.

In your original terminal, publish a message to this topic with 

`mosquitto_pub -t test -m thisisatest`

If your mosquitto configuration is correct, you should see both the topic test and the message `thisisatest`.

Next, we need to add the MQTT integration to Home Assistant. 

In your Home Assistant dashboard, navigate to *Settings*, then *Devices & Services*. In the bottom right corner, click *Add Integration*. Next, search for MQTT. Select the basic MQTT integration. 

For *Broker*, fill in `localhost`. Make sure *Port* is set to `1883`. You can leave the other fields blank. 

Submit, and MQTT is now talking to Home Assistant.

Whenever MQTT enabled devices supporting Home Assistant are added to your local network, they will now show up in the MQTT integration.

## Making your Home Assistant remotely accessible
You can now access and control your Home Assistant from your local network. But the fun only begins when you can also control your smart home away from home. This requires your Home Assistant to be accessible from the internet. 

There are two ways to do this:

- Use the 7.50 EUR/month Nabu Casu subscription
- Roll your own remote access
- 
While I’m not opposed to the Nabu Casu subscription, the company funding a big part of the Home Assistant development, you might want to try rolling your own remote access first. This does require some advanced tinkering though!

To be able to remotely access Home Assistant (and other HTTP services on your Pi) it needs to be accessible from the internet. As a security measure, your internet router however blocks all incoming internet traffic if this traffic was not requested by a device on the internal, local network.

To be able to traverse your router from the internet and reach your Pi, we will need to open up some TCP ports to the internet and forward them to the IP address of the Pi. Then, when internet traffic arrives on these ports, the router will forward it to the Pi instead of blocking it. This is called **Port Forwarding**.

You will need to forward TCP ports **80** and **443** on your router to the IP address of your Pi.

Port forwarding requires you to login to the web interface of your router, which will differ from brand to brand or ISP. You will have to figure it out yourself here. 

> Small tip: Port Forwarding is sometimes also called *NAT Forwarding*, *Port Mapping* or *Virtual Server*. Good luck.

Once the port forwarding has been set up, we can now reach our Pi from the internet over ports 80 and 443. But wait, didn’t Home Assistant run on port 8123? That is correct. To add a layer of security and extra functionality, we will not be exposing Home Assistant directly to the internet. Instead, we will use a reverse proxy.

### Installing Nginx
In this guide, we will only expose the Home Assistant HTTP interface to the web. But what if you want to use other HTTP services remotely? E.g. you want to use nice Grafana dashboards (port 3000)? Host your own webpage (80)?  Access your NVR? Use the web interface of a Transmission bittorrent client (9091)? Sure, you could forward all these ports on your internet router, one for each HTTP service. But how safe would that be?

Enter **Nginx** (pronounced *Engine X*). Nginx is both a web server and a reverse proxy. It allows multiple internal HTTP services to be remotely accessed through the same hostname and TCP/IP port, while maintaining maximum security. E.g. it allows you to access unencrypted local HTTP services over a remote encrypted HTTPS connection.

Imagine all the internal HTTP services you want to access are houses on a street. Every service has a door (port) which opens from the street for all traffic wanting to enter that specific house. Doesn’t really sound safe, right?

What we will do is arrange all houses facing each other around a private square, with just one common access to the public street (port 442, with port 80 as a service access). 

On this one common access point, Nginx will be our doorman. Every time a gentleman caller enters, Nginx will ask them who they have an appointment with. If they state the name of the service correctly, Nginx will gently but firmly guide them to the right door to do their business. If the gentleman caller utters a wrong name, or no name at all? Better turn back right now, buddy.

Furthermore, Nginx allows us to secure communications between the internal HTTP service and the remote user (you), even if the internal service only supports unencrypted HTTP. We will encapsulate all unencrypted HTTP traffic in a secure HTTPS connection before it goes out over the internet. This means no third parties can eavesdrop on the traffic sent between you and the Pi.

Let’s install Nginx

`sudo apt-get install nginx`

Next, we’re going to beef up security by implementing Forward Secrecy. HTTP communication between your Pi and your remote user will be encrypted with a private key, stored on your Pi. The main advantage of forward secrecy is that even if an attacker manages to obtain this private key at some point in the future, they cannot use it to decrypt past sessions.

Go the Nginx directory.

`cd /etc/nginx`

Generate a 2048-bit key. You could also generate a 4096-bit key for additional security if you want, but 2048 bits is generally considered safe enough. Generating a 4096-bit key will take quite some time.

`sudo openssl dhparam -out dh2048.pem 2048`

When the key has been generated, create the forward secrecy file

`sudo nano perfect-forward-secrecy.conf`

Copy and paste this into the new file:
```
ssl_ciphers "ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS";
ssl_dhparam /etc/nginx/dh2048.pem;
add_header Strict-Transport-Security "max-age=31536000; includeSubdomains;";
```

Save and exit.

Now open your Nginx configuration file.

`sudo nano /etc/nginx/nginx.conf`

In the `http block`, in `Basic Settings` add 

`tcp_nodelay on;`

under the `tcp_nopush on;` line. This will improve performance for Home Assistant a little bit. Make sure the indentation is correct!

A bit further, uncomment the `server_tokens off;` and `server_names_hash_bucket_size 64;` lines. This will prevent Nginx to report its version number to visitors, which might then try to find exploits for this specific version.

In `SSL Settings`, in the line starting with `ssl_protocols`, delete `TLSv1 TLSv1.1` so the line becomes `ssl_protocols TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE`

By deleting TLSv1 and TLSv1.1, Nginx will no longer accept secure connections using these outdated encryption protocols, instead forcing clients to the much more secure TLSv1.2 or TLSv1.3.

In `Logging Settings`, add the line

`error_log /var/log/nginx/error.log;`

In `Virtual Host Configs`, add our forward secrecy file:

`include /etc/nginx/perfect-forward-secrecy.conf;`

We'll also harden Nginx against DDOS attacks a little bit by limiting the amount of time Nginx waits for client connections to be established. This decreases the opportunity window of a potential attacker.

Add these lines inside the `http block`, just before the end bracket }:
```
##
# Harden Nginx against DDOS
##

client_header_timeout 10;
client_body_timeout   10;
keepalive_timeout     10 10;
send_timeout          10;
```

Save and exit. Reload Nginx 

`sudo /etc/init.d/nginx reload`

Check if Nginx is running fine

`sudo /etc/init.d/nginx reload`

If you didn’t make any mistakes in the configuration file, it should display the active (running) status.

## Getting a dynamic DNS
To access your Pi remotely, you would need to connect to the external IP address of your router on the internet. This can be troublesome, as your ISP might randomly assign you a new IP address any time. It’s also not that easy to remember a string of numbers like 78.125.36.15, right?

How much easier would it be if your Pi had an easy to remember static domain name, always pointing to your Pi? 

Well, you can have this thanks to so-called dynamic DNS services. A DynDNS service generates a domain name for you and makes it point to the IP address of your router. That way, if you type in the domain name in your browser, you will end up on whatever website Nginx is hosting on your Pi! And with a little magic, we can also make our Pi update the DynDNS redirect should the IP address of your router change.  

In this guide, we will be using **DuckDNS**, a free dynamic DNS service hosted on AWS. It only requires you to have an account on Google, Github or Twitter in order to create up to 5 domain names (I wouldn’t recommend using your Twitter account these days). 

Every domain name is a subdomain of the duckdns.org domain.

Go over to [DuckDNS](https://www.duckdns.org/), login, and create your unique yet easy to remember domain name. Pick wisely, as this will be your personal public domain name! 

For this guide, I’m going to use `hasstutorial`, making the full domain name `hasstutorial.duckdns.org`.

When you add your domain name, you will see it automatically fills in the current IP address of your router. This IP address can change over time. We will make a script which periodically updates the IP the domain should point to. 

On your Pi, move to you home folder

`cd`

We will make a new directory for DuckDNS and create an update script in it.

`mkdir duckdns`

`cd duckdns`

`nano duck.sh`

Paste in the following lines. Make sure you use your domain name and token. You can find the token on the main DuckDNS page (after logging in).

`echo url="https://www.duckdns.org/update?domains=hasstutorial&token=a7c4d0ad-114e-40ef-ba1d-d217904a50f2&ip=" | curl -k -o ~/duckdns/duck.log -K -`

Save and exit. Next, we will make the script executable.

`chmod 700 duck.sh`

Test the script with

`./duck.sh`

This should return no output. 

Every time this script runs, it will update the IP address the DuckDNS domain should point to. 

To have it run automatically, we will add a *cron job*. Cron is a task scheduler available in most Linux distributions.

Open the list of cron jobs

`crontab -e`

Add a new cron job at the end of the file:

`*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1`

This will run our script every 5 minutes and suppress any error output.

## Setting up your first site
To test if we can reach our Pi from the internet, we’re going to make a small web page first.

Nginx keeps a list of available and enabled sites it can redirect incoming traffic to. Go to the location of the available sites 

`cd /etc/nginx/sites-available`

If you view the contents of this location (ls -la), you will see there is currently one site available, `default`. Lets add our website

`sudo nano hasstutorial.duckdns.org`

Paste the following configuration into the file, making sure you replace hasstutorial with your own domain everywhere!
```
server {
listen 80 default_server;
listen [::]:80 default_server;
    server_name hasstutorial.duckdns.org www.hasstutorial.duckdns.org;

    root /var/www/hasstutorial.duckdns.org/www;
    index index.php index.html index.htm;

    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /var/www/hasstutorial.duckdns.org/www;
    }

    # Error & Access logs
    error_log /var/www/hasstutorial.duckdns.org/error.log error;
    access_log /var/www/hasstutorial.duckdns.org/access.log;

    location / {
        index index.html index.php;
    }

    location ~ /.well-known {
                allow all;
    }

}
```

Save and close the file.

Our site is now available to Nginx but it is not enabled yet, so Nginx will not yet serve it. To have it enabled, it should also be in the `/etc/nginx/sites-enabled location`. 
To do this, we will create a *symbolic link* inside that location pointing to our file.

`sudo ln -s /etc/nginx/sites-available/hasstutorial.duckdns.org`

`etc/nginx/sites-enabled/hasstutorial.duckdns.org`

If you ever want to disable this site, just delete the symbolic link with

`sudo rm /etc/nginx/sites-enabled/hasstutorial.duckdns.org`

In fact, we are now going to delete the default sites from both locations.

`sudo rm /etc/nginx/sites-enabled/default`

`sudo rm /etc/nginx/sites-available/default`

Now we need something to actually show to the user visiting our website. Lets go to

`cd /var/www`

This location should already contain one folder, html. Let’s copy and rename it:

`sudo cp -r html hasstutorial.duckdns.org`

Enter our new folder

`cd hasstutorial.duckdns.org`

Create a new subfolder and enter it

`sudo mkdir -p www`

`cd www`

Here, we will create our very own site.

`sudo nano index.html`

Paste the following lines into the file
```
<html>
<body>
<h1>My very first site</h1>
</body>
</html>
```

Save and exit.

We have created our site as the sudo user. We need to assign ownership to the www user in order for it to be served through Nginx.

`sudo chown -R www-data:www-data /var/www/hasstutorial.duckdns.org`

Also make sure the file is readable

`sudo chmod -R 755 /var/www/hasstutorial.duckdns.org`

To make sure Nginx serves our site, let’s reload its configuration.

`sudo /etc/init.d/Nginx reload`

Now fire up a browser, surf to your domain name (e.g. `http://hasstutorial.duckdns.org/`), making sure your browser does **not** put https in front of it, et voila. 

Congratulations, you just created your very first site, self-hosted on your own Pi and accessible from the internet over a domain name!

### Securing your site with a SSL/TLS certificate
Currently, you are serving your site to the whole wide world over a plaintext, unsecured HTTP connection, enabling a potential attacker to eavesdrop on the information sent to and from the website. That’s not very secure. Let’s switch that over to an encrypted HTTPS connection.

HTTPS does two things. 

First, it encrypts the communication between the remote client and Nginx, making sure nobody can read its contents. 

Secondly, in order for the secure connection to be established, it verifies whether the server you are connecting to is effectively the one hosting your website on your domain, your Pi, and not some malicious imposter who has hijacked your domain name and makes it point to their server. 

To verify if the server is the real deal, it must present the connecting client a certificate from a *Certificate Authority* (CA). The CA is a trusted entity which verifies that a server is who it is claiming to be, and then issues digital certificates as proof of this check. A connecting client examines this certificate and only establishes the secure connection when it checks out. If the certificate is not valid, no connection will be made and the client will get a big bad warning in their browser not to trust this server. 

Procuring such a certificate used to be a cumbersome and often expensive process. A CA had to manually check if the server was who he claimed to be, especially if you wanted so-called Extended validation to also prove you were the owner of the website. I remember having to fax a signed letterhead of my company to the CA to get our website validated, in 2012!

Gone are those days, and thanks to initiatives like Lets Encrypt procuring TLS certificate is now automated and free. We will be using Lets Encrypt’s handy certbot to get a TLS certificate and automatically renew it when its expiration date nears.

Unfortunately, Let’s Encrypt recently (2023) has chosen to distribute certbot by default as a *snap*. Snaps are containerized software packages that include dependencies needed for the application to run, thereby simplifying software installation and updates, and providing a more secure and isolated environment as each snap is sandboxed and isolated from the host system and other snaps. The downside is more resource use and less control over the application, as it manages updates by itself. 

The open source community is somewhat split over snaps, as old school users (like myself) tend to favor the latter arguments more, while software development companies give higher priority to the snaps ability to deliver and update their applications.

Anyway, we’ll stow away our principles for once and go with the default installation method recommended by Let’s Encrypt. First, we need snapd, the background service that manages and maintains snaps.

`sudo apt install snapd`

After the installation has completed, reboot your Pi.

`sudo reboot`

Wait a minute or two before restarting the PuTTY session, then log back in. Install the core snap which will manage snapd.

`sudo snap install core`

Now we can finally install certbot

`sudo snap install --classic certbot`

Execute the following instruction on the command line on the machine to ensure that the certbot command can be run

`sudo ln -s /snap/bin/certbot /usr/bin/certbot`

Then run certbot to get our certificate and install it in Nginx in one go.

`sudo certbot --nginx`

Follow the instructions on screen. certbot will procure and install the certificates for your requested sites.

Certficates are only valid for a limited time before they have to be renewed. Certbot will automatically keep track of this. We recommend to simulate the renewal process now to check if it works. 

`sudo certbot renew --dry-run`

If you now refresh your website in your browser, you will see a https and/or a lock icon appear, meaning the connection to your site is now fully encrypted!

### Redirecting your domain to Home Assistant
Now your domain name points to your basic website. We want it now to point to Home Assistant instead. 

Go back the available sites location

`cd /etc/nginx/sites-available`

Open your site config.

`sudo nano hasstutorial.duckdns.org`

Notice how certbot has moved things around while installing the certificates?

We now need to point Nginx to Home Assistant in stead of our basic website whenever someone visits the domain.

Under the access_log line, just above `location /` add
`proxy_buffering off;`

In the `location /` block, delete index index.html index.php; Instead, add these lines:
```
       proxy_pass http://localhost:8123/;
       proxy_set_header Host $host:$server_port;
       proxy_redirect http:// https://;
       proxy_http_version 1.1;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
```

Respect the indentation, and make sure the closing curly brace } remains!

A bit further, delete this whole block:
```
    location ~ /.well-known {
                allow all;
    }
```

Save and exit.

Reload Nginx with 

`sudo /etc/init.d/nginx reload`

 
## Upgrading Python
Upgrading your default system Python is not for the faint of heart. It might break existing applications depending on Python remaining the version which shipped with the OS, and so it’s generally not recommended to upgrade it anyway. Instead, we will be installing the latest Python version next to the default one. Don’t worry, this is perfectly normal and will not have any impact on performance. These so-called altinstalls are just part of the Python quirkiness.
To install a new Python, we first need the Rust compiler.
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
This will download and setup the Rust installer. Make sure the displayed installation options are as follows:
   default host triple: armv7-unknown-linux-gnueabihf
     default toolchain: stable (default)
               profile: default
  modify PATH variable: yes
If so, select option 1 to install Rust. If not, select option 2 and make these changes. You can leave other options to default.
When the Rust compiler has been installed, reload the shell interface with
source "$HOME/.cargo/env"
Verify Rust is installed with 
rustc -V
This should display a fairly recent version of the Rust compiler.
We’re going to automate the rest of the installation with a shell script. Create a file with the Nano text editor
nano updatepython.sh
Copy/paste the following script into this file (tip: to paste in a PuTTY shell, just right click. Also, selecting text by highlighting it copies it to the clipboard).
#!/bin/bash

if [ true ]; then
  # get a current python3
  
  sudo apt-get install wget build-essential libreadline-gplv2-dev libncursesw5-dev libssl-dev libsqlite3-dev tk-dev libgdbm-dev libc6-dev libbz2-dev libffi-dev zlib1g-dev liblzma-dev -y
  VERSION=3.12.1
  VERSION_SHORT=3.12
  
  mkdir -p tmp
  cd tmp
  
  if [ ! -f Python-${VERSION}.tgz ]; then
    wget -O Python-${VERSION}.tgz https://www.python.org/ftp/python/${VERSION}/Python-${VERSION}.tgz
  fi
  
  tar xzf Python-${VERSION}.tgz
  cd Python-${VERSION}
  if [ ! -f python ]; then
    ./configure --enable-optimizations
  fi
  echo "### make altinstall"
  sudo make altinstall
  sudo update-alternatives --install /usr/bin/python python /usr/local/bin/python${VERSION_SHORT} 1
  
  echo "### install pip"
  /usr/local/bin/python${VERSION_SHORT} -m pip install --upgrade pip
  sudo update-alternatives --install /usr/bin/pip pip /usr/local/bin/pip${VERSION_SHORT} 1
  
  cd ..
fi

Close Nano with the key combination of Control + X.
Make the script executable.
chmod +x updatepython.sh
Now run the installation script with sudo ./ updatepython.sh
While the Pi is compiling Python, you can go get another cup of coffee. Perhaps even 2. Depending on the version (and thus CPU power) of your Pi, this is going to take a while.
When the compilation and installation process has finished, you will return to the command line. You can verify if Python3.12 has been installed with
python3.12 -V
By using the command python3.12 in stead of python you redirect Python applications to the newer 3.12 Python interpreter, while python still refers to the default Python interpreter which came with the system, thereby ensuring that system applications keep working.
 
sudo apt-get install libjpeg-dev zlib1g cmake libopenblas-dev

Next, we need an additional install tool, wheels, which we install through the pip Python package manager.
python3.12 -m pip install wheel
python3.12 -m pip install --upgrade pip
We are now ready to install Home Assistant. Instead of installing it under our own user account, we are going to create a new user exclusively for Home Assistant. This way, Home Assistant lives in a different location, safe from you accidentally deleting your home folder or creating other unwarranted mayhem which might impact its performance.
Add the new user
sudo useradd -rm homeassistant -G dialout,gpio,i2c
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
Verify Rust is installed with 
rustc -V
We will install the Home Assistant Core Python application in a virtual environment, or venv. A venv is kind of like an isolated copy of an existing Python installation, where you can mess around with installing additional libraries or downgrading packages without botching the rest of the system. If you really end up turning the venv into a hot mess, you can delete it and start over fresh. It’s not a virtual machine, but what Python’s concerned, it’s pretty close.
Go to the directory where the venv will live and create it
cd /srv
sudo mkdir homeassistant
Change the ownership of the directory to our homeassistant user
sudo chown homeassistant:homeassistant homeassistant
Switch our user account to the homeassistant user. You might notice the name of our user in the terminal will change.
sudo -u homeassistant -H -s

Enter the directory and create the venv
cd /srv/homeassistant
python3.12 -m venv .
This might take a minute or so while the isolated Python environment is getting setup. When it’s done, activate the venv
source bin/activate
After this, you’re ready to install Home Assistant!
pip install homeassistant
The pip package manager will download and install all necessary components to run Home Assistant, ending with the installation of our favourite smart home software itself. When it is finished, start Home Assistant with
hass
The first time you run Home Assistant, it might take a while to setup and start all required integrations. You might receive a waiting for integration to finish warning, but after a few minutes you should be able to access Home Assistant at the IP address of your pi, followed by :8123, e.g. 192.168.1.209:8123 or at the configured mDNS hostname, e.g. homeassistant.local:8123 
Congratulations! You can now start you journey into the smart home!


 
Securing you Pi
Later in this guide, we’re going to expose your Pi to the big bad internet. This allows you to use your Home Assistant remotely, but also exposes your Pi to 
First, we’re going to set up a firewall to limit and block unwanted inbound traffic to your Pi.
Check your Pi's default firewall rules by entering the following command:
sudo iptables -L

This should give you the following, empty ruleset:
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

We’re going to create a file to hold your 
sudo nano /etc/iptables.firewall.rules

*filter

#  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
-A INPUT -i lo -j ACCEPT
-A INPUT -d 127.0.0.0/8 -j REJECT

#  Accept all established inbound connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

#  Allow all outbound traffic - you can modify this to only allow certain traffic
-A OUTPUT -j ACCEPT

#  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

#  Allow access to Home Assistant 
-A INPUT -p tcp --dport 8123 -j ACCEPT

# Allow MQTT
-A INPUT -p tcp --dport 1883 -j ACCEPT

#  Allow SSH connections
#  The -dport number should be the same port number you set in sshd_config
-A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT

#  Allow ping
-A INPUT -p icmp --icmp-type echo-request -j ACCEPT

#  Log iptables denied calls
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

#  Drop all other inbound - default deny unless explicitly allowed policy
-A INPUT -j DROP
-A FORWARD -j DROP

COMMIT



## Installation

This work is licensed under a
[Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
