Deprecated in favor of https://github.com/natanlao/dotfiles

# homelab

This repository documents the configuration of my "homelab," which is really
just a Raspberry Pi and a router. So not much of a homelab. I'm documenting it
here so that if / when it breaks, I can restore it in a similar way, and because
writing things out helps me understand what I'm doing and better navigate the
decision-making process.


## Raspberry Pi

I use a Raspberry Pi 3B+ to host some network-accessible services:

* UniFi Network Controller, for controlling my network hardware.
* Pi-hole, for ad blocking.
* An SFTP server, to expose an attached hard drive that I use for backups.
* Watchtower, to automatically update all of the above

These services run as Docker containers, specified in
[docker-compose.yml](docker-compose.yml). I chose to use Docker for the
streamlined install process and to keep things isolated from each other, given
the amount of moving parts. Each of these components could have been installed
without Docker, but it seemed more straightforward -- it's a lot easier to
specify the name of a container published on Docker Hub than it is to encode
and maintain some install process independently. It's also super useful to be
able to configure and see used ports in YAML.

The Pi runs on Ubuntu Server arm64. I considered a few other distributions:

* Raspbian (Raspberry OS?) doesn't yet have a stable aarch64 build.
* Photon OS seems interesting, but I wasn't able to successfully configure it.
  I could never actually get it to boot. Plus, it didn't have a reliable
  headless setup process that I could find.
* balenaOS also seemed cool, but it wasn't accessible enough to me without a
  Balena account.
* I didn't want to make an account to use Ubuntu Core. Ubuntu Minimal Core?
  Ubuntu Snappy Core? Ubuntu Snappy?


### Setup

To set up the device, we need to:

1. Download an image of Ubuntu Server.
1. Configure the image for unattended and headless setup.
1. Set up the Pi using Ansible.
1. Reconfigure each container.

Alternatively, if you have an HDMI cable, you can skip the first two steps.


#### Image configuration

Start by downloading an arm64 preinstalled server image of
[Ubuntu Server][server] -- the xz, not the iso. Flash it, and configure it:

1. Copy [usercfg.txt](usercfg.txt) to `/system-data/usercfg.txt`. This isn't
   imperative, it just configures some power-saving options.
1. Generate a `cloud-init` config file from `user-data.tmpl` with your SSH
   public key and a hostname and place it in `/system-data/user-data`, a la:

       $ env HOSTNAME=myhostname SSH_PUBKEY=(cat ~/.ssh/id_ed25519.pub) envsubst < pi/user-data.tmpl > /mnt/system-data/usercfg.txt

   on Fish. Otherwise, you want something like:

       $ HOSTNAME=myhostname SSH_PUBKEY=$(cat ~/.ssh/id_ed25519.pub) envsubst < pi/user-data.tmpl > /mnt/system-data/usercfg.txt


Insert the SD card into the Raspberry Pi, and wait a few minutes for it to boot.
It's probably a good idea to give it a static IP.

If you're doing an interactive install, you should still copy those files --
`user-data` is configured such that it will do setup things regardless of
whether the install is interactive or unattended.

  [server]: http://cdimage.ubuntu.com/ubuntu/releases/20.04/release/


#### Ansible

We need some roles first:

    $ ansible-galaxy install jnv.unattended-upgrades
    $ ansible-galaxy install geerlingguy.docker

Once the Pi is booted, SSH into it and change the default password. Make sure
that you can authenticate and connect to the Pi before moving on, as the
playbook will configure ufw and ssh in a way that could lock you out if you
aren't careful.

Once you're ready:

    $ ansible-playbook bootstrap.yml -i ${IP_ADDRESS_OF_PI}, -u ubuntu

The trailing comma, I'm told, is important.


### Services

Configuration for all of these servers is stored on the remote `homelab/`
folder. In theory, you can provision another device using the process above,
move the `homelab/` folder, and everything would run the same (with minimal
manual tweaking).

#### Pi-Hole

The Pi-Hole install might choke on port 53 already being in use. This is because
of dnsmasq, which runs by default on Ubuntu Server. It can be replaced with
Pi-Hole's dnsmasq instance.

#### Unifi Network Controller

By default, the controller is limited to a gigabyte of memory. Given that the
3B+ only has a gigabyte of memory, and that we're only managing a single device
with this controller instance, I've found that it's acceptable to limit its
consumption to 256M. It might be necessary to increase that limit if more
devices are added.

#### SFTP Server

Some manual setup is required:
* You'll need to manually configure /etc/fstab to automatically mount a
  connected drive on boot.
* If you don't have them already, you'll need SSH keys for the remote user.
* There is only one configured user, `backup`. Add public keys using
  `ssh-copy-id`.

When configuring clients, remember to specify the backup destination as
`/home/backup/share`.


### Notes

I referenced:

* chhantyal/5minutes
* guillaumevincent/Ansible-My-First-5-Minutes-On-A-Server

If you're using @geerlingguy's Docker role (geerlingguy/ansible-role-docker), be
sure to specify the architecture manually as I did (i.e. `docker_apt_arch:
arm64`) or the role will fail with some opaque error that I can't remember.

The Pi-Hole install might choke on port 53 already being in use. This is
because of dnsmasq.
