# Prerequisites:

- schroot
- deboostrap
- *inotify-tools*
- *npm install jquery*

*Command which prepends with $ is run as current user.*

*Command which prepends with # has to be run as root.*

*Replace **<USER>** with your user name.*

# Preparing directory for chroot env.:

I use encrypted folder for work stuff in my home directory, I also want to have chroot env. there because it will contain VPN configuration with my password:

	$ mkdir -p ~/work/vpn-client/chroot

# Configuring schroot:

Ubuntu 16.04 (Xenial) 32-bit/64-bit is used as chroot env. because Kerio VPN Client works without any troubles under it.

Append following into `/etc/schroot/schroot.conf`:

	[xenial]
	description=Ubuntu 16.04
	directory=/home/<USER>/work/vpn-client/chroot
	personality=linux32
	groups=root
	root-groups=root

# Setting up chroot with debootstrap:

**Use (for 32-bit)**

	# debootstrap --variant=minbase --arch i386 xenial /home/<USER>/work/vpn-client/chroot http://archive.ubuntu.com/ubuntu/

**OR (for 64-bit)**

	# debootstrap --variant=minbase --arch amd64 xenial /home/<USER>/work/vpn-client/chroot http://archive.ubuntu.com/ubuntu/

# Mounting:

	# export MOUNT_POINT=/home/<USER>/work/vpn-client/chroot
	# mount -o bind /proc $MOUNT_POINT/proc
	# mount -o bind /dev $MOUNT_POINT/dev
	# mount -o bind /dev/pts $MOUNT_POINT/dev/pts
	# mount -o bind /sys $MOUNT_POINT/sys

# Preparing system and installing Kerio VPN Client:

	# schroot -c xenial -u root

**Following commands are run in chroot:**

	# apt-get update
	# apt-get --no-install-recommends install wget debconf openssl iputils-ping

**Use (for 32-bit)**

	# wget http://download.kerio.com/dwn/kerio-control-vpnclient-linux.deb

**OR (for 64-bit)**

	# wget http://download.kerio.com/dwn/kerio-control-vpnclient-linux-amd64.deb
	# dpkg -i kerio-control-vpnclient*.deb

Now you will be prompted to enter Kerio VPN server and your credentials.
After all this is successfully configured it's time to test your connection to network behind VPN.

	# /etc/init.d/kerio-kvc start
	# ping <some internal IP or hostname>

If you are getting responses back, you are able to access internal network. You can now exit from chroot…

	# exit

…and check if new network interface kvnet is created:

	# ip addr

# Usage

I've created two shell scripts and one Node.js script for seamless day to day usage:

**start.sh**

	#!/bin/bash
	VPN_CLIENT_DIR=/home/<USER>/work/vpn-client
	MOUNT_POINT=$VPN_CLIENT_DIR/chroot
	VPN_DEV=kvnet

	sudo mount -o bind /proc $MOUNT_POINT/proc
	sudo mount -o bind /dev $MOUNT_POINT/dev
	sudo mount -o bind /dev/pts $MOUNT_POINT/dev/pts
	sudo mount -o bind /sys $MOUNT_POINT/sys

	sudo cp /etc/resolv.conf $MOUNT_POINT/etc/resolv.conf

	sudo schroot -c xenial -d / -u root /etc/init.d/kerio-kvc start

	EVENT=$(inotifywait -q -e modify --format '%e' $MOUNT_POINT/etc/resolv.conf)
	if [ $? -eq 0 ] && [ "$EVENT" = "MODIFY" ]; then
		sudo resolvconf -d NetworkManager
		cat $MOUNT_POINT/etc/resolv.conf | sudo resolvconf -a $VPN_DEV.inet
		echo "/etc/resolv.conf updated"
	else
		echo "something was wrong with inotifywait"
		exit 1
	fi

	cd $VPN_CLIENT_DIR
	curl -s -b TOTP_CONTROL=<TOTP> http://<Internal GW IP>:4080/nonauth/totpVerify.php | ./kerio-control-parse-2fa.sh

**stop.sh**

	#!/bin/bash
	MOUNT_POINT=/home/<USER>/work/vpn-client/chroot
	VPN_DEV=kvnet

	sudo schroot -c xenial -d / -u root /etc/init.d/kerio-kvc stop

	sudo umount $MOUNT_POINT/proc $MOUNT_POINT/dev/pts $MOUNT_POINT/dev $MOUNT_POINT/sys

	sudo resolvconf -d $VPN_DEV.inet
	sudo systemctl reload NetworkManager.service
	echo "/etc/resolv.conf updated"

**kerio-control-parse-2fa.sh**

*(requires npm install jquery in /home/<USER>/work/vpn-client directory)*

	#!/usr/bin/node
	var fs = require('fs');
	var jsdom = require('jsdom');
	var jquerySrc = fs.readFileSync('./node_modules/jquery/dist/jquery.js', 'utf-8');
	var html = [];

	process.stdin.setEncoding('utf8');

	process.stdin.on('readable', function() {
		var chunk = process.stdin.read();
		if (chunk !== null)  html.push(chunk);
	});

	process.stdin.on('end', function() {
		jsdom.env({
			html: html.join(''),
			src: [jquerySrc],
			done: function (errors, window) {
				if (!errors) {
					var $ = window.jQuery;
					console.log('%s: %s', $('#header').text(), $('#customMessage').text());
				}
				else {
					console.error(errors);
				}
			}
		});
	});

*Please note you need to have `inotify` installed. For Arch Linux use following:*

	# pacman -S inotify-tools

# Resources

- http://www.binarytides.com/setup-chroot-ubuntu-debootstrap/
- https://wiki.ubuntu.com/DebootstrapChroot
- http://ubuntuforums.org/showthread.php?t=1156240
- https://github.com/masterkorp/openvpn-update-resolv-conf/blob/master/update-resolv-conf.sh
