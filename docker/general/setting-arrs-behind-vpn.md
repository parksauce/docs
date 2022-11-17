---
title: Setting the *arrs behind a VPN
description: This guide will cover setting up Sonarr, Radarr, and various others behind a VPN
published: false
date: 2022-11-17T00:45:28.519Z
tags: docker, general
editor: markdown
dateCreated: 2022-06-05T00:42:25.363Z
---

# VPNs and the *arrs

Setting Sonarr, Radarr and the like behind a VPN

# Overview
This guide will cover setting up services like sonarr, radarr, and prowlarr, and whatever torrent client you prefer behind a VPN. The benefits of doing so is hiding your internet activity from your ISP. While I will set up Sonarr and Radarr behind a VPN in this guide the developers recommend to not do so as it may break some functionalities; [read more here](https://wiki.servarr.com/en/sonarr/faq#vpns-jackett-and-the-arrs). 

# The Setup
This guide will assume a few things about your environment however I provided an environment file to make it easier to adapt to different configurations.

## Prerequisites
There's tons of guides on this and its out of the scope of this guide so if you don't already have these packages installed please go do so now. 
- Docker
- Docker-Compose


## The Folder Structure
This is the most important part of this whole thing if you want atomic moving to work. That means that once sonarr or radarr sees that the file has been downloaded it will move the file into your media directory and rename it while still leaving a link of it for the torrent client to use. Otherwise sonarr and radarr will just copy the files and you will need to remove them manually later.

The basic idea of this is the folder structure needs to remain generally the same throughout all of the containers and needs to be located on the same drive on the host. A simple directory tree may be. We will continue to use this as our example throughout the guide be sure to substitute this with however your structure is setup. 
```
/mnt/data
|-- media
|   |-- books
|   |-- movies
|   |-- music
|   `-- tv
`-- torrents
```

## Preparing Your System
So your first step is to create the `docker-compose.yml` file if you're following my example you can use the below command to create the compose file in a `media_stack` folder in the `/opt` directory

```bash
sudo mkdir /opt/media_stack
```
Now lets change the permission of the media_stack file so our user can edit it. This will set the permission so anyone can read the folder but only the user and group can edit and execute in the directory. Replace `<USER>` with your username.
```bash
sudo chown <USER>:<USER> /opt/media_stack
sudo chmod 774 /opt/media_stack
```
Now lets create the `docker-compose.yml` file
```bash
touch /opt/media_stack/docker-compose.yml
```
Next lets make the environment variable file
```bash
touch /opt/media_stack/.env
```
Now we need to edit the permissions so not just anyone can see the `.env` file since it contains secrets. To do so run the command below this will allow only the user and group to read and write to this file.
```bash
sudo chmod 660 /opt/media_stack/.env
```

# The Compose File
Below is an example compose file for setting up Sonarr, Radarr, Prowlarr, and Deluge behind another container called Gluetun that hosts our VPN. Don't worry too much about how intimidating the stack looks as we begin to break it down you'll see that its not as complicated as it looks at first.


Now you just have to copy this file make any necessary edits and paste it in the file. More on the edits needed in the [break down section](#breaking-it-down).
```yaml
version: "3"
services:

  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    ports:
      - 8888:8888/tcp # HTTP proxy
      - 8388:8388/tcp # Shadowsocks
      - 8388:8388/udp # Shadowsocks
      - 8000:8000/tcp # Built-in HTTP control server
      - ${DELUGE_PORT}:8112 # Deluge
      - ${PROWLARR_PORT}:9696 # Prowlarr
      - ${SONARR_PORT}:8989 # Sonarr
      - ${RADARR_PORT}:7878 # Radarr
      - ${BAZARR_PORT}:6767 # Bazarr
    environment:
      - OPENVPN_USER=${GLUETUN_OPENVPN_USER}
      - OPENVPN_PASSWORD=${GLUETUN_OPENVPN_PASSWORD}
      - VPNSP=${GLUETUN_VPNSP}
    volumes:
      - ${CONFIG_FOLDER}/gluetun:/gluetun
    restart: unless-stopped

  deluge:
    image: linuxserver/deluge:latest
    container_name: deluge
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - DELUGE_LOGLEVEL=error
    volumes:
      - ${CONFIG_FOLDER}/deluge:/config
      - ${MEDIA_FOLDER}/${DOWNLOADS_FOLDER}:/data/torrents
    network_mode: service:gluetun
    restart: unless-stopped

  prowlarr:
    image: linuxserver/prowlarr:nightly
    container_name: prowlarr
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_FOLDER}/prowlarr:/config
    network_mode: service:gluetun
    restart: unless-stopped

  sonarr:
    image: linuxserver/sonarr:latest
    container_name: sonarr
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_FOLDER}/sonarr:/config
      - ${MEDIA_FOLDER}:/data
    network_mode: service:gluetun
    restart: unless-stopped

  radarr:
    image: linuxserver/radarr:latest
    container_name: radarr
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_FOLDER}/radarr:/config
      - ${MEDIA_FOLDER}:/data
    network_mode: service:gluetun
    restart: unless-stopped

  bazarr:
    image: linuxserver/bazarr:latest
    container_name: bazarr
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_FOLDER}/bazarr:/config
      - ${MEDIA_FOLDER}:/data
    network_mode: service:gluetun
    restart: unless-stopped
```
This is an environment file we use these to specify certain variables we need to run the containers. The benefits of using them is the ease of editing the stack in the future. I love using them because it makes it super easy to change environments and make any changes needed in a much faster manner. You may need to add parameters for gluetun depending on your VPN client. 

The downloads folder is automatically placed into the media folder if you don't want this behavior you will have to edit the deluge container to remove this. You don't need to use a `/` when specifying the `DOWNLOADS_FOLDER`.
``` yaml
# Global Parameters
PUID=1000
PGID=1000
TZ=America/New_York
CONFIG_FOLDER=/opt/media_stack
MEDIA_FOLDER=/mnt/data
DOWNLOADS_FOLDER=torrents

# Ports
DELUGE_PORT=8112
PROWLARR_PORT=9696
SONARR_PORT=8989
RADARR_PORT=7878
BAZARR_PORT=6767

# Gluetun Parameters

GLUETUN_VPNSP=nordvpn
GLUETUN_OPENVPN_USER=
GLUETUN_OPENVPN_PASSWORD=
```

## Breaking it down
In this section we will breakdown each container and cover what they do.

### Gluetun
This is perhaps the most important container in the entire stack as this will be the VPN for your containers. This section will require special configuration for each use case. It supports a variety of different VPN providers but some may require different environment variables. Be sure to check their [documentaion](https://github.com/qdm12/gluetun/wiki) to ensure its setup correctly. The example used below is for Nord VPN.

Most of the information in this section of the file we can just gloss over. Such as the section below this just specifies the image conatainer name and gives it the permissions it needs i.e. `NET_ADMIN`

```yaml
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
```
The below section specifies all of the ports we use for our services. You'll see some are set as port numbers and others are set using envrionment variables. Those variables are pulled from the envrionment file we specified above. You may need to change them if they conflict with anything you already have running but for most use cases you shouldn't need to change them. We need to specify all of our services through here since gluetun will be acting as our gateway to the internet. With that said if you would like to use docker DNS to load any of these services with their names you won't be able to do it with a simple `sonarr` for Sonarr instead you would use `gluetun:8989`. This is because all of our traffic first goes through the gluetun container before reaching any of your devices and thus it gets seen as gluetun traffic. 

```yaml
    ports:
      - 8888:8888/tcp # HTTP proxy
      - 8388:8388/tcp # Shadowsocks
      - 8388:8388/udp # Shadowsocks
      - 8000:8000/tcp # Built-in HTTP control server
      - ${DELUGE_PORT}:8112 # Deluge
      - ${PROWLARR_PORT}:9696 # Prowlarr
      - ${SONARR_PORT}:8989 # Sonarr
      - ${RADARR_PORT}:7878 # Radarr
      - ${BAZARR_PORT}:6767 # Bazarr
```

This is the meat and potatoes of this container. These are the variables you need to change to be able to connect to your VPN client. These variables are specified through the `.env` file. You may need to edit this section of the compose file to fit your needs. But if you're using NordVPN as your client then go get the credentails you need by logging into your [Nord Account](https://nordaccount.com/login). I'm not gonna delve much more into obtaining those credentials because its out of scope for this guide. Once you locate them just copy those over and your all set with this part.

```yaml
    environment:
      - OPENVPN_USER=${GLUETUN_OPENVPN_USER}
      - OPENVPN_PASSWORD=${GLUETUN_OPENVPN_PASSWORD}
      - VPNSP=${GLUETUN_VPNSP}
```

This last section contains the volume for the container. This is where the configuration files for gluetun are stored. Then under that we specify to have the container restart itself unless you manually stop the container.
```yaml
    volumes:
      - ${CONFIG_FOLDER}/gluetun:/gluetun
    restart: unless-stopped
```
### Deluge
The next container in the stack is deluge. This handles the downloading of torrents. Optionally you can replace deluge with any other torrent client your prefer.

This part of the file specifies the image user and the name for the container
```yaml
    image: linuxserver/deluge:latest
    container_name: deluge
    depends_on:
      - gluetun
```
Next we specify the environment variables for the container. These variables are pulled from the `.env` file. The PUID and GUID are used to set permissions. These should be set to your user which is typically `1000` for both but run `id` in your terminal to verify those values. Note if your using a user other than the default one created on most linux systems then these will be different. Lastly we set the deluge log level to error for debugging you may want to change that.  
```yaml
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - DELUGE_LOGLEVEL=error
```
Now we specify where the config files will be located deluge this will be set by `.env`. Then we specify the location of our downloads folder. While deluge only needs access to the downloads directory it still needs to maintain the same general file structure so if you have the same file structure as my example you would want `/data/torrents` inside the container while the host mounts `/mnt/data/torrents`
```yaml
    volumes:
      - ${CONFIG_FOLDER}/deluge:/config
      - ${MEDIA_FOLDER}/${DOWNLOADS_FOLDER}:/data/${DOWNLOADS_FOLDER}
```
The network mode is an important part of tunneling the traffic through the gluetun container. This specifies that deluge will use the service gluetun as its network adapter. If you were to run this using the docker CLI you would specify container in place of service.
```yaml
    network_mode: service:gluetun
    restart: unless-stopped
```

### Prowlarr
Prowlarr is what we will use to get the torrent files from different trackers. Alternatively you can use Jackett if you prefer. I like prowlarr because it has some extra functionality that makes it seamless to add new trackers to all of your services instead of configuring them individually. If you decide not to use a VPN with most of your stack I highly suggest you use one for this container especially if your ISP is known to snoop. There's nothing new or important with this stack so we will skip over step by step explaination. 
```yaml
  prowlarr:
    image: linuxserver/prowlarr:nightly
    container_name: prowlarr
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_FOLDER}/prowlarr:/config
    network_mode: service:gluetun
    restart: unless-stopped
```
### Sonarr
Sonarr is in charge of obtaining all of your TV media. Most of this stack looks the same as everything else. The reason we don't limit this down more to only the TV folder is that it messes with the atomic moving of the files if it can't see the downloads directory. 
```yaml
  sonarr:
    image: linuxserver/sonarr:latest
    container_name: sonarr
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_FOLDER}/sonarr:/config
      - ${MEDIA_FOLDER}:/data
    network_mode: service:gluetun
    restart: unless-stopped
```
### Radarr
Radarr is in charge of downloading movies for your media stack. There's nothing new to introduce here so were going to move on. 
```yaml
  radarr:
    image: linuxserver/radarr:latest
    container_name: radarr
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_FOLDER}/radarr:/config
      - ${MEDIA_FOLDER}:/data
    network_mode: service:gluetun
    restart: unless-stopped
```
### Bazarr
Lastly Bazarr is in charge of getting subtitles for your content. This container really isn't necessary unless you want to get subtitles for everything. 
```yaml
  bazarr:
    image: linuxserver/bazarr:latest
    container_name: bazarr
    depends_on:
      - gluetun
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${CONFIG_FOLDER}/bazarr:/config
      - ${MEDIA_FOLDER}:/data
    network_mode: service:gluetun
    restart: unless-stopped
```

# Running the Stack
Running the stack should be simple just pull the container with
```
docker-compose pull
```
Then start the stack with
```
docker-compose up -d
```

# Configuration
The final step is to configure all of the services to communicate with each other. Its a fairly simple process but takes a bit of time. 

Now that our containers are running we should be able to access them using the server ip and the port number. You'll need to open up a web brower and type in `http://<ServerIP>:<PORT#>`

## Deluge
First lets go to deluge and get it setup and prepared for sonarr and radarr. To access deluge you'll have to first find your servers IP address. You can find it with `ip a`, once you have that you'll want to go to your browser and type in `http://<ServerIP>:8112`. The default password is `deluge`

After you log in the first place we need to go is `Preferences > Downloads`. We need to change the default download location to the one we specified in the `.env` file if you followed me its `/data/torrents`

<center><img src=/assets/arrs_behind_vpn/deluge_download_folder.png></center>

Next lets go to the Plugins Section, sonarr and radarr use labels to help keep track of files that pertain to them. Because of this we will need to install the `Label` plugin to be able to use sonarr and radarr with deluge. Just check the box next to `Label` and press `Apply`.

<center><img src=/assets/arrs_behind_vpn/deluge_-_add_label_plugin.png></center>

Thats all the configuration we need to do in deluge so lets move on to the next step

## Prowlarr
To access prowlarr you use the same IP address as before but with port 9696. Your url will look like this `http://<ServerIP>:9696`. 

Make sure you're on the Indexers Tab on the left side of the screen. Now go through and add your favorite trackers, I've had luck with 1337x, Rarbg, and YTS to name a few.

Keep this window handy as you'll need it for the next steps. 

## Sonarr
Now lets access sonarr in our browser I think your starting to understand how to access the services so this time I'll just give you the port 8989.

After you open sonarr navigate to `Settings > Media Management` then click `Add Root Folder`. Now you will want to select your tv folder mine is `/data/media/tv` and click `Ok`

Now click `Download Clients` on the left panel, then click the big plus to add a download client. Select whatever client you chose if youre following my example its deluge. Give it a friendly name and make sure the `Host` is set to localhost and `Port` is set to 8112, then enter your password. I also like to shorten the category from tv-sonarr to sonarr, then click the `Test` button at the bottom of the window. If it succeeds click the `Save` button otherwise double check the password.

<center><img src=/assets/arrs_behind_vpn/sonarr_-_download_client.png></center>

We need to add indexers to sonarr to be able to start grabbing content to do so we first need to get the API key navigate to `Settings > General` under `Security` you should see an `API Key` section copy that and go back to prowlarr.

From prowlarr navigate to `Settings > Apps` then click the plus sign to add an app. Next click sonarr, for prowlarr server enter `http://localhost:9696` and for sonarr server enter `http://localhost:8989` and paste in the API key in the `ApiKey` field. Then click `Test` if it succeeds click `Save` otherwise double check your values.

<center><img src=/assets/arrs_behind_vpn/sonarr_-_indexer.png></center>

Now if you go back to sonarr you shouldn't see any warnings. Go to `Series > Add New` and try adding a new show. If it works great, now see if you can go configure radarr or follow along below.

If youre having issues setting a root folder make sure your user has permissions to write in that directory. 

## Radarr
Okay I can't keep typing so here's the port 7878

Alright first lets add a root folder, go to `Settings > Media Management` and click the big blue button. Select your folder in my case its `/data/media/movies` and click `Ok`.

Now lets add a download client look on the left pane and select `Download Clients`, leave `Host` as `localhost` and leave the `Port` as `8112` and enter your password then test and save it.

<center><img src=/assets/arrs_behind_vpn/radarr_-_download_client.png></center>

Alright lets go copy the API key now use the left pane to go to `General` and copy the `API Key`, now lets pop back over to prowlarr. 

From the Apps menu in settings add an app. Select Radarr and 





Oh by the way, I'm sorry if this is riddled with tons of typos and gramatical issues its 2am. 
