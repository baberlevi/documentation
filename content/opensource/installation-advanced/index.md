---
title: "Advanced Installation"
description: "More flexibility and control over your Kerberos Open Source agents."
lead: "More flexibility and control over your Kerberos Open Source agents."
date: 2020-10-06T08:49:31+00:00
lastmod: 2020-10-06T08:49:31+00:00
draft: false
images: []
menu:
  opensource:
    parent: "opensource"
weight: 204
toc: true
---

**_Kerberos Open Source is deprecated, please [use Kerberos Agent](/agent/first-things-first) instead._**

Following installation methods are only for advanced users, who need to have more control over the Kerberos Open Source environment.

## Raspbian

If you have already an OS (e.g. Raspbian Buster) flashed to your Raspberry Pi, then it makes sense to install the Kerberos agent on top of your existing OS. There are two ways to achieve this:

1. Install the Kerberos agent using Docker (more info below),
2. or you can install the Kerberos agent manually (machinery + web).

This section will focus on option 2, and will show you how to install the Kerberos agent manually.

### Update OS

Let's start with updating the OS, and installing a couple of packages.

> We have tested the Raspbian installation on `Raspbian GNU/Linux 10 (buster)`. It might be that if you have a different version of Raspbian, you will need to install additional/different packages.

```ts
sudo apt-get update && sudo apt-get install -y ffmpeg
```

### Machinery

Download the debian file from the machinery repository.

> https://github.com/kerberos-io/machinery/releases

A `.deb` file is available for every version of the Raspberry Pi. For example if you are using a Raspberry Pi 4 for version 2.8.0, execute following command. You can change the version and Raspberry Pi board to your needs.

> Please make sure you pick the right board (rpi, rpi2, rpi3 or rpi4), and choose the version you want (e.g. 2.8.0). Replace below url, with your preferences.

```ts
wget https://github.com/kerberos-io/machinery/releases/download/v2.8.0/rpi4-machinery-kerberosio-armhf-2.8.0.deb
sudo dpkg -i rpi4-machinery-kerberosio-armhf-2.8.0.deb
```

Download the x265 library (version 160) from the machinery repository, as Raspbian Buster 10 only ships with version 165. Make sure you select the right board.

> `v2.8.0/rpi4-libx265.so.160` this will download the libx265 shared library for the Raspberry Pi 4.

```ts
wget https://github.com/kerberos-io/machinery/releases/download/v2.8.0/rpi4-libx265.so.160
sudo mv rpi4-libx265.so.160 /usr/lib/arm-linux-gnueabihf/libx265.so.160
```

Same applies for the libx264 library, download the right shared library.

> `v2.8.0/rpi4-libx264.so.148` this will download the libx264 shared library for the Raspberry Pi 4.

```ts
wget https://github.com/kerberos-io/machinery/releases/download/v2.8.0/rpi4-libx264.so.148
sudo mv rpi4-libx264.so.148 /usr/lib/arm-linux-gnueabihf/libx264.so.148
```

