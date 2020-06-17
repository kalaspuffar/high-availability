## high-availability

This is a small example of how to configure a high availability Linux system.

In this repository, I've gathered some commands to get a multiple server setup with failover.

### General setup. This should be done on both machines.

First, we need to install a heartbeat and drbd for monitoring and replication.
```
apt install heartbeat drbd-utils
```

Next up, we will configure heartbeat, so it knows about our hosts. This can be done in the ```/etc/ha.d/ha.cf``` just update the IP addresses and node names to fit your environment.
```
logfacility     local0
autojoin none
warntime 5
deadtime 15
initdead 60
keepalive 2

ucast enp0s3 192.168.6.19
ucast enp0s3 192.168.6.20

auto_failback on

node vm1
node vm2
```

For all the servers to communicate, they need to have a known secret, so no one else can hijack the conversation. This can be set up in the ```/etc/ha.d/authkeys``` using a simple sha hash. In this example, we use a simple sha1, in your setup, a more extensive code might be the best option.
```
auth1
1 sha1 d1e6557e2fcb30ff8d4d3ae65b50345fa46a2faa
```

The keys are sensitive information so we need to secure them by changing the access rights so only the root user may access it.
```
chmod 600 /etc/ha.d/authkeys
```

Lastly, in the heartbeat configuration, we need to tell the system what services it should keep track of. In this setup, we stay that vm1 is the primary host, and we want it to have the shared drive running, then the host is available.
/etc/ha.d/haresources
```
vm1 shared-drive
```

We will use a drbd set up to have a distributed replicated block device that can hold the data for both machines. To have a mount point for that, we create a directory on each machine.
```
mkdir -p /sharefs
```

Next, we create the service that the heartbeat system will monitor and keep alive on the machine currently in use. In this case, it will be implemented in the ```/etc/init.d/shared-drive``` script.
```
#!/bin/bash

param=$1

if [ "start" == "$param" ] ; then
  drbdadm primary test  
  mount /dev/drbd0 /sharefs
  exit 0
elif [ "stop" == "$param" ] ; then
  umount /sharefs
  drbdadm secondary test  
  exit 0;
elif [ "status" == "$param" ] ; then
  exit 0;
else
  echo "no such command $param"
  exit 1;
fi
```

Now we need to make the script runnable, and for security, we can lock it down so only the superuser can run it.
```
chmod 700 /etc/init.d/shared-drive
```

Next up, we configure the drbd service, and the global configuration is located in ```/etc/drbd.d/global_common.conf``` before you replace this file, make a copy or move it so you can replace it with the code below.
```
global {
    usage-count  yes;
}
common {
    net {
        protocol C;
    }
}
```
This script will count the usage of this installation, sending information to the maintainers. Not required, but it is a nice thing to do. We will also configure to use the protocol C. This is for close network setups as our installation here. The service can be used for replication over longer distances and far networks. In that case, the synchronous or asynchronous methods A and B can be used.


Now we will configure the resource that we will keep synced between the systems. You can manage many resources, so let's create one we call test in the file ```/etc/drbd.d/test.res```.
```
resource test {
    device /dev/drbd0;
    disk /dev/sda2;
    meta-disk internal; 
    on vm1 {
        address 192.168.6.12:7789;
    }
    on vm2 {
        address 192.168.6.13:7789;
    }
}
```
Above, we specify the new block device name of /dev/drbd0 and the disk we want to use in both systems for syncing. I have the same disk name on both so I can just say /dev/sda1. You may move this configuration down to any of the machine-specific configurations. We will keep the meta-disk information internal to this device. Not really sure why you would want to split it up or what the implications would be. But this is the simplest of setups.

Now we will create the block device so we can mount it later.
```
drbdadm create-md test
```

Next, we bring up the service.
```
drbdadm up test
```

After creating and bringing up the service, we should see the block device and also the status of the service. So far, the disk between systems is not in sync yet.
```
lsblk
drbdadm status test
```

### Specific setup for vm1

In this machine-specific setup, I will set the first VM as my primary VM, where all the data will be written. Then if we ever switch over and make this secondary, it will take the data produced by the primary node and keep it in sync.
Here we will force this node to be primary and then watch the status to ensure that it is up and running.
```
drbdadm primary --force test
drbdadm status test
```

When the block device is created, we need to create a new file area. We use ext4 here as it's a modern journaling system that works well for this purpose.
```
mkfs -t ext4 /dev/drbd0 
```

### Specific setup for vm2

On the second node, you need to define that it is now a secondary system, and the syncing will start. We can follow the progress using the status command.
```
drbdadm secondary test
drbdadm status test
```

### Testing the system.

Now you want to test that everything works. If the secondary system goes offline, nothing will happen, but hopefully, you will have some monitoring that can bring that system up again. But this will not inflict any downtown.

On the other hand, if the primary system goes down or if the heartbeat service shuts off. Then the secondary node will notice this and bring up the shared drive on that system. 

If you ```tail -f /var/log/messages``` on the second VM and then run ```service heartbeat stop``` on the first VM, you can watch the handover. Depending on how high the load you have on your service, you might want to tweak the configuration parameters. This setup will handover the information after 2 seconds, but you might require a quicker handover.


### Installing munin node

For each node only add the munin node package.
```
apt install munin-node
```

Next you need to add the node configuration for each node in munin-node.conf
```
allow ^192\.168\.6\.20$
host_name vm1
host 192.168.6.20
```

```
allow ^192\.168\.6\.20$
host_name vm2
host 192.168.6.19
```

We need to restart the munin node in order for the configuration to take place.
```
/etc/init.d/munin-node restart
```

### Installing munin collection server

For the munin server you need to add both the apache, node and munin package.

```
apt install munin apache2
```

On the munin server you need to define which hosts are allowed in munin.conf
```
[vm1]
    address 192.168.6.20
[vm2]
    address 192.168.6.19
```

Configure apache.
```
<VirtualHost *:80>
    ServerName vm1.ea.org
    ServerAlias vm1

    ServerAdmin  info@example.org

    DocumentRoot /var/www

    Alias /munin/static/ /etc/munin/static/
    <Directory /etc/munin/static>
        Require all granted
    </Directory>

    Alias /munin /var/cache/munin/www
    <Directory /var/cache/munin/www>
        Require all granted
    </Directory>

    CustomLog /var/log/apache2/munin.example.org-access.log combined
    ErrorLog  /var/log/apache2/munin.example.org-error.log
</VirtualHost>
```

Restart services.
```
/etc/init.d/apache2 restart
/etc/init.d/cron restart
/etc/init.d/munin restart
```

### Adding extra plugins

First we need to add our plugin to the plugins directory and make it executable.
```
cd /usr/share/munin/plugins
vi drbd
chmod +x drbd
```

Then we need to create a link in the configuration directory in order to enable the plugin.
```
cd /etc/munin/plugins
ln -s /usr/share/munin/plugins/drbd
```

Next up we need to uptade the configuration file ```/etc/munin/plugin-conf.d/munin-node``` adding permissions for our plugin.
```
[drbd]
user root
```

Lastly we restart the node for the configuration to take hold.
```
/etc/init.d/munin-node restart
```