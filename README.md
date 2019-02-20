# Setting up a Pi-Hole VPN on a VPS

TODO switch back to Ubuntu (because I'm a stubborn idiot)

# Introduction

The purpose of this guide is to document the steps I take to set up [Pi-Hole](https://pi-hole.net/) on a VPS. The ultimate goal is to have an ad-blocker that can work with any device connected to the VPS with a VPN connection. 

All the software will be installed using Docker containers and `docker-compose`. Use a `docker-compose.yml` file will allow you to quickly deploy your Pi-Hole VPN to any VPS as necessary.

After completing this tutorial you will have:

- Your own recursive DNS resolver using **unbound**
- A **Pi-Hole** accessible from anywhere
- A VPN server using **Wireguard**
- Auto-updating containers using **Watchtower**

## Prerequisites

In order to follow this tutorial you will need to have a VPS with at least 512 MB of memory, although I would personally recommend at least 1 GB if you plan on having a large number of blocklists. This guide assumes that you are using Debian 9 and Pi-Hole Version 4.2. Other distros will mostly likely work, but I have only tested the steps covered in this tutorial on Debian 9.

Companies like **DigitalOcean** provide [tutorials](https://www.digitalocean.com/docs/droplets/how-to/create/) for creating a VPS on their servers.

## Initial Server Setup

We will be using `ssh` to remotely log into the VPS and configure it. If you are on a Unix-based operating system, it should already be installed. If you are Windows, you will need to install [PuTTY](http://www.putty.org/). Make sure you know your server's IP address and login credentials.

This section essentially covers all of the steps from [**DigitalOcean**'s tutorial for setting up Ubuntu](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04), but with a few differences. We will be creating a specific user, `pi`, that will use for logging into our VPS and running the Docker containers.

### `root` Login

When you have your server's IP address and root passphrase, log into the server as the `root` user

```shell
ssh root@your_server_ip
```

If you are not asked to create a new passphrase, use the `passwd` command to generate a new one. Although we will be disabling password authentication, be sure to create or [generate](http://passwordsgenerator.net/) a secure passphrase anyway.

### Update VPS

We will update and then reboot the VPS to ensure everything is up-to-date and the latest security patches are installed.

Update your VPS (assuming you are using Ubuntu/Debian):

```
sudo apt update && sudo apt upgrade -y
```

Once the updates have been installed, reboot the VPS:

```
sudo reboot
```

### Create user `pi`

Log back into your VPS as `root` and create new user `pi`

```shell
adduser pi
```

Grant root privileges to `pi`

```shell
usermod -aG sudo pi
```

### Public Key Authentication

[Public Key Authentication](https://the.earth.li/~sgtatham/putty/0.55/htmldoc/Chapter8.html) provides an alternative method of identifying yourselve to a remote server and increases the overall security of your server.

If you do not already have an SSH key, you will need to create one on your local computer

```shell
ssh-keygen
```

Save your key in the default file (where `$user` is your user)

```shell
Enter file in which to save the key (/Users/$user/.ssh/id_rsa):
```

Create a secure passphrase. You will need to enter this passphrase each time you utilize your SSH key

Copy the public key from your local machine to your remote server with `ssh-copy-id`

```shell
ssh-copy-id pi@your_server_ip
```

- If you opted to add SSH during the server creation process anyway, this method will not work.

You should repeat these steps for each device you want to access the server, including desktops, laptops, tablets, and mobile phones.

#### Disable Passphrase Authentication

Once you have added SSH keys from all of your devices, we can disable passphrase authentication.

Log into your server as `root`, if you are not already logged in

```shell
ssh root@your_server_ip
```

Open the SSH daemon configuration file

```shell
sudo nano /etc/ssh/ssdh_config
```

- Find the line containing `PasswordAuthentication` and uncomment it by deleting the preceeding `#`. Change it's value to **no**
- Find the line containing `PubkeyAuthentication` and ensure it's value is set to **yes**
- Find the line containing `ChallengeResponseAuthentication` and ensure it's value is set to **no**

Save your changes and close the file

- `CTRL + X`
- `Y`
- `ENTER`

While still logged in as `root`, open a new terminal window and test logging in as `pi` and verify that the public key authentication works

```shell
ssh pi@your_server_ip
```

### Add environment variables

Many Docker containers require similar settings, such as timezones and application directories. To save us from having to type these values multiple times, we will set them as environment variables instead and use them in our `docker-compose.yml` file. We will need to reboot the VPS again after setting these variables.

Log in to your VPS as the `pi` user and open `/etc/environment`:

```
sudo nano /etc/environment
```

Add the following environment variables to the bottom of the file, making changes where necessary.

TODO add instructions for quickly obtaining interface

```
TZ=America/New_York                 # Set this to your timezone
SERVER_IP=127.0.0.1                 # Set this to your VPS's IPv4 Address
SERVER_IPV6=::1                     # Set this to your VPS's IPv6 Address, if using IPv6
DOCKER_DIR=/home/pi/docker
WEB_PASSWORD=myawesomepassphrase    # Your passphrase for the Pi-Hole web interface
INTERFACE=eth0                  # Set to the main interface on your VPS
```

Save your changes and close the file. Then reboot:

```
sudo reboot
```

### (Optional) Install **Mosh**

[**Mosh**](https://mosh.org/), or **mo**bile **sh**ell, is a remote terminal application that allows roaming and intermittent connectivity. It's intended as a replacement for SSH but both can be used on the same server.

```
# Update your sources, if necessary
sudo apt update

# Install mosh
sudo apt install mosh
```

### Set up **`ufw`**

We will set up a basic firewall, [`ufw`](https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29), that will restrict access to certain services on the VPS. Specifically, we want to ensure that only ports needed for our applications. Additional ports can be opened later depending on your specific needs.

We will be opening ports for secure FTP so that `.ovpn` files needed for connecting to our VPN later can be retrieved via a FTP application such as [Filezilla](https://filezilla-project.org/) or [Transmit](https://panic.com/transmit/).

To set up `ufw`, enter the following commands:

TODO: Add rules for Wireguard

```shell
# Apply basic defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Open ports for OpenSSH
sudo ufw allow OpenSSH

# Optionally, allow all access from your IP Address
sudo ufw allow from $yourIPAddress

# Open ports for secure FTP
sudo ufw allow sftp

# Open ports for Mosh if you installed it
sudo ufw allow mosh
```

## Set up Docker and `docker-compose`

We will be using [**Docker**](https://www.docker.com/) to run the software covered in this guide in containers. We will install Docker and Docker Compose, add the `pi` user to the Docker group, and then create a Docker Compose configuration file that will be expanded throughout the guide.

### Install Docker

Docker provides a bash script that we can use to automate the entire installation.

Go to your temporary directory:

```
cd /tmp
```

And run the following command to install Docker:

```
curl -fsSL get.docker.com -o get-docker.sh && sh get-docker.sh
```

### Add `pi` user to Docker Group

Running docker containers requries `sudo` privileges. To avoid needing `sudo` for every command or having to switch to the `root` user, we will add the `pi` user to the `docker` group that was added when we installed docker.

To add the `pi` user to the `docker` group, use the following command:

```shell
sudo usermod -aG docker ${USER}
```

Log out of your VPS and log back in as the `pi` user. `pi` should now be part of the `docker` group. To test this, and to test that Docker has been successfully installed, run the following command:

```
docker run hello-world
```

### Install `docker-compose`

There are multiple options for installing `docker-compose`, including installing it via `pip` and running it in a Docker container itself. We'll be downloading the binary from the GitHub repository and adding executable permissions to it.

Be sure to check the [release page](https://github.com/docker/compose/releases) and ensure you run the following command with the most recent release.

Download `docker-compose`:

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

Apply executable permissions:

```
sudo chmod +x /usr/local/bin/docker-compose
```

### Create `docker-compose.yml` Configuration File

Create a docker folder in `pi`'s home directory:

```
mkdir ~/docker
```

Type in the following command to set the necessary permissions. They will make any new subfolders inherit the permissions from the docker folder. We will be storing configuration folders for our containers in the docker folder, hence why we want such liberal permissions.

```
sudo chmod -R 775 ~/docker
```

Now create the Docker Compose file:

```
nano ~/docker/docker-compose.yml
```

And add the following two lines:

```
version: "3.6"
services:
```

## Install `unbound`

The first service we will install is `unbound`. TODO: explain why

Add the following lines to your `docker-compose.yml` file, after the `services:` line:

```yaml
version: "3.6"
services:

  unbound:
    container_name: unbound_docker
    image: klutchell/unbound:latest
    ports:
      - "5353:53/tcp"
      - "5353:53/udp"
    environment:
      TZ: ${TZ}
    restart: always
```

## Install **Pi-Hole**

- add pi-hole to docker compose file

- start docker containers with command:
  ```
  docker-compose -f ~/docker/docker-compose.yml up -d
  ```

---------------------------------------------------------------------------

### (Optional) Configure **Pi-Hole**

**Pi-Hole** allows you to customize what websites you want to block and allows to you whitelist any false positives (e.g., unblocking Netflix or Facebook). **Pi-Hole** developer [WaLLy3K](https://github.com/WaLLy3K) provides a [popular collection of blocklists](https://wally3k.github.io/) that you can add to your own blocklists. Another blocklist collection is provided by [the Block List Project](https://tspprs.com/).

I would also recommend checking out [this GitHub repository](https://github.com/anudeepND/whitelist) that will load commonly whitelisted domains (e.g., Facebook, Instagram, XBox Live) into your Pi-Hole.

Finally, I would suggest following [this guide](https://docs.pi-hole.net/guides/unbound/) from the official Pi-Hole documentation to set up [**unbound**](https://nlnetlabs.nl/projects/unbound/about/) as your own recursive DNS server (rather than using a public DNS server such as Google DNS or Cloudflare). This will help to further increase the privacy of your DNS queries.

## Install **PiVPN**

Installing **PiVPN** will be just as easy as installing **Pi-Hole**, although there is a bit more configuration required on our part for **PiVPN**. **PiVPN** automatically installs an [**OpenVPN** server](https://openvpn.net/) for us as well as any additional required software. The script will also automatically open ports in `ufw` so that an **OpenVPN** client can communicate with our VPS.

Please note that on a Raspberry Pi, we would be asked to select a network interface, but since we are on a VPS the only available interface is `eth0` and that is automatically selected for us as well as the static IP address.

Start by running the [**PiVPN** installer](https://github.com/pivpn/pivpn/blob/master/auto_install/install.sh)

```shell
curl -L https://install.pivpn.io | bash
```

- When asked to choose a local user to hold your `.ovpn` configuration files, select the user `pi`
- When asked about enabling `UnattendedUpgrades`, pick yes
- When asked to select the protocol, pick `UDP`
- When asked to select the port, either accept the default `1194` or enter a random port (e.g., `11948`)
- When asked to set the size of your encryption key, select `2048`
  - Generating the encryption key will take a few minutes
- When asked to select a Public IP or DNS, select your server's IP address
- When asked to select a DNS provider, select the `custom` option and enter the IP address of your server

Once the installer is finished, allow it to reboot your VPS

## Configure **Pi-Hole** and **PiVPN**

Now that both **Pi-Hole** and **PiVPN** are installed, there are a couple of critical steps we must take before we can start generating `.ovpn` configuration files and connecting to our VPS. Specifically we want to ensure that **PiVPN** uses **Pi-Hole** as it's DNS server and that we can connect using an **OpenVPN** client.

### `dnsmasq`

First we will create a configuration file for `dnsmasq`, the DNS service that powers **Pi-Hole**

Log into your server as `pi` if you are not logged in already:

```shell
ssh pi@your_server_ip
```

Create a new configuration file called `02-pivpn.conf`:

```shell
sudo nano /etc/02-pivpn.conf
```

Add the following line to the file:

```shell
listen-address=127.0.0.1, your_server_ip, 10.8.0.1
```

Save and exit the file, and restart **Pi-Hole**'s FTL service:

```shell
pihole restartdns
```

### Network Adjustments

Second we will need to adjust rules in `ufw` to allow **OpenVPN** to correctly route connections. This section covers Step 8 from [**DigitalOcean**'s guide to setting up OpenVPN on a VPS](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04#step-8-adjust-the-server-networking-configuration).

We will start by modifying the `sysctl.conf` file to allow IP forwarding:

```shell
sudo nano /etc/sysctl.conf
```

Look for the line that contains `net.ipv4.ip_forward`. If there is a `#` character prepended, remove it to uncomment the line. Ensure that it is set to `1` and not `0`.

```shell
net.ipv4.ip_forward=1
```

Save and close the file. Then instruct `sysctl` to reload it.

```shell
sudo sysctl -p
```

Then we will modify `ufw` to allow masquerading of client connections. Before we can modify any rules, we need to find the public network interface of our VPS:

```shell
ip route | grep default
```

Your public interface will follow the word "`dev`" in the output. For example:

```shell
default via 203.0.113.1 dev eth0  proto static  metric 600
```

If your public interface is not `eth0`, make note of what it is. We will be using that interface to modify a `ufw` file that loads rules before regular rules are loaded. We will be adding a rule that will masquerade any traffic comming in from the VPN.

Open the `before.rules` file:

```shell
sudo nano /etc/ufw/before.rules
```

Towards the top of `before.rules` add the following text, starting with `# START OPENVPN RULES`:

```text
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES

```

Save and close the `before.rules`.

Finally, we need to tell `ufw` to allow forward packets by default. Open the `/etc/default/ufw` file:

```shell
sudo nano /etc/default/ufw
```

Fine the line containing `DEFAULT_FORWARD_POLICY`. Change the value to `ACCEPT` if necessary:

```text
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Save and close `/etc/default/ufw`.

Enter the following commands to restart `ufw` and **OpenVPN**:

```shell
# Restart ufw
sudo ufw disable
sudo ufw enable

# Restart OpenVPN
sudo service openvpn reload
```

## Let's Encrypt

The following section is optional and **requires you to have your own domain name**, but it will configure your Pi-Hole's web interface to use `https` courtesy of [Let's Encrypt](https://letsencrypt.org/) and [Certbot](https://certbot.eff.org/). It can be considered overkill just for Pi-Hole, but it certainly doesn't hurt. First we will acquire the certificate and then we will configure `lighttdp` to automatically redirect any `http` requests to `https`. The steps are based on [this reddit post](https://www.reddit.com/r/pihole/comments/6e2jyr/self_signed_ssl_cert_for_the_admin_login_page/di7ct0b).

### Acquiring the certificate

1. Log into your remote server again either as `root` or with root privileges.
2. Go to this [Certbot page](https://certbot.eff.org/lets-encrypt/ubuntubionic-other) (for Ubuntu 18.04) and following the **Install** commands to install Certbot on your server.
3. Perform a dry run to acquire a certificate for your domain. For example:
    ```bash
    certbot certonly --webroot -w /var/www/html -d example.com --dry-run
    ```
4. If acquiring the certificate was successful, run the same command again without `--dry-run`. For example:
    ```bash
    certbot certonly --webroot -w /var/www/html -d example.com
    ```
5. Edit the file `/etc/lighttpd/conf-available/10-ssl.conf`. Replace `example.com` with your own domain name:
    ```bash
    ssl.pemfile = "/etc/letsencrypt/live/example.com/combined.pem"
    ssl.ca-file = "/etc/letsencrypt/live/example.com/chain.pem"
    ```
6. Run the following commands, replacing `example.com` with your domain name:
    ```bash
    ln -s /etc/lighttpd/conf-available/10-ssl.conf /etc/lighttpd/conf-enabled/10-ssl.conf
    cd /etc/letsencrypt/live/example.com/
    cat privkey.pem cert.pem > combined.pem
    ```
7. Restart `lighttpd`:
    ```bash
    sudo systemctl restart lighttpd
    ```
    `https` should now be enabled on your web interface.
8. Add a cron job to automatically renew the certificate every 90 days. Open `/etc/crontab` and add the following line:

    ```bash
    47 5 * * * root certbot renew --quiet --no-self-upgrade --renew-hook "cat $RENEWED_LINEAGE/privkey.pem $RENEWED_LINEAGE/cert.pem > $RENEWED_LINEAGE/combined.pem;systemctl reload-or-try-restart lighttpd"
    ```

### Configure the Redirect

Open the `lighttpd` configuration file, `/etc/lighttpd/lighttpd.conf`, and add the following block of code:

```bash
compress.cache-dir = "/var/cache/lighttpd/compress/"
compress.filetype = ( "application/javascript", "text/css", "text/html", "text/plain" )

# [add after the syntax above]

# Redirect HTTP to HTTPS
$HTTP["scheme"] == "http" {
    $HTTP["host"] =~ ".*" {
        url.redirect = (".*" => "https://%0$0")
    }
}
```

Restart `lighttpd` again:

```bash
sudo systemctl restart lighttpd
```

Your web interface should now automatically redirect any `http` requests to `https`.


## Sources

- **Pi-Hole**
  - [Official Website](https://pi-hole.net/)
  - [Github](https://github.com/pi-hole/pi-hole)
- **PiVPN**
  - [Official Website](http://www.pivpn.io/)
  - [Github](https://github.com/pivpn/pivpn)
- [Digital Ocean: Initial Server Setup with Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)
- [Secure Password Generator](http://passwordsgenerator.net/)
- [Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.55/htmldoc/Chapter8.html)
- [Mosh](https://mosh.org/)
- [Debian Wiki: Uncomplicated Firewall](https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29)
- [FileZilla](https://filezilla-project.org/)
- [Transmit](https://panic.com/transmit/)
- [Static vs. dynamic IP addresses](https://support.google.com/fiber/answer/3547208?hl=en)
- [The Big Blocklist Collection](https://wally3k.github.io/)
- [Pi-Hole: Commonly Whitelisted Domains](https://discourse.pi-hole.net/t/commonly-whitelisted-domains/212)
- [OpenVPN](https://openvpn.net/)
- [How To Set Up an OpenVPN Server on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04#step-8-adjust-the-server-networking-configuration)
- <https://itchy.nl/raspberry-pi-3-with-openvpn-pihole-dnscrypt>