If you wish to use the Raspberry Pi Camera Module, make sure [to enable it](https://www.raspberrypi.org/documentation/configuration/camera.md) using `sudo raspi-config`.

```ts
sudo raspi-config
```

Add the libopenmaxil library path to the shared libraries.

```ts
sudo nano /etc/ld.so.conf
```

Copy and paste the following path in a new line.

```ts
/opt/cv / lib;
```

Update the library cache.

```ts
sudo ldconfig
```

Enable machinery to start on boot, and start the service.

```ts
sudo systemctl enable kerberosio
sudo service kerberosio start
```

### Web

Before you can run the web interface, you'll need to download and configure a webserver. We recommend to use Nginx, as it is a light-weight and fast webserver. The web interface is written in PHP, so we also need to download PHP and some packages. Update the packages and kernel.

Install Nginx and PHP 7.1 (+extensions). It is required to add another repository to download PHP 7.1.

```ts
sudo apt -y install lsb-release apt-transport-https ca-certificates
sudo wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list
sudo apt update
sudo apt-get install -y nginx php7.1 php7.1-curl php7.1-gd php7.1-fpm php7.1-cli php7.1-opcache php7.1-mbstring php7.1-xml php7.1-zip php7.1-mcrypt php7.1-readline
```

Creating a Nginx config.

```ts
sudo rm -f /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-enabled/kerberosio.conf
```

Copy and paste following config file; this file tells nginx where the web will be installed and that it requires PHP.

```ts
server
{
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/web/public;
    server_name kerberos.rpi;
    index index.php index.html index.htm;
    location /
    {
            autoindex on;
            try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$
    {
            fastcgi_pass unix:/var/run/php/php7.1-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
    }
}
```

Restart nginx

```ts
sudo service nginx restart
```

Now we have installed all the dependencies, we can download the web interface source code.

```ts
sudo mkdir -p /var/www/web && sudo chown www-data:www-data /var/www/web
cd /var/www/web
sudo -u www-data wget https://github.com/kerberos-io/web/releases/download/v2.8.0/web.tar.gz
sudo -u www-data tar xvf web.tar.gz .
sudo chown www-data -R storage bootstrap/cache config/kerberos.php
sudo chmod -R 775 storage bootstrap/cache
sudo chmod 0600 config/kerberos.php
```

Once everything is setup correctly, you should be able to browse towards the ip-address of your Raspberry Pi and see the Kerberos web interface.

### Auto removal

By default images or videos aren't removed automatically. This means that the Kerberos agent will keep writing to disk, even if there is no more space available on your SD card. When your SD card is full you'll be experiencing strange errors: a corrupt web interface, blank images or corrupt videos.

To resolve this your should install a simple bash script and initiate a cronjob which continuously poll your filesystem, and start removing media when your disk is getting full.

Create a bash script and copy following script.

```bash
nano /home/pi/autoremoval.sh
```

Copy following script (make sure the partition is correct, this is the default one for a Raspberry Pi).

```bash
partition=/dev/root
imagedir=/etc/opt/kerberosio/capture/
if [[ $(df -h | grep $partition | head -1 | awk -F' ' '{ print $5/1 }' | tr ['%'] ["0"]) -gt 90 ]];
then
    echo "Cleaning disk"
    find $imagedir -type f | sort | head -n 100 | xargs -r rm -rf;
fi;
```

Make the script executable.

```bash
chmod +x /home/pi/autoremoval.sh
```

Initiate a cronjob, and select the nano editor.

```bash
crontab -e
```

Append following line, to execute the autoremoval.sh script every 5min.

```bash
*/5 * * * * /bin/bash /home/pi/autoremoval.sh
```

## Generic

If you want to install the Kerberos agent from source on your working station or server, either for development or running the software, this is the preferred installation procedure. We are assuming that you use a Linux OS, when using Mac OSX the installation is slightly different.

> This was tested on a Ubuntu VM (18.04.3 (LTS) x64).

### Machinery

Update the packages and kernel, and install some development tools.

```ts
sudo apt-get -y update
sudo apt-get install -y git cmake subversion dh-autoreconf libcurl4-openssl-dev yasm libx264-dev pkg-config libssl-dev
```

Install the FFmpeg library with x264 support.

> It's recommended to install FFmpeg 3.1. Later versions might give issues at compilation or run time.

```ts
git clone https://github.com/FFmpeg/FFmpeg ffmpeg
cd ffmpeg && git checkout remotes/origin/release/3.1
./configure --enable-gpl --enable-libx264 --enable-shared --prefix=/usr
make && sudo make install
```

Go to your home directory, or any place your prefer and pull the machinery from Github. Afterwards create a build directory and start the compilation.

> You can change the `-j` attribute of `make -j8`to the number of cores of your compilation host.

```ts
cd && git clone https://github.com/kerberos-io/machinery
cd machinery && mkdir build && cd build
cmake .. && make -j8 && make check && sudo make install
```

After the machinery is build and installed succesfully, you can enable `kerberosio` to start on boot.

```ts
sudo systemctl enable kerberosio
sudo systemctl start kerberosio
```

### Web

If you want to install the web, you'll need to have a webserver (e.g. Nginx) and PHP running with some extensions. You'll also need NodeJS and npm installed, to install Bower.

```ts
cd ~
curl -sL https://deb.nodesource.com/setup_10.x | sudo bash -
sudo apt-add-repository -y ppa:ondrej/php
sudo apt-get -y update
sudo apt-get install -y git php7.1-cli php7.1-gd php7.1-mcrypt php7.1-curl php7.1-mbstring php7.1-dom php7.1-zip php7.1-fpm nodejs
sudo npm -g install bower
```

Next we'll need to install Nginx, and create a config file.

```ts
sudo apt-get -y install nginx
sudo rm -f /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-enabled/default
```

Assign Nginx to our web interface (which we will soon create).

```ts
server
{
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/web/public;
    server_name kerberos.rpi;
    index index.php index.html index.htm;
    location /
    {
            autoindex on;
            try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$
    {
            fastcgi_pass unix:/var/run/php/php7.1-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
    }
}
```

Now we have all the dependencies installed we can clone the web repository to our working station.

```ts
mkdir -p /var/www
cd /var/www && sudo git clone https://github.com/kerberos-io/web && cd web
```

Install the PHP dependency manager, `composer`, the install all the needed dependencies.

```ts
curl -sS https://getcomposer.org/installer | sudo php
sudo mv composer.phar /usr/bin/composer
sudo composer install
```

Change some permissions, to make sure we can write logging and caching files.

```ts
sudo chmod -R 777 storage
sudo chmod -R 777 bootstrap/cache
sudo chmod 777 config/kerberos.php
```

We will need to install a couple of more JavaScript libraries using bower.

```ts
cd public
sudo bower --allow-root install
service nginx restart
```

Once everything is setup correctly, you should be able to browse towards the ip-address of your Raspberry Pi and see the Kerberos web interface.

### Auto removal

By default images or videos aren't removed automatically. This means that the Kerberos agent will keep writing to disk, even if there is no more space available on your SD card. When your SD card is full you'll be experiencing strange errors: a corrupt web interface, blank images or corrupt videos.

To resolve this your should install a simple bash script and initiate a cronjob which continuously poll your filesystem, and start removing media when your disk is getting full.

Create a bash script and copy following script.

```bash
nano /home/[user]/autoremoval.sh
```

Copy following script (make sure the partition is correct, this is the default one for a Raspberry Pi).

```bash
partition=/dev/root
imagedir=/etc/opt/kerberosio/capture/
if [[ $(df -h | grep $partition | head -1 | awk -F' ' '{ print $5/1 }' | tr ['%'] ["0"]) -gt 90 ]];
then
    echo "Cleaning disk"
    find $imagedir -type f | sort | head -n 100 | xargs -r rm -rf;
fi;
```

Make the script executable.

```bash
chmod +x /home/[user]/autoremoval.sh
```

Initiate a cronjob, and select the nano editor.

```bash
crontab -e
```

Append following line, to execute the autoremoval.sh script every 5min.

```bash
*/5 * * * * /bin/bash /home/[user]/autoremoval.sh
```
