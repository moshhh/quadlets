# Quadlets
## Disclaimer
I'm writing this documentation as I'm learning so there might be some mistakes. What is written are the steps I've taken to get my setup working. I'm posting here as I found it difficult to find practical examples of simple containers and the steps to get them running when searching online. I'll update this as my journey with podman quadlets continues. 

## Getting Started
Quadlets are a declarative way of building podman/docker containers on your device(s). They let you run your containers as systemd services, bringing all the benefits, including automatically starting/restarting and dependency management. They're a solid alternative to docker/podman compose. 

Importantly, quadlets run as user level services, which means their declarative files are stored in the user folder `~/.config/containers/systemd` by default. Quadlet files are written similarly to systemd unit files, except they contain (pun intended) a `[container]` header and section within them. 

A basic quadlet will look something like this:

```
[Unit]
Description=A minimal container

[Container]
# Use the centos image
Image=quay.io/centos/centos:latest

# This allows the container image to be automatically updated assuming the podman-auto-update.timer is running. 
AutoUpdate=registry

# Use volume and network defined below
Volume=/opt/$container:/data
Network=main
PublishPort=9090:9090


[Service]
# Restart service when sleep finishes
Restart=always
# Extend Timeout to allow time to pull the image - this is important to extend as pulling images can take a while
TimeoutStartSec=900
# ExecStartPre flag and other systemd commands can go here, see systemd.unit(5) man page.
ExecStartPre=/usr/share/mincontainer/setup.sh

[Install]
# Start by default on boot
WantedBy=default.target

```
After writing your `.container` file, first run `systemd --user daemon-reload` to reload systemd in **user** mode which generates the related service. Then start the service with `systemctl --user $containerName.service`. You can then monitor the service using `journalctl -u` as you would with any other systemd service. 

As these are systemd services, you can configure them as you would unit files, see `podman-systemd.unit(5)`. This can be especially useful when ordering dependencies, for instance having my karakeep container starting before the karakeep_chrome container. You can also refer to other systemd services in the `After=` line, for example starting a container after your Syncthing instance starts. Very useful, and much easier than requiring additional scripting. 


## Network
One particular challenge I found when configuring the quadlets in this repo was getting them to communicate correctly. For that a `.network` is requried and this needs to be included in the `[container]` section of the quadlet too. 
Networks are pretty simple to configure and I'm using a single standard network named `main` which is present across my unit files. My 'main' network file looks like this:

```
[Unit]
Description=Main Podman Network

[Network]
NetworkName=main
Subnet=10.68.100.0/24
Gateway=10.68.100.1

[Install]
WantedBy=default.target
```

## Automatic updates 
One major advantage of running quadlets over traditional docker containers is that these images can be updated automatically assuming `AutoUpdate=Registry` is included in the unit file. Then, the systemd timer, `podman-auto-update.timer`, needs to be enabled on a **system** level to ensure that the latest images are pulled and applied. Because this is a systemd timer, the frequency can be easily modified to suit your needs.

It is best practice not to automatically update containers blindly, and only enable for select units. 

## `default.target`
`WantedBy=default.target` is required for containers to be rootless as `multi-user.target` [is not defined in the user mode in systemd.](https://github.com/dani-garcia/vaultwarden/discussions/4206#discussioncomment-8015380)

## Debugging
Services aren't generated if there is **any** error in a `.container` file, even if that file isn't related to the `.service`

`/usr/lib/systemd/system-generators/podman-system-generator --user --dryrun daemon-reload` is extremely useful for debugging issues in your unit files as it produces a verbose summary of each container reload. 

## Links
Documentation on how to write the various unit files - https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html

Documentation on the podman-auto-update `.timer` - https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html

Blog post providing an overview of quadlets and how they compare to the old method of setting up podman containers - https://mo8it.com/blog/quadlet/
