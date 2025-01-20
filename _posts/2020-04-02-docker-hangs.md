---
title: Docker hangs during build
date: 2020-04-02
categories: [Devops]
permalink: /2020/04/docker-hangs-during-build/
---
Yesterday I had a strange problem with Docker during a build process. I use Linux Mint. I didn’t have enough space at the main system directory /. By default, Docker saves all data (images, volumes, etc.) in /var/lib/docker. Firstly I wanted to move this directory to the /home, but unfortunately, there was probably a problem with encryption, and Docker’s daemon couldn’t start properly. The second option was to move the directory to a hdd drive. But it also wasn’t a piece of cake (just Linux user normal day 😀 ).  

## The problem
When Docker’s data were in the default location, the project has been built successfully. Then I changed the location of Docker’s data to another drive. It caused a problem and Docker froze during the build.

## How to change the default location of Docker’s data?
Stop the Docker’s daemon

```
sudo service docker stop
```

Add an option with the path to the directory to the Docker’s daemon configuration file (/var/docker/daemon.json) (Just add this file if not exists)

```
{ 
   "data-root": "/mnt/drive"
}
```

Copy the current data to the new directory

```
sudo rsync -aP /var/lib/docker/ /media/sarven/drive
```

Remove the old data

```
sudo rm -rf /var/lib/docker
```

Restart the Docker’s daemon

```
sudo service docker start
```

## How to fix a hanging?
The problem was with the NTFS filesystem on the drive that I wanted to use as default Docker’s data directory. I discovered that Docker had set aufs storage driver, but I expected overlay or overlay2. I stopped Docker, formated the drive and then started Docker. Then Docker had set storage driver as overlay2 and the build was successful. I decided to write about this problem, maybe someone will find this as helpful.
