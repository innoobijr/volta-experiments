# FRR Installation Instructions

## Installing Dependencies
```
sudo apt update
sudo apt-get install \
   git autoconf automake libtool make libreadline-dev texinfo \
   pkg-config libpam0g-dev libjson-c-dev bison flex \
   libc-ares-dev python3-dev python3-sphinx \
   install-info build-essential libsnmp-dev perl \
   libcap-dev libelf-dev libunwind-dev \
   protobuf-c-compiler libprotobuf-c-dev
```
### Source Install

Ensure that the libyang build requirements are met before continuing. Usually this entails installing cmake and libpcre2-dev or pcre2-devel.

```
git clone https://github.com/CESNET/libyang.git
cd libyang
git checkout v2.1.128
mkdir build; cd build
cmake --install-prefix /usr \
      -D CMAKE_BUILD_TYPE:String="Release" ..
make
sudo make install
```

## GRPC

If GRPC is enabled using --enable-grpc the following packages should be installed.
```
sudo apt-get install libgrpc++-dev protobuf-compiler-grpc
```

## Config Rollbacks

If config rollbacks are enabled using --enable-config-rollbacks the sqlite3 developer package also should be installed.

```
sudo apt install libsqlite3-dev
```
## Building & Installing FRR
### Add FRR user and groups

```
sudo groupadd -r -g 92 frr
sudo groupadd -r -g 85 frrvty
sudo adduser --system --ingroup frr --home /var/run/frr/ \
   --gecos "FRR suite" --shell /sbin/nologin frr
sudo usermod -a -G frrvty frr
```


## Compile
Clone the FRR git repo and use the included configure script to configure FRR’s build time options to your liking. The full option listing can be obtained by running ./configure -h. The options shown below are examples.

```git clone https://github.com/frrouting/frr.git frr
cd frr
./bootstrap.sh
./configure \
    --prefix=/usr \
    --includedir=\${prefix}/include \
    --bindir=\${prefix}/bin \
    --sbindir=\${prefix}/lib/frr \
    --libdir=\${prefix}/lib/frr \
    --libexecdir=\${prefix}/lib/frr \
    --sysconfdir=/etc \
    --localstatedir=/var \
    --with-moduledir=\${prefix}/lib/frr/modules \
    --enable-configfile-mask=0640 \
    --enable-logfile-mask=0640 \
    --enable-snmp \
    --enable-multipath=64 \
    --enable-user=frr \
    --enable-group=frr \
    --enable-vty-group=frrvty \
    --with-pkg-git-version \
    --with-pkg-extra-version=-MyOwnFRRVersion
make
sudo make install
```

## Install FRR configuration files
```
sudo install -m 775 -o frr -g frr -d /var/log/frr
sudo install -m 775 -o frr -g frrvty -d /etc/frr
sudo install -m 640 -o frr -g frrvty tools/etc/frr/vtysh.conf /etc/frr/vtysh.conf
sudo install -m 640 -o frr -g frr tools/etc/frr/frr.conf /etc/frr/frr.conf
sudo install -m 640 -o frr -g frr tools/etc/frr/daemons.conf /etc/frr/daemons.conf
sudo install -m 640 -o frr -g frr tools/etc/frr/daemons /etc/frr/daemons
sudo install -m 640 -o frr -g frr tools/etc/frr/support_bundle_commands.conf /etc/frr/support_bundle_commands.conf
```

## Tweak sysctls

Some sysctls need to be changed in order to enable IPv4/IPv6 forwarding and MPLS (if supported by your platform). If your platform does not support MPLS, skip the MPLS related configuration in this section.

Edit /etc/sysctl.conf and uncomment the following values (ignore the other settings):
```
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
net.ipv6.conf.all.forwarding=1
```
Reboot or use sysctl -p to apply the same config to the running system.

## Add MPLS kernel modules

Ubuntu 20.04 ships with kernel 5.4; MPLS modules are present by default. To enable, add the following lines to /etc/modules-load.d/modules.conf:

```
# Load MPLS Kernel Modules
mpls_router
mpls_iptunnel
```

And load the kernel modules on the running system:

```
sudo modprobe mpls-router mpls-iptunnel
```

If the above command returns an error, you may need to install the appropriate or latest linux-modules-extra-<kernel-version>-generic package. For example apt-get install linux-modules-extra-`uname -r`-generic

## Enable MPLS Forwarding

Edit /etc/sysctl.conf and the following lines. Make sure to add a line equal to net.mpls.conf.eth0.input for each interface used with MPLS.

```
# Enable MPLS Label processing on all interfaces
net.mpls.conf.eth0.input=1
net.mpls.conf.eth1.input=1
net.mpls.conf.eth2.input=1
net.mpls.platform_labels=100000
```

## Install service files
```
sudo install -m 644 tools/frr.service /etc/systemd/system/frr.service
sudo systemctl enable frr
```

## Enable daemons
Open /etc/frr/daemons with your text editor of choice. Look for the section with watchfrr_enable=... and zebra=... etc. Enable the daemons as required by changing the value to yes.

## Start FRR
```
systemctl start frr
```


