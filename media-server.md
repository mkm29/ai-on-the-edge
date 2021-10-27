# Homelab Media Stack

```yaml
Author: Mitch Murphy
Date: 2021-10-26
```

## Table of Contents

1. [Helm Repo](#helm-repo)  
    1. [Configure](#pre-configure-transmission)  
2. [Jackett](#jackett)  
    1. [Configure](#pre-configure-jackett)  
    2. [Install](#install-jackett)  
3. [Sonarr](#sonarr-overview)  
    1. [Configure](#pre-configure-sonarr)  
    2. [Install](#install-sonarr)  
4. [Radarr](#radarr-overview)  
    1. [Configure](#pre-configure-radarr)  
    2. [Install](#install-radarr)  

## Helm Repo

Someone already made a very good job at creating specific Helm Charts for the all the software we wish to install in this tutorial. Add the following repository to your Helm using the following command:  

```shell
helm repo add bananaspliff https://bananaspliff.github.io/geek-charts
helm repo update
```

## Transmission

First lets install the popular, open-source BitTorrent client Transmission. First we need to create a secret containing out VPN credentials.  

```shell
kubectl create secret generic openvpn \
    --from-literal='username=mitch.murphy@gmail.com' \
    --from-literal='password=Mitch%*%363502' \
    --namespace media
```

### Pre-configure Transmission

Adjust the [values](media.transmission-openvpn.values.yml) file, and then install the chart:  

```shell
helm install transmission bananaspliff/transmission-openvpn \
    --values media.transmission-openvpn.values.yml \
    --namespace media
```

After a few minutes the Transmission pod should be running, and will be accessible at `<HOST>/transmission` (in my case `media.smigula.io/transmission`). The Ingress resource specifies that it should redirect from `http` to `https` and procure a certificate using Lets Encrypt (and store in a secret in the media namespace).

## Jackett

Jackett is a Torrent Providers Aggregator which translates search queries from applications like Sonarr or Radarr into tracker-site-specific http queries, parses the html response, then sends results back to the requesting software. Because some Internet Providers might also block access to Torrent websites, I packaged a version of Jackett using a VPN connection (similar to transmission-over-vpn) accessible on [Docker hub](https://hub.docker.com/repository/docker/smigula/jackett-ovpn).  

Let’s now configure the chart bananaspliff/jackett. The default configuration can be seen by running the following command `helm show values bananaspliff/jackett`. I have made some slight modifications to the [values](media.jackett.values.yml).  

### Pre-configure Jackett

For this deploy we will not use OpenVPN (you can as it is installed on the image, will just require a few more modifications to the host), but you will need to create the following directory:  

```shell
sudo mkdir -p /mnt/storage/media/configs/jackett/Jackett/
sudo chown -R 1001:0 /mnt/storage/media/configs/jackett/Jackett/
chmod +w /mnt/storage/media/configs/jackett/Jackett/
```

Now you need to create the file ServerConfig.json into the folder `/mnt/storage/media/configs/jackett/Jackett/` with the following content:

```json
{
  "BasePathOverride": "/jackett"
}
```

### Install Jackett

We are now ready to install the chart:  

```shell
helm install jackett bananaspliff/jackett \
    --values media.jackett.values.yml \
    --namespace media
```

## Sonarr - Overview

Sonarr is a TV Show library management tool that offers multiple features:

* List all your episodes and see what’s missing  
* See upcoming episodes  
* Automatically search last released episodes (via Jackett) and launch download (via Transmission)  
* Move downloaded files into the right directory  
* Notify when a new episodes is ready (Kodi, Plex)  

I have provided a customizing Helm values file for this chart, which can be found [here](media.sonarr.values.yml).  

### Pre-configure Sonarr  

Create the following directory structure on your host node:  

```shell
mkdir -p /mnt/ssd/media/configs/sonarr/
sudo chown -R 1001:0 /mnt/ssd/media/configs/sonarr/
chmod +w /mnt/ssd/media/configs/sonarr/
```

Create the file config.xml into the folder `/mnt/storage/media/configs/sonarr/` with the following content:  

```xml
<Config>
  <UrlBase>/sonarr</UrlBase>
</Config>
```

### Install Sonarr  

We are now ready to install the chart:  

```shell
helm install sonarr bananaspliff/sonarr \
    --values media.sonarr.values.yml \
    --namespace media
```

## Radarr - Overview

Radarr is a Movie library management tool that offers multiple features:  

* List all your movies  
* Search movies (via Jackett) and launch download (via Transmission)  
* Move downloaded files into the right directory  
* Notify when a new movie is ready (Kodi, Plex)  

I have provided a customizing Helm values file for this chart, which can be found [here](media.radarr.values.yml).  

### Pre-configure Radarr  

Create the following directory structure on your host node:  

```shell
sudo mkdir -p /mnt/storage/media/library/movies
sudo mkdir -p /mnt/storage/media/configs/radarr/
sudo chown -R 1001:0 /mnt/storage/media/library/movies
sudo chown -R 1001:0 /mnt/storage/media/configs/radarr/
chmod +w /mnt/storage/media/library/movies -R
chmod +w /mnt/storage/media/configs/radarr/ -R
```

Now create the file `config.xml` into the folder `/mnt/storage/media/configs/radarr/` with the following content:  

```xml
<Config>
  <UrlBase>/radarr</UrlBase>
</Config>
```

### Install Radarr

```shell
helm install radarr bananaspliff/radarr \
    --values media.radarr.values.yml \
    --namespace media
```
