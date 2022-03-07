# ISO-of-doom-2022

A small read on how to build the Offsec ISO of doom, but actually make it work.

## Preamble

So work had asked me to make something after dealing with constant client VPN issues, to help us gain and maintain access to client networks with little interaction and low maintence. This is when I stumbled upon the annoying nusience that is Offensives Securitys version of Live Build for Kali.

A long time ago, Offensive Security put out an article named [The ISO of DOOM](https://www.offensive-security.com/kali-linux/kali-linux-iso-of-doom/). In their words,

> The idea we had was to build an “unattended self-deploying” instance of Kali Linux that would install itself on a target machine along with a customized configuration requiring no user input whatsoever. On reboot after the installation completes, Kali would automagically connect back to the attacker using a reverse OpenVPN connection. The VPN setup would then allow the attacker to bridge the remote and local networks as well as have access to a full suite of penetration testing tools on the target network.

Cool concept right? A ISO that you could fire and forget in a network that would allow the pentesting team to access the network with minimal involment from the System Administrator or Network Administration team. It would do this with an OpenVPN client, that would call back to the OpenVPN server that attacker/pentester controls. Closely resembling a reverse shell. The article was made in 2013 which is a little dated (Currently 2022), but luckly, Offensive Security has [another article](https://www.kali.org/docs/development/dojo-mastering-live-build/) [Wayback Mirror](https://web.archive.org/web/20211023233914/https://www.kali.org/docs/development/dojo-mastering-live-build/) that was "Updated" Jan 2022. This article turned out to be inaccurate, outdated, or just plain wrong.

## 

## Setup

For the purpose of portablility, this is all going to be one large file. Offsec does provide a template git that does make your ISO, at its core, a Kali Machine.

 I started with [this barebones template]("https://gitlab.com/kalilinux/recipes/live-build-config-examples/-/raw/master/offsec-awae-live.sh") offsec had for their AWAE course.

Running this script did compile are working unattented Kali install, but it did not have any auto start functionalilty. It was the only hope I had for this idea to work at all, as the articles above could not even get a working ISO to compile.

After removing some of the personalization, we are left with:

```bash
sudo apt-get update
sudo apt-get install -yqq git live-build simple-cdd cdebootstrap curl

## Get base-image build-script
if [ -e "/opt/kali-build/" ]; then
  cd /opt/kali-build/
  sudo git reset --hard HEAD >/dev/null
  sudo git clean -f -d >/dev/null
  sudo git pull >/dev/null
else
  sudo git clone https://gitlab.com/kalilinux/build-scripts/live-build-config.git /opt/kali-build/
  cd /opt/kali-build/
fi

## Select packages for image (Live)
cat <<'EOF' | sudo tee kali-config/variant-default/package-lists/kali.list.chroot >/dev/null
## Graphical desktop
kali-desktop-xfce

EOF



## Boot menu for image
cat <<'EOF' | sudo tee kali-config/common/includes.binary/isolinux/install.cfg >/dev/null
label install
    menu label ^Unattended Install
    linux /install/vmlinuz
    initrd /install/initrd.gz
    append vga=788 -- quiet file=/cdrom/install/preseed.cfg locale=en_US keymap=us hostname=kali domain=local.lan
EOF



## Point to our boot menu
cat <<'EOF' | sudo tee kali-config/common/includes.binary/isolinux/isolinux.cfg >/dev/null
include menu.cfg
ui vesamenu.c32
default install
prompt 0
timeout 1
EOF

## Add to preseed file
cat <<'EOF' | sudo tee -a kali-config/common/includes.installer/preseed.cfg >/dev/null
d-i debian-installer/locale string en_US.UTF-8
d-i console-keymaps-at/keymap select us
d-i keyboard-configuration/xkb-keymap select us
d-i mirror/http/proxy string
d-i mirror/suite string kali-rolling
d-i mirror/codename string kali-rolling

d-i clock-setup/utc boolean true
d-i time/zone string Etc/UTC

# Partitioning
d-i partman-auto/method string regular
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-auto/choose_recipe select atomic
d-i partman-auto/disk string /dev/sda
d-i partman/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label boolean true

# Change default hostname
d-i netcfg/get_hostname string kali
d-i netcfg/get_domain string unassigned-domain
#d-i netcfg/choose_interface select auto
d-i netcfg/choose_interface select eth0
d-i netcfg/dhcp_timeout string 60

d-i hw-detect/load_firmware boolean false

d-i passwd/user-fullname string Kali Linux
# Normal user's username
d-i passwd/username string kali
# Normal user's password (clear text)
d-i passwd/user-password password kali
d-i passwd/user-password-again password kali

d-i apt-setup/use_mirror boolean true
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean false
d-i grub-installer/bootdev string /dev/sda
d-i finish-install/reboot_in_progress note

kismet kismet/install-setuid boolean false
kismet kismet/install-users string

sslh sslh/inetd_or_standalone select standalone

mysql-server-5.5 mysql-server/root_password_again password
mysql-server-5.5 mysql-server/root_password password
mysql-server-5.5 mysql-server/error_setting_password error
mysql-server-5.5 mysql-server-5.5/postrm_remove_databases boolean false
mysql-server-5.5 mysql-server-5.5/start_on_boot boolean true
mysql-server-5.5 mysql-server-5.5/nis_warning note
mysql-server-5.5 mysql-server-5.5/really_downgrade boolean false
mysql-server-5.5 mysql-server/password_mismatch error
mysql-server-5.5 mysql-server/no_upgrade_when_using_ndb error
EOF


time sudo ./build.sh \
  --debug \
  --live \
  --version offsec-awae

## Output
ls -lh /opt/kali-build/images/*.iso

## Done
echo "[i] Done"
```

A few things we will be changing within the existing template, is the preseed.cfg file, and the selected packages.

#### Base Image and Live Build Tools

Don't touch this. You need this.

```bash
## Make sure we have programs needed
sudo apt-get update
sudo apt-get install -yqq git live-build simple-cdd cdebootstrap curl

## Get base-image build-script
if [ -e "/opt/kali-build/" ]; then
  cd /opt/kali-build/
  sudo git reset --hard HEAD >/dev/null
  sudo git clean -f -d >/dev/null
  sudo git pull >/dev/null
else
  sudo git clone https://gitlab.com/kalilinux/build-scripts/live-build-config.git /opt/kali-build/
  cd /opt/kali-build/
fi
```

#### Packages

In this section, you can specify packages that you want auto installed with your ISO. Anything listed below will be downloaded when you compile the ISO and will **not** need to be re-downloaded when booting from the ISO. You can look at the packages that kali offers in their repositories [here](https://www.kali.org/tools/kali-meta/). You can see below that I used only kali-core, kali-tools-top10 and a few tools of choice like mosh, python3-pip, an most importantly, openVPN.

```bash
cat <<'EOF' | sudo tee kali-config/variant-default/package-lists/kali.list.chroot >/dev/null
openssh-server
openvpn
kali-tools-top10
kali-linux-core
network-manager
mosh
python3-pip
```

*Be the animal you want to be and add `kali-linux-everything` for a massive ISO*

### Bootmenu

While I mostly left this section alone, feel free to customized bits like the Menu Label. The second file listed below will auto load into your option when the iso is booted from. So it will be rather fleeting.

```bash
cat <<'EOF' | sudo tee kali-config/common/includes.binary/isolinux/install.cfg >/dev/null
label install
    menu label ^Custom Unattended Install
    linux /install/vmlinuz
    initrd /install/initrd.gz
    append vga=788 -- quiet file=/cdrom/install/preseed.cfg locale=en_US keymap=us hostname=Vumetric-VM domain=local.lan
EOF

## Point to our boot menu
cat <<'EOF' | sudo tee kali-config/common/includes.binary/isolinux/isolinux.cfg >/dev/null
include menu.cfg
ui vesamenu.c32
default install
prompt 0
timeout 1
EOF
```

### Preseed.cfg

> "Preseeding provides a way to set answers to questions asked during the 
> installation process, without having to manually enter the answers while
>  the installation is running. This makes it possible to fully automate 
> most types of installation and even offers some features not available 
> during normal installations." This kind of customization is best 
> accomplished with *live-build* by placing the configuration in a preseed.cfg file included in config/includes.installer/.

[Debian.net Live Manual](https://live-team.pages.debian.net/live-manual/html/live-manual/customizing-installer.en.html)

This file will determine how the system will be installed, the inital network settings, and basic user creation.

In my scenario, I needed root to be enabled, have a predefined password, and a client user, that the client can log into but not with root access.

#### Users and Passwords

```
# Client User
d-i passwd/user-fullname string Client IT Department 
# d-i passwd/make-user boolean false #Use this if you don't want a user and just want root.
d-i passwd/username string setup 
d-i passwd/user-password password Nice21QuietPlace
d-i passwd/user-password-again password Nice21QuietPlace

# Root
d-i passwd/root-login boolean true # Allow for root to login
d-i passwd/root-password password SuperSecretPassword
d-i passwd/root-password-again password SuperSecretPassword
```

Aletnatively, you can preencrypt the passwords

```bash
echo "r00tme" | mkpasswd -s -H MD5
#$1$XzO3iZDo$dBrNFr3F6tF5NM.yI/sQu0
```

```
d-i passwd/root-password-crypted password $1$XzO3iZDo$dBrNFr3F6tF5NM.yI/sQu0
d-i passwd/user-password-crypted password $1$J3/7CXvA$eH2OFza/XFl6to7ar36rT1
```

Unfortunatly, you can only make the root and inital user at this stage.

More on [preseed.cfg]([B.4. Contents of the preconfiguration file](https://www.debian.org/releases/etch/i386/apbs04.html.en) here.

#### Partitioning

I mostly left this default. **BE WARNED, THIS WILL AUTO WIPE THE FIRST HARD DRIVE IT FINDS**. 

```
d-i partman-auto/method string regular
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-auto/choose_recipe select atomic
d-i partman-auto/disk string /dev/sda
d-i partman/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label boolean true
```

#### Hostname and Network

Besides hostname, I left this mostly alone.

```
d-i netcfg/get_hostname string Custom-VM
d-i netcfg/get_domain string unassigned-domain
#d-i netcfg/choose_interface select auto
d-i netcfg/choose_interface select eth0 
#Change line above if eth0 is not the default network interface name.
d-i netcfg/dhcp_timeout string 60
```

#### Localization

```
d-i debian-installer/locale string en_US.UTF-8
d-i console-keymaps-at/keymap select us
d-i keyboard-configuration/xkb-keymap select us
d-i mirror/http/proxy string
d-i mirror/suite string kali-rolling
d-i mirror/codename string kali-rolling
d-i clock-setup/utc boolean true
d-i time/zone string Etc/UTC
```

#### Misc, Grub, and Others

```
d-i apt-setup/use_mirror boolean true
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean false
d-i grub-installer/bootdev string /dev/sda
d-i finish-install/reboot_in_progress note
d-i hw-detect/load_firmware boolean false 
d-i apt-setup/use_mirror boolean false
kismet kismet/install-setuid boolean false
kismet kismet/install-users string
sslh sslh/inetd_or_standalone select standalone
mysql-server-5.5 mysql-server/root_password_again password
mysql-server-5.5 mysql-server/root_password password
mysql-server-5.5 mysql-server/error_setting_password error
mysql-server-5.5 mysql-server-5.5/postrm_remove_databases boolean false
mysql-server-5.5 mysql-server-5.5/start_on_boot boolean true
mysql-server-5.5 mysql-server-5.5/nis_warning note
mysql-server-5.5 mysql-server-5.5/really_downgrade boolean false
mysql-server-5.5 mysql-server/password_mismatch error
mysql-server-5.5 mysql-server/no_upgrade_when_using_ndb error
```

Final result looks like:

```bash
cat << EOF > kali-config/common/includes.installer/preseed.cfg
d-i passwd/user-fullname string Client IT Department 
d-i passwd/username string setup 
d-i passwd/user-password password Nice21QuietPlace
d-i passwd/user-password-again password Nice21QuietPlace
d-i passwd/root-login boolean true # Allow for root to login
d-i passwd/root-password password SuperSecretPassword
d-i passwd/root-password-again password SuperSecretPassword
d-i partman-auto/method string regular
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-auto/choose_recipe select atomic
d-i partman-auto/disk string /dev/sda
d-i partman/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label bo
d-i netcfg/get_hostname string Custom-VM
d-i netcfg/get_domain string unassigned-domain
d-i netcfg/choose_interface select eth0 
d-i netcfg/dhcp_timeout string 60
d-i debian-installer/locale string en_US.UTF-8
d-i console-keymaps-at/keymap select us
d-i keyboard-configuration/xkb-keymap select us
d-i mirror/http/proxy string
d-i mirror/suite string kali-rolling
d-i mirror/codename string kali-rolling
d-i clock-setup/utc boolean true
d-i time/zone string Etc/UTC
d-i apt-setup/use_mirror boolean true
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean false
d-i grub-installer/bootdev string /dev/sda
d-i finish-install/reboot_in_progress note
d-i hw-detect/load_firmware boolean false 
d-i apt-setup/use_mirror boolean false
kismet kismet/install-setuid boolean false
kismet kismet/install-users string
sslh sslh/inetd_or_standalone select standalone
mysql-server-5.5 mysql-server/root_password_again password
mysql-server-5.5 mysql-server/root_password password
mysql-server-5.5 mysql-server/error_setting_password error
mysql-server-5.5 mysql-server-5.5/postrm_remove_databases boolean false
mysql-server-5.5 mysql-server-5.5/start_on_boot boolean true
mysql-server-5.5 mysql-server-5.5/nis_warning note
mysql-server-5.5 mysql-server-5.5/really_downgrade boolean false
mysql-server-5.5 mysql-server/password_mismatch error
mysql-server-5.5 mysql-server/no_upgrade_when_using_ndb error
EOF
```

Once your preseed.cfg is something you are comforable with, it will need to be stored in `kali-config/common/includes.installer/preseed.cfg`. This should be located in the barebones template downloaded in step 1. Alternatively, you can put this is a shell script using `cat << EOF > kali-config/common/includes.installer/preseed.cfg` similar to the bash tempate we downloaded.

#### Build

Feel free to change the name. The tail end of the script that will actually build the ISO.

```bash
## Build image
time sudo ./build.sh \
  --debug \
  --live \
  --version OurCoolKali

## Output
ls -lh /opt/kali-build/images/*.iso

## Done
echo "[i] Done"
# exit 0
```

So, depending on everything done, building the ISO might talk a while but you will be able to monitor it the entire time.

At this point, you have a fully made, unattended ISO installer. But it doesnt do much. We can make it do stuff.

## Make it do stuff

So this is the section that caused me the most pain. Offensive Security was so inaccurate with the documentation, I had to do a lot of trial and error. Compiling the ISO and building a VM in virtualbox with it made this lengthy process.

#### Auto Start Services

We start off basic. Earlier, we installed `openssh-server`. In this example, we are going to have the ssh server automatically start on boot.

In Offensive Securitys example, they show the following command:

```
### DONT USE THIS. 
kali@kali:~$ echo 'systemctl enable ssh' >>  kali-config/common/hooks/01-start-ssh.chroot
kali@kali:~$ chmod +x kali-config/common/hooks/01-start-ssh.chroot
### I KNOW YOUR JUST LOOKING AT CODE BLOCKS
```

This is wrong for a couple of reasons.

First, the folder that this command goes into is wrong. The path is not correct. 

**Bad path:**

```
kali-config/common/hooks/
```

**Actual Path:**

```
kali-config/common/hooks/normal/
```

Second, the file name needs to have `.hook.chroot`  as the extention.

In my example, I turned:

```bash
### BAD DONT USE
echo 'systemctl enable ssh' >>  kali-config/common/hooks/01-start-ssh.chroot
chmod +x kali-config/common/hooks/01-start-ssh.chroot
### READ AND LEARN NOT TO DO IT THIS WAY.
```

Into:

```bash
mkdir -p kali-config/common/hooks/normal/
cat <<EOF > kali-config/common/hooks/normal/01-start-ssh.hook.chroot 
#!/bin/bash
systemctl enable ssh
systemctl start ssh
EOF
chmod +x kali-config/common/hooks/normal/01-start-ssh.hook.chroot
```

*It is important to make the chroot script executable with `chmod +x`*

### Transfering of files and configs

In my scenario, I needed to transfer an openVPN configuration that will be used to connect back to my openVPN server. If you need a quick way of setting up a vpn server and client config, use [this]([GitHub - angristan/openvpn-install: Set up your own OpenVPN server on Debian, Ubuntu, Fedora, CentOS or Arch Linux.](https://github.com/angristan/openvpn-install)). The only change I would advise you make is changing ``/etc/openvpn/server.conf`` to allow multiple connections using the same key/cert. As multiple computer will be using the file we are about to transfer over.




Live Build offer a very simple way of transfering files and configurations over. The `kali-config/common/includes.chroot/` directory directly translates to the root directory of a files system `/`.



For example, if you put a file in `kali-config/common/includes.chroot/opt/MenInBlack.txt`, the same file will be inserted into the file system when the OS is install at `/opt/MenInBlack.txt`. It is important that you make the folder in `kali-config/common/includes.chroot/` using `mkdir -p kali-config/common/includes.chroot/path/to/folder`.



In our example, we need to move our vpn config file to `/etc/openvpn/` with the ext `.conf`

```bash
mkdir -p kali-config/common/includes.chroot/etc/openvpn
cat <<EOF > kali-config/common/includes.chroot/etc/openvpn/openvpn.conf # Copy VPN Config file to new system 
client
proto tcp-client
remote remote.pentest.com 443
route-nopull 
route 10.8.0.0 255.255.255.0 #Add this to limit the routes going back to the VPN Server.
dev tun
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
verify-x509-name [...]
auth SHA256
auth-nocache
cipher AES-128-GCM
tls-client
tls-version-min 1.2
tls-cipher TLS-ECDHE-ECDSA-WITH-AES-128-GCM-SHA256
ignore-unknown-option block-outside-dns
setenv opt block-outside-dns # Prevent Windows 10 DNS leak
verb 3
<ca>
-----BEGIN CERTIFICATE-----
[...]
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
[...]
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
[...]
-----END PRIVATE KEY-----
</key>
<tls-crypt>
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
[...]
-----END OpenVPN Static key V1-----
</tls-crypt>
EOF
```

We need to make a small change to change to the `/etc/default/openvpn` file. While we are at it, we also will have the openvpn service start on boot. Once again, it is important to make the chroot script executable.

```bash
cat <<EOF > kali-config/common/hooks/normal/02-enable-openvpn.hook.chroot
#!/bin/bash
echo AUTOSTART="all" >> /etc/default/openvpn
systemctl enable openvpn.service
systemctl start openvpn.service
EOF
chmod +x kali-config/common/hooks/normal/02-enable-openvpn.hook.chroot
```

## Compile and Run

Your final script should look something like this.



```bash

## Make sure we have programs needed
sudo apt-get update
sudo apt-get install -yqq git live-build simple-cdd cdebootstrap curl

## Get base-image build-script
if [ -e "/opt/kali-build/" ]; then
  cd /opt/kali-build/
  sudo git reset --hard HEAD >/dev/null
  sudo git clean -f -d >/dev/null
  sudo git pull >/dev/null
else
  sudo git clone https://gitlab.com/kalilinux/build-scripts/live-build-config.git /opt/kali-build/
  cd /opt/kali-build/
fi


cat <<'EOF' | sudo tee kali-config/common/includes.binary/isolinux/install.cfg >/dev/null
label install
    menu label ^Custom Unattended Install
    linux /install/vmlinuz
    initrd /install/initrd.gz
    append vga=788 -- quiet file=/cdrom/install/preseed.cfg locale=en_US keymap=us hostname=Vumetric-VM domain=local.lan
EOF

## Point to our boot menu
cat <<'EOF' | sudo tee kali-config/common/includes.binary/isolinux/isolinux.cfg >/dev/null
include menu.cfg
ui vesamenu.c32
default install
prompt 0
timeout 1
EOF

## Install our packages

cat <<'EOF' | sudo tee kali-config/variant-default/package-lists/kali.list.chroot >/dev/null
openssh-server
openvpn
kali-tools-top10
kali-linux-core
network-manager
mosh
python3-pip
EOF

## Enable SSH
mkdir -p kali-config/common/hooks/normal/
cat <<EOF > kali-config/common/hooks/normal/01-start-ssh.hook.chroot 
#!/bin/bash
systemctl enable ssh
systemctl start ssh
EOF
chmod +x kali-config/common/hooks/normal/01-start-ssh.hook.chroot

## Config and Enable openVPN

mkdir -p kali-config/common/includes.chroot/etc/openvpn
cat <<EOF > kali-config/common/includes.chroot/etc/openvpn/openvpn.conf # Copy VPN Config file to new system 
client
proto tcp-client
remote remote.pentest.com 443
route-nopull 
route 10.8.0.0 255.255.255.0 #Add this to limit the routes going back to the VPN Server.
dev tun
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
verify-x509-name [...]
auth SHA256
auth-nocache
cipher AES-128-GCM
tls-client
tls-version-min 1.2
tls-cipher TLS-ECDHE-ECDSA-WITH-AES-128-GCM-SHA256
ignore-unknown-option block-outside-dns
setenv opt block-outside-dns # Prevent Windows 10 DNS leak
verb 3
<ca>
-----BEGIN CERTIFICATE-----
[...]
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
[...]
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
[...]
-----END PRIVATE KEY-----
</key>
<tls-crypt>
#
# 2048 bit OpenVPN static key
#
-----BEGIN OpenVPN Static key V1-----
[...]
-----END OpenVPN Static key V1-----
</tls-crypt>
EOF


cat <<EOF > kali-config/common/hooks/normal/02-enable-openvpn.hook.chroot
#!/bin/bash
echo AUTOSTART="all" >> /etc/default/openvpn
systemctl enable openvpn.service
systemctl start openvpn.service
EOF
chmod +x kali-config/common/hooks/normal/02-enable-openvpn.hook.chroot


## PreSeed File

cat << EOF > kali-config/common/includes.installer/preseed.cfg
d-i passwd/user-fullname string Client IT Department 
d-i passwd/username string setup 
d-i passwd/user-password password Nice21QuietPlace
d-i passwd/user-password-again password Nice21QuietPlace
d-i passwd/root-login boolean true # Allow for root to login
d-i passwd/root-password password SuperSecretPassword
d-i passwd/root-password-again password SuperSecretPassword
d-i partman-auto/method string regular
d-i partman-lvm/device_remove_lvm boolean true
d-i partman-md/device_remove_md boolean true
d-i partman-lvm/confirm boolean true
d-i partman-auto/choose_recipe select atomic
d-i partman-auto/disk string /dev/sda
d-i partman/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
d-i partman-partitioning/confirm_write_new_label bo
d-i netcfg/get_hostname string Custom-VM
d-i netcfg/get_domain string unassigned-domain
d-i netcfg/choose_interface select eth0 
d-i netcfg/dhcp_timeout string 60
d-i debian-installer/locale string en_US.UTF-8
d-i console-keymaps-at/keymap select us
d-i keyboard-configuration/xkb-keymap select us
d-i mirror/http/proxy string
d-i mirror/suite string kali-rolling
d-i mirror/codename string kali-rolling
d-i clock-setup/utc boolean true
d-i time/zone string Etc/UTC
d-i apt-setup/use_mirror boolean true
d-i grub-installer/only_debian boolean true
d-i grub-installer/with_other_os boolean false
d-i grub-installer/bootdev string /dev/sda
d-i finish-install/reboot_in_progress note
d-i hw-detect/load_firmware boolean false 
d-i apt-setup/use_mirror boolean false
kismet kismet/install-setuid boolean false
kismet kismet/install-users string
sslh sslh/inetd_or_standalone select standalone
mysql-server-5.5 mysql-server/root_password_again password
mysql-server-5.5 mysql-server/root_password password
mysql-server-5.5 mysql-server/error_setting_password error
mysql-server-5.5 mysql-server-5.5/postrm_remove_databases boolean false
mysql-server-5.5 mysql-server-5.5/start_on_boot boolean true
mysql-server-5.5 mysql-server-5.5/nis_warning note
mysql-server-5.5 mysql-server-5.5/really_downgrade boolean false
mysql-server-5.5 mysql-server/password_mismatch error
mysql-server-5.5 mysql-server/no_upgrade_when_using_ndb error
EOF


## Build image
time sudo ./build.sh \
  --debug \
  --live \
  --version MyNiceCustomVM

## Output
ls -lh /opt/kali-build/images/*.iso

## Done
echo "[i] Done"
# exit 0




```

The iso should be located in `/opt/kali-build/images/`

Hopefully, when booting from this iso, it will take care of all the additional steps needed, and will automatically call back to your openvpn server. Minimal involvment from the client, supports across most Hyper Visors.



## Final Notes:

Offensive Security.... You know what you did. I mostly did this out of spite. Fix your docs. Don't let the people experice the frustration. 



If you have issues, feel free to reach out on twitter and I will see if I can help.

@gray_sec



----



## Knowledge, Terms, and commands

```bash
mkdir -p /path/to/folder
```

Make Folder and parent folders if they dont exist. If the folders do exists, no error is returned and the command contiunes.

![](assets/2022-03-07-15-44-47-image.png)

```bash
cat <<EOF > newfile.txt
some
multi
line
text
EOF
```

Multi Line Text to Output. Useful for adding and moving configurations without having seperate files.
