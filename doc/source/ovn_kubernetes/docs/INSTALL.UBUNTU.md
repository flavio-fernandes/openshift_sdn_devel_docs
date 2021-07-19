# Installing OVS and OVN on Ubuntu

## Installing OVS and OVN from packages

For testing and POCs, we maintain OVS and OVN packages in a AWS VM.
(For production, it is recommended that you build your own OVS packages.)
To install packages from there, you can run:

```
sudo apt-get install apt-transport-https
echo "deb http://3.19.28.122/openvswitch/stable /" |  sudo tee /etc/apt/sources.list.d/openvswitch.list
wget -O - http://3.19.28.122/openvswitch/keyFile |  sudo apt-key add -
sudo apt-get update
```

To install OVS bits on all nodes, run:

```
sudo apt-get build-dep dkms
sudo apt-get install python-six openssl -y

sudo apt-get install openvswitch-datapath-dkms -y
sudo apt-get install openvswitch-switch openvswitch-common -y
```

On the master, where you intend to start OVN's central components,
run:

```
sudo apt-get install ovn-central ovn-common ovn-host -y
```

On the agent nodes, run:

```
sudo apt-get install ovn-host ovn-common -y
```

## Installing OVS and OVN from sources

Install a few pre-requisite packages.

```
sudo apt-get update
sudo apt-get install -y build-essential fakeroot debhelper \
                        autoconf automake libssl-dev \
                        openssl python-all \
                        python-setuptools \
                        python-six \
                        libtool git dh-autoreconf \
                        linux-headers-$(uname -r)
```

Clone the OVN repo.
**Note:** Starting on OVN [version v21.03.0](https://github.com/ovn-org/ovn/commit/9ea1f092d2ed6b90bc8c444a2c73f4e0fceebeff),
it is recommended to obtain the compatible OVS via git submodule, as shown below.

```
git clone https://github.com/ovn-org/ovn.git
cd ovn

# not recommended option: git clone https://github.com/openvswitch/ovs.git
git submodule update --init  ; # recommended option
pushd ./ovs
```

Configure and compile the OVS sources.

```
./boot.sh
./configure --prefix=/usr --localstatedir=/var  --sysconfdir=/etc --enable-ssl --with-linux=/lib/modules/`uname -r`/build
make -j3
```

Install the OVS executables.

```
sudo make install
sudo make modules_install
```

Create a depmod.d file to use OVS kernel modules from this repo instead of
upstream linux.

```
cat << EOF | sudo tee /etc/depmod.d/openvswitch.conf
override openvswitch * extra
override vport-geneve * extra
override vport-stt * extra
override vport-* * extra
EOF
```

Copy a startup script and start OVS.

```
sudo depmod -a
sudo cp debian/openvswitch-switch.init /etc/init.d/openvswitch-switch
sudo /etc/init.d/openvswitch-switch force-reload-kmod
```

Configure and compile the OVN sources.

```
popd  ; # back to ovn directory
./boot.sh
./configure --prefix=/usr --localstatedir=/var  --sysconfdir=/etc --enable-ssl --with-ovs-source=${PWD}/ovs
make -j3
```

Install the OVN executables.

```
sudo make install
```

Copy startup scripts and start OVN.

```
sudo cp -v ./debian/ovn-central.init /etc/init.d/ovn-central
sudo cp -v ./debian/ovn-host.init /etc/init.d/ovn-host
sudo ln -s /usr/share/openvswitch/scripts/ovs-lib /usr/share/ovn/scripts/

# Reference commands for starting central and host locally
sudo /etc/init.d/ovn-central start
sudo /etc/init.d/ovn-host start

sudo ovn-nbctl set-connection ptcp:6641
sudo ovn-sbctl set-connection ptcp:6642
sudo ovs-vsctl set open . external_ids:system-id=chassis1 \
     external_ids:ovn-remote=tcp:127.0.0.1:6642 \
     external_ids:ovn-encap-type=geneve external_ids:ovn-encap-ip=127.0.0.1
```
