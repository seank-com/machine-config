

# Raspbian Configuration

Configuration steps, scripts and tools i use for Raspbian on RaspberryPi platforms. Feel free to skip any steps that don't work for you or add steps if you think something is missing.

_Note:_ Adafruit has an excellent [tutorial](https://learn.adafruit.com/node-embedded-development/connecting-via-ssh) with lots of helpful info. Likewise, Dave has some [excellent info](http://thisdavej.com/beginners-guide-to-installing-node-js-on-a-raspberry-pi/) as well.

## Configure Raspbian

- Open a terminal window and run the following command ```sudo raspi-config```
- Select 7 Advanced Options | A1 Expand Filesystem
- Select 3 Boot options | B1 Desktop / CLI | B1 Console
- Select 4 Localisation Options | I1 Locale
  - then de-select en_GB.UTF-8 and select en_US.UTF-8 instead
- Select 4 Localisation Options | I2 Change Timezone | US | Pacific Ocean
- Select 7 Advanced Options | A3 Memory Split | 0
- Select 5 Interfacing Options | P2 SSH | Yes
- Selete 5 Interfacing Options | P4 I2C | Yes
- Select Finish and reboot

## Configure ssh

See steps for creating public key file in [Linux](./Linux.md) if you don't already have one.
Then from a terminal window on your dev machine run the following.

```bash
$ scp ~/.ssh/server-access-public-key.pem pi@<ip>:~/server-access-public-key.pem
```

Now ssh to the pi and complete with the folowing commands

```bash
$ mkdir ~/.ssh
$ chmod 700 ~/.ssh
$ ssh-keygen -i -m PKCS8 -f ~/server-access-public-key.pem >~/.ssh/authorized_keys
$ rm ~/server-access-public-key.pem
```

on dev machine add the following lines to ~/.ssh/config

```
Host raspberry
  User pi
  HostName <ip>
  IdentityFile ~/.ssh/server-access-key.pem
```

Now you should be able to ssh into your server from your development machine using the following command with getting a password prompt

```bash
$ ssh raspberry
```

More importantly assuming you have a directory on your dev machine named ~/Development/sshfs-mount you can run the following command to mount the pi's file system

```bash
$ sshfs raspberry:/ ~/Development/sshfs-mount/
```
If the mac loses the connect, reset with the following

```bash
$ pgrep -lf sshfs
$ kill <pid_of_sshfs_process>
$ sudo umount -f <mounted_dir>
$ sshfs raspberry:/ ~/Development/sshfs-mount/
```

## i2c tools

```bash
$ sudo apt-get install -y i2c-tools
```

## configure I2C

```bash
$ sudo nano /etc/rc.local
```

add the following line just before the ```exit 0```

```
echo -n 1 > /sys/module/i2c_bcm2708/parameters/combinedpreview
```

## Setting up Kubernetes

Here are the steps, but it will likely not work. I ended up using [this guide](https://blog.hypriot.com/post/setup-kubernetes-raspberry-pi-cluster/) instead.

```bash
$ sudo nano /boot/cmdline.txt
```

add ```cgroup_enable=memory cgroup_memory=1``` before ```rootwait``` and save

```bash
# disable swap
$ sudo chmod -x /etc/init.d/dphys-swapfile
$ sudo swapoff -a
$ sudo rm /var/swap

# install docker
$ sudo curl -sSL https://get.docker.com | sh
$ sudo usermod -aG docker pi
$ sudo /etc/init.d/docker start
$ docker run arm32v7/hello-world

# install kubeadm
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ cat <<EOF >kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
$ sudo mv kubernetes.list /etc/apt/sources.list.d/
$ sudo apt-get update
$ sudo apt-get install -y policykit-1 kubelet kubeadm kubectl
$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet

# run on master
$ sudo kubeadm init --pod-network-cidr=192.168.1.0/24
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ curl -sSL https://rawgit.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml | sed "s/amd64/arm/g" >kube-flannel.yml
$ kubectl apply -f kube-flannel.yml

# the kubeadm init above should print a command like the following to run on each of the worker nodes
$ kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>

$ kubeadm join --token e15d07.86cd214c6d788319 192.168.1.100:6443 --discovery-token-ca-cert-hash sha256:398d1b62a4d0c6f652482b4ac3a0c4234369c09147620aac58e59e0e9de43b56
```

## Setting up a Node server

SSH and run the following commands to install node and nginx

```bash
$ sudo apt-get remove -y nodered
$ sudo apt-get remove -y nodejs nodejs-legacy
$ curl -sL https://deb.nodesource.com/setup_4.x | sudo -E bash -
$ sudo apt-get update
$ sudo apt-get dist-upgrade
$ sudo apt-get install -y nodejs
$ sudo apt-get install -y git
$ sudo apt-get install -y nginx
$ sudo apt-get install -y xrdp
$ sudo apt-get autoremove
$ sudo /etc/init.d/nginx stop
```

Now setup the node server

```bash
$ sudo su -
$ useradd node
$ exit
$ sudo mkdir /var/node
$ sudo mkdir /var/node/logs
$ sudo chown -R $(whoami):node /var/node
$ sudo chmod 775 /var/node/logs
$ sudo mkdir /var/forever
$ sudo chown $(whoami):node /var/forever
$ sudo chmod 775 /var/forever
$ sudo mkdir /var/azure-event-hubs
$ sudo chown -R $(whoami):node /var/azure-event-hubs
$ git clone https://github.com/seank-com/azure-event-hubs.git /var/azure-event-hubs
$ cd /var/azure-event-hubs
$ git checkout develop
$ git pull
```

Getting it ready to run forever

```bash
$ sudo npm install forever -g
$ nano /var/node/forever.json
```

Paste the following for /etc/init.d/iot-server

```
{
  "uid": "iot-server",
  "append": true,
  "script": "app.js",
  "path": "/var/forever",
  "sourceDir": "/var/node",
  "workingDir": "/var/node",
  "killSignal": "SIGTERM",
  "logFile": "/var/node/logs/forever.log",
  "outFile": "/var/node/logs/out.log",
  "errFile": "/var/node/logs/err.log"
}
```

Now make it a service

```bash
$ sudo touch /etc/init.d/iot-server
$ sudo chmod 755 /etc/init.d/iot-server
$ sudo nano /etc/init.d/iot-server
```

Paste the following for /etc/init.d/iot-server

```
### BEGIN INIT INFO
# Provides:             iot-server
# Required-Start:
# Required-Stop:
# Default-Start:        2 3 4 5
# Default-Stop:         0 1 6
# Short-Description:    IoT Server Node App
### END INIT INFO

case "$1" in
  start)
    sudo su node -c 'FOREVER_ROOT=/var/forever /usr/bin/forever start /var/node/forever.json'
    ;;
  stop)
    sudo su node -c 'FOREVER_ROOT=/var/forever /usr/bin/forever stop iot-server'
    ;;
  *)

  echo "Usage: /etc/init.d/iot-server {start|stop}"
  exit 1
  ;;
esac
exit 0
```

*Note:* if you ever need to list the forever process running you should run the following.
```bash
sudo su node -c 'FOREVER_ROOT=/var/forever /usr/bin/forever list'
```

Now configure logrotate to handle the logs

```bash
$ sudo nano /etc/logrotate.d/iot-server
```

Paste the following

```
/var/node/logs/*.log {
  daily
  missingok
  size 100k
  rotate 7
  notifempty
  su pi node
  create 0764 pi node
  nomail
  sharedscripts
  prerotate
    sudo /etc/init.d/iot-server stop >/dev/null 2>&1
  endscript
  postrotate
    sudo /etc/init.d/iot-server start >/dev/null 2>&1
  endscript
}
```

*Note:* if you want to test/force rotation run the following
```bash
sudo logrotate -f /etc/logrotate.conf
```

Now configure Nginx

```bash
$ sudo nano /etc/nginx/sites-enabled/default
```

Replace the contents of /etc/nginx/sites-enabled/default with
the following

```
##
# You should look at the following URL's in order to grasp a solid understanding
# of Nginx configuration files in order to fully unleash the power of Nginx.
# http://wiki.nginx.org/Pitfalls
# http://wiki.nginx.org/QuickStart
# http://wiki.nginx.org/Configuration
#
# Please see /usr/share/doc/nginx-doc/examples/ for more detailed examples.
##

# HTTP server
server {
  listen 		80 default;
  server_name 	iot-server;

  # Proxy pass-though to the local node server
  location / {
    proxy_pass http://127.0.0.1:4000/;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_connect_timeout       300;
    proxy_send_timeout          300;
    proxy_read_timeout          300;
    send_timeout                300;
  }
}
```

Register and start services

```bash
$ sudo update-rc.d iot-server defaults
$ sudo /etc/init.d/iot-server start
$ sudo /etc/init.d/nginx start
```

http://weworkweplay.com/play/automatically-connect-a-raspberry-pi-to-a-wifi-network/
