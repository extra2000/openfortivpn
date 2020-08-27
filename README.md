# openfortivpn

| License | Versioning | Build |
| ------- | ---------- | ----- |
| [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT) | [![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release) | [![Build Status](https://travis-ci.com/extra2000/openfortivpn.svg?branch=master)](https://travis-ci.com/extra2000/openfortivpn) [![Build status](https://ci.appveyor.com/api/projects/status/al6ub31jyhnd83jb/branch/master?svg=true)](https://ci.appveyor.com/project/nikAizuddin/openfortivpn/branch/master) |

Automate OpenFortiVPN deployment.


## Getting started

Clone this repository and `cd` into the project:
```
$ git clone git@github.com:extra2000/openfortivpn.git
$ cd openfortivpn
```


## Prepare certificates

Copy your `.crt` and `.key` PEM certificates into `salt/roots/formulas/openfortivpn-formula/openfortivpn/files/secrets/`.

If you are using `.p12` certificate, OpenFortiVPN doesn't support such certificate. You need to convert `.p12` certificate into PEM certificates:
```
$ openssl pkcs12 -in mycert.p12 -out mycert.crt -clcerts -nokeys
$ openssl pkcs12 -in mycert.p12 -out mycert.key -nocerts -nodes
```

Find out your server certificate digest using the following command. You need the certificate digest for `openfortivpn.server.certdigest` later in `salt/roots/pillar/openfortivpn.sls` file (Section [Prepare SaltStack pillar for OpenFortiVPN](#Prepare-SaltStack-pillar-for-OpenFortiVPN)). Replace the `${SERVER}:${PORT}` with your own server, for example `172.1.1.1:443`:
```
$ echo -n | openssl s_client -connect ${SERVER}:${PORT} | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > /tmp/myfortigate.cert
$ openssl x509 -in /tmp/myfortigate.cert -fingerprint -sha256 -noout | awk '{print $2}' | gawk 'match($0, /=(.*)/, a) {print a[1]}' | tr -d ':' | awk '{print tolower($0)}'
```


## Prepare SaltStack pillar for OpenFortiVPN

Create `salt/roots/pillar/openfortivpn.sls` file as shown in example below. Use your own `certificate` and `server`:
```
openfortivpn:
  build:
    username: vagrant
  certificate:
    crtfile: mycert.crt
    keyfile: mycert.key
  server:
    host: 172.1.1.1
    port: 443
    certdigest: 36d91cf360b272163273f12dab5a66806895cf4328c32f29b4ddb35cdab89fb9
```


## Deployment

This Section will describe how to deploy OpenFortiVPN using SaltStack for both on localhost and on production server. For localhost deployment, you can use Vagrant.


### Localhost deployment using Vagrant

Install Vagrant and virtual machine such as Libvirt or VirtualBox. However it is recommended to use Libvirt instead of VirtualBox.

Create `openfortivpn-ubuntu1804` Vagrant box:
```
$ vagrant up --provider=libvirt openfortivpn-ubuntu1804
```

You may change `--provider=libvirt` to other provider such as `--provider=virtualbox` or `--provider=hyperv`, but this is experimental.

SSH into the box:
```
$ vagrant ssh openfortivpn-ubuntu1804
```

Install OpenFortiVPN client and deploy using the following commands:
```
$ sudo salt-call state.sls openfortivpn
```

To stop and disable OpenFortiVPN client service:
```
$ sudo salt-call state.sls openfortivpn.clean
```

To uninstall openfortivpn VPN client and clean files:
```
$ sudo salt-call state.sls openfortivpn.clean
```


### Production deployment

Install SaltStack:
```
$ curl -o bootstrap-salt.sh -L https://bootstrap.saltstack.com
$ sudo sh bootstrap-salt.sh -x python3 -P -c /tmp git v3001.1
```

Clone [openfortivpn-formula](https://github.com/extra2000/openfortivpn-formula) repository into `/srv/formulas/`:
```
$ mkdir /srv/formulas/
$ cd /srv/formulas/
$ git clone --recursive https://github.com/extra2000/openfortivpn-formula.git
```

Make sure the `/srv/formulas/openfortivpn-formula` is added into your `/etc/salt/master` or `/etc/salt/minion` for masterless. Note that any changes in `/etc/salt/master` requires `$ sudo systemctl restart salt-master.service`:
```
file_roots:
  base:
    - /srv/salt
    - /srv/formulas/openfortivpn-formula
```

Prepare your `crtfile` and `keyfile` PEM certificates as described in Section [Prepare certificates](#Prepare-certificates). Then your `crtfile` and `keyfile` PEM certificates into `/srv/formulas/openfortivpn-formula/openfortivpn/files/secrets/`.

Create your SaltStack pillar file for the OpenFortiVPN as described in Section [Prepare SaltStack pillar for OpenFortiVPN](#Prepare-SaltStack-pillar-for-OpenFortiVPN), but into `/srv/pillar/openfortivpn.sls`. Then add the `openfortivpn.sls` into `/srv/pillar/top.sls`:
```
base:
  '*':
    - openfortivpn
```

Note that any changes to pillar also requires `$ sudo systemctl restart salt-master.service`, unless you are running on masterless.

Install and deploy OpenFortiVPN via SaltStack master. Replace `MINION-ID` with your minion's ID:
```
$ sudo salt 'MINION-ID' state.sls openfortivpn
```

Or execute `salt-call` on minion if you are using masterless:
```
$ sudo salt-call state.sls openfortivpn
```


## Allow host to use VPN connections from Vagrant box

Suppose that your Vagrant box IP address is `192.168.121.2`, execute the following commands on the minion:
```
$ sudo iptables -t nat -A POSTROUTING -s 192.168.121.2 -j MASQUERADE
$ sudo sysctl -w net.ipv4.ip_forward=1
```

Let's say if you want to SSH to `172.168.100.64` host, execute the following `ip route` command on host:
```
$ sudo ip route add 172.168.0.0/16 via 192.168.121.2
```


## Allow other devices to use VPN connection

Suppose that your minion's LAN IP address is `192.168.1.2` and the other device that you want to give VPN access is `192.168.1.10`

On the minion, execute the following command:
```
$ sudo iptables -t nat -A POSTROUTING -s 192.168.1.10 -j MASQUERADE
$ sudo sysctl -w net.ipv4.ip_forward=1
```

Let's say if you want to SSH to `172.168.100.64` from the other device, execute the same `ip route` command as above on the device:
```
$ sudo ip route add 172.168.0.0/16 via 192.168.1.2
```
