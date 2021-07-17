# HA FTP Server

### Environment
We use 3 servers, and each server:
| System(Env.) Parameters  | Values |
| :-------------: | :-------------: |
| OS  | Ubuntu 20.04.2  |
| RAM  | 2 GB  |
| CPU  | 2 Cores  |
| Hard Disk  | 20 GB  |

---
### Prerequisite
| Package Name  | Version |
| :-------------: | :-------------: |
| [vsftpd](https://security.appspot.com/vsftpd.html) | 3.0.3  |
| [pacemaker](https://github.com/ClusterLabs/pacemaker) | 2.0.3  |
| [corosync](https://github.com/corosync/corosync) | 3.0.3  |
| [crmsh](https://crmsh.github.io/) | 4.2.0  |
| [haveged](https://github.com/jirka-h/haveged) | 1.9.1  |
---
### Networking

| Server Name | IP | Domain | Description | 
| :-------------: | :-------------: | :-------------: | :-------------: |
| Server1 | 192.168.153.101  | noode1 | First Node |
| Server2 | 192.168.153.102  | noode1 | Second Node |
| Server3 | 192.168.153.103  | noode1 | Third Node | 
| - | 192.168.153.200 | ftp.server | Floating IP |
---
### Install FTP Server

Let’s begin by updating the package lists and installing vsftpd on Ubuntu 20.04

```
$ sudo apt update && sudo apt install vsftpd
```

Once installed, check the status of vsftpd

```
$ sudo service vsftpd status
```
```
● vsftpd.service - vsftpd FTP server
     Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2021-07-17 16:33:47 UTC; 2h 3min ago
   Main PID: 1594 (vsftpd)
      Tasks: 1 (limit: 2245)
     Memory: 3.0M
     CGroup: /system.slice/vsftpd.service
             └─1594 /usr/sbin/vsftpd /etc/vsftpd.conf

Jul 17 16:33:47 server3 systemd[1]: Starting vsftpd FTP server...
Jul 17 16:33:47 server3 systemd[1]: Started vsftpd FTP server.
```

### Create FTP User
Create FTP User.
```
$ sudo adduser ftpuser
```

### Configure vsftpd
Rename the config file.
```
$ sudo mv /etc/vsftpd.conf /etc/vsftpd.conf.bak
```
Create new config file.
```
$ vim /etc/vsftpd.conf
```
Paste in the following
```
listen=NO
listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
force_dot_files=YES
pasv_min_port=40000
pasv_max_port=50000
allow_writeable_chroot=YES
```

Restart vsftpd.
```
$ sudo systemctl restart vsftpd
```

---

### Make it HA

##### Mapping the Hosts File

Edit the  "/etc/hosts" file on all servers and add domains.
```
$ vim /etc/hosts
```
Paste in the following
```
192.168.153.101 node1
192.168.153.102 node2
192.168.153.103 node3
192.168.153.200 ftp.server
```

##### Install Pacemaker, Corosync, and Crmsh

Install Pacemaker, Corosync, and crmsh with the apt command below.
```
$ install -y pacemaker corosync crmsh
```
After the installation, all these services are running automatically on the system. Stop them with the systemctl commands below.
```
$ systemctl stop corosync
$ systemctl stop pacemaker
```

##### Configure Corosync
> Note that these configurations should only be run on one server, e.g. Server1.

Install haveged for better random numbers
```
$ apt install -y haveged
```
Now generate a new Corosync key with the command below.
```
$ corosync-keygen
```
When the key generation is complete, you can see the new key 'authkey' in the '/etc/corosync/' directory.
```
$ ls -lah /etc/corosync/
```

Next, go to the '/etc/corosync' directory and backup the default configuration file 'corosync.conf'.
```
$ cd /etc/corosync/
$ mv corosync.conf corosync.conf.bekup
```
Then create a new 'corosync.conf' configuration file.
```
$ vim corosync.conf
```
Paste in the following
```
# Totem Protocol Configuration
totem {
  version: 2
  cluster_name: ftpserver
  transport: udpu

# Interface configuration for Corosync
  interface {
    ringnumber: 0
    bindnetaddr: 192.168.153.0
    broadcast: yes
    mcastport: 5407
  }
}

# Nodelist - Server List
nodelist {
  node {
    ring0_addr: node1
  }
  node {
    ring0_addr: node2
  }
  node {
    ring0_addr: node3
  }
}

# Quorum configuration
quorum {
  provider: corosync_votequorum
}

# Corosync Log configuration
logging {
  to_logfile: yes
  logfile: /var/log/corosync/corosync.log
  to_syslog: yes
  timestamp: on
}

service {
  name: pacemaker
  ver: 0
}

```
Next, copy the authentication key and the configuration file from 'Server1' server to 'Server2' and 'Server3' server.
```
$ scp /etc/corosync/* root@node2:/etc/corosync/
$ scp /etc/corosync/* root@node3:/etc/corosync/
```
##### Start All Cluster Services
Start the HA cluster software stack, pacemaker and corosync, on all servers. Then enable it to start automatically at boot time with these commands.

```
$ systemctl start corosync
$ systemctl enable corosync
$ systemctl start pacemaker
$ update-rc.d pacemaker defaults 20 01
$ systemctl enable pacemaker
```
All services have been started, check all nodes and make sure the server status is 'Online' on all of them with
```
$ crm status
```

##### Create and Configure the Cluster
> Note that these configurations should only be run on one server, e.g. Server1.

Since we are not using a STONITH device, we want to disable STONITH and ignore the Quorum policy on our cluster.
```
$ crm configure property stonith-enabled=false
$ crm configure property no-quorum-policy=ignore
```

Next, we need to create some new resources for the cluster.

Create a new 'virtual_ip' resource for the floating IP configuration with the crm command below.

```
$ crm configure primitive vip ocf:heartbeat:IPaddr2 params ip="192.168.153.200" cidr_netmask="24" op start interval="0s" timeout="60s" op monitor interval="5s" timeout="20s" op stop interval="0s" timeout="60s"
```

And for the vsftpd 'vsftpd', create the resource with the command below.

```
$ crm configure primitive vsftpd lsb:vsftpd op start interval="0s" timeout="60s" op monitor interval="5s" timeout="20s"  op stop interval="0s" timeout="60s"
```
When this is done, check the new resources 'virtual_ip' and 'vsftpd' with the command below. Make sure all resources have the status 'started'.
```
$ crm resource status
```

We created already the Floating IP and the Service, now add those resources to a new group named 'ftpserver' with the command below.
```
$ crm configure group ftpserver vip vsftpd
```

A new group of resources with the name 'ftpserver' has been defined. You can check it with the command below.
```
$ crm resource show
```
You will get a group named ftpserver with members 'virtual_ip' and 'webserver' resources.

##### Testing

Testing node status and the cluster status.
```
$ crm status
```
We've 3 Nodes with status 'Online'.

---
### Test HA

To test our HA FTP Server, first we download a file from our floating  IP:
```
$ curl -u ftpuser:amir 'ftp://ftpuser@192.168.153.200/DevOps%20Application%20Task.pdf' -o /mnt/c/Users/Amir/Desktop/test.pdf
```
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  283k  100  283k    0     0  11.0M      0 --:--:-- --:--:-- --:--:-- 11.0M
```

Then turn off both servers and then download the file again
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  283k  100  283k    0     0  18.4M      0 --:--:-- --:--:-- --:--:-- 18.4M
```

As you can see, there is no problem downloading the file.


























