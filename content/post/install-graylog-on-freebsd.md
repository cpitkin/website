---
title: "Install Graylog on Freebsd 10.3"
date: 2017-08-01
tags: [freebsd, logging, infra]
draft: false
---
I have always needed a central logging server at home. After setting up a couple for my day job I finally decided to setup one for my home servers. This guide is meant to serve as a way install Graylog into FreeBSD. The FreeBSD server is a jail running in FreeNAS 9.10. We are looking to install Graylog 1.3.3 with the web interface. I know Graylog has moved to 2.0 but as of now, the FreeBSD packages are on version 1.3.3. This setup has been working on my server for 15 days so far with an input of 3-6 msgs a second. This guide assumes you already have access to the machine via ssh. [Graylog]: (https://www.graylog.org)

Nano is my editor of choice in the terminal so all the examples will use it. To install (it is not installed by default) run `pkg install nano`. Feel free to use the editor of your choice.

First, we will want to configure and start ntpd. This will make sure we have an accurate time for indexes and logs flowing into the system. **NOTE:** If you are running this in a jail you need to setup the ntpd service on the host not in the jail.

First lets enable the service.
`echo "ntpd_enable="YES" >> /etc/rc.conf`

Then we need to add some time servers. I always use the time servers from [pool.ntp.org](http://www.pool.ntp.org/en/)

Check `/etc/ntpd.conf`. If you have the following lines in your file then you don't need to do anything.  

```
server 0.freebsd.pool.ntp.org iburst                                            server 1.freebsd.pool.ntp.org iburst                                            server 2.freebsd.pool.ntp.org iburst                                            #server 3.freebsd.pool.ntp.org iburst
```

If you do not have the lines above add the lines below to `/etc/ntpd.conf`

```
cat <<EOF >> /etc/ntp.conf
server 0.north-america.pool.ntp.org
server 1.north-america.pool.ntp.org
server 2.north-america.pool.ntp.org
server 3.north-america.pool.ntp.org
EOF
```

Now start the service.
`service ntpd start`

### Install
Now we need to install the graylog and its dependancies.
`pkg install -y mongodb elasticsearch graylog graylog-web-interface pwgen`

Copy the following and run it to make sure all the needed services are started on boot.

```
cat <<EOF >> /etc/rc.conf
# Graylog with dependancies
graylog_enable="YES"
graylog_web_interface_enable="YES"
elasticsearch_enable="YES"
mongod_enable="YES"
EOF
```

### Elastic Search Configs

`pw groupmod elasticsearch -m graylog`
`nano /usr/local/etc/elasticsearch/elasticsearch.yml`

  - network.host: 127.0.0.1

### Mongo DB Configs

First lets start Mongo.
`service mongod start`

Open the mongo configuration file.
`nano /usr/local/etc/mongodb.conf`

Add the following to the file and save.
```
net:
  port: 27017
  bindIp: 127.0.0.1
  security:   
    authorization: enabled
```

Start the mongo interactive shell
`mongo`

Switch to the admin user
`use graylog2`

Create a user with the rw role to the database graylog2. Graylog creates the graylog2 database so do not replace this value.

```
db.createUser({
  user: "graylog",
  pwd: "MY_SUPER_SECRET_PASSWORD",
  roles: [
    {
      role: "readWrite\",
      db:"graylog2"
    }
  ]  
})
```
Exit the shell
`exit`

Lets restart mongo to load in the new configurations.
`service mongod restart`

### Graylog Server Configs

Change directory into graylog then copy the example configuration to the the graylong.conf (which is not there by default). Next we can touch node-id which we will need in a minute.
`cd /usr/local/etc/graylog && cp graylog.conf.example graylog.conf && touch node-id && chown graylog:graylog node-id`

Open the config file
`nano /usr/local/etc/graylog/graylog.conf`

All of the configation options below are in different parts of the file and are commented as needed.
```
node_id_file = /usr/local/etc/graylog/node-id
#Generate one by using for example: pwgen -N 1 -s 96\npassword_secret =
root_username = admin

#echo -n yourpassword | shasum -a 256
root_password_sha2 = hash from the above command

root_email ="YOU_EMAIL_HERE"
graylog2-server.uris="http://<your-server-ip>:12900//"

rest_enable_gzip = true
rest_enable_tls = true

elasticsearch_cluster_name = elasticsearch
elasticsearch_discovery_zen_ping_multicast_enabled = false
elasticsearch_discovery_zen_ping_unicast_hosts =         127.0.0.1:9300
elasticsearch_max_time_per_index = 1w

#mongodb_uri = mongodb://localhost/graylog2
mongodb_uri = mongodb://graylog:MY_SUPER_SECRET_PASSWORD@localhost:27017/graylog2
```

Start the service
`service graylog start`

#### Graylog Web Interface Configs

Change directory and edit the graylog web interface configuration file
`cd .. && nano graylog-web-interface.conf`

```
graylog2-server.uris="https://127.0.0.1:12900/"
# Generate for example with: pwgen -N 1 -s 96
application.secret=""

timezone="America/Chicago"

# This needs to be enabled to allow for a self-signed certificate. If you're using a valid certificate this can be set to false.
graylog2.client.accept-any-certificate=true
```

Start the web interface
`service graylog_web_interface start`

Open your browser and go to `http://HOSTNAME:9000`

I hope this gives you some help with configuring your own central log server.
