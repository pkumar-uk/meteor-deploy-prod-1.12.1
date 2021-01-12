# Introduction
This is a write up to help deploy a Meteor application to production. 

This guide is for those who probably want to do the setup themselves, or probably have a specific need. It can be used to write your own script to automate stuff when this is working for you.  

Following assumptions are made or it assumes that this your setup(but it can easily be used for any Meteor setup with small change)
* You have a Meteor app on version 1.12.1 ( version is important as each meteor version is locked to a node version)
* Meteor app using Redis oplog, so will need to set it up as well. We will use the Application server to host Redis server as well.
* Two Ubuntu 20.04 servers  one will act as Application server and another will act as a database server. Hopefully both in the same datacenter. 
* Domain for your app
* nginx as proxy server amd load balancer
* pm2 for running the node app and can be extended for monitoring




# Preparing your server
## Manage Domain

You will need to set a 'A' record to point your domain to your application server. For this you should do the portal of the domain provider. Under Manage Domain your will need to add a 'A' record with the domain or subdomain pointing to the application server IP. 


## Setting up  appid

Login to your server as root using password or key(if you had setup during vps creation) xxx.xx.xx.xxx being your ip address
```
ssh root@xxx.xx.xx.xxx
```
If everything is good, you should be now logged into the server.

Since root has lot of priviliges let us create userid **appid** on both servers which we will use for our implementation

Lot of information is available on the net as well. 



let us add the user now 

Note - you can replace **appid** with any name you prefer for now let us use in this note as **appid**. You will be asked some questions, fill it up and give **Y** at the end to confirm.

```
adduser stppeify
```
Now you need the newly added **appid** to sudo group to give admin privileges when needed. Use the following command
```
usermod -aG sudo appid
```

Let us setup the firewall rules before we exit, to make our servers secure. For now we will set the firewall to allow only ssh access, for this we will use **ufw** that comes with ubuntu.

We can see the available list of rules in ufw by typing
```
ufw app list
```

you should see 
```
OpenSSH
```

let us add that as a rule to ufw
```
ufw allow OpenSSH
```
having added this rule, now let us enable the firewall - by typing at prompt
```
ufw enable
```
confirm, do not be afraid. This will now secure the server.

So far we have done using root. Let us logout and check if our new userid **appid** works 

```
ssh appid@xxx.xx.xx.xxx
```

if everything is good you should be logged in now. Do that for the other server as well. The steps following this will be done using the **appid** as userid.

# Setting up Database Server
MongoDB is going to be our database. Meteor 1.12.1 is compatible with MongoDB 4.*. As of the date of writing MongoDB 4.4 is the latest vesion so we will use it. Most of the installtion instructions you will see is from installation instruction from https://docs.mongodb.com/manual/installation/ .
We will setup mongodb in only one server and not as a traditional replica set of servers. But to use it with meteor it needs to be set for replication to enable oplog.

Log in to your designated database server using **appid**
```
ssh appid@xxx.xx.xx.xxx
```
Execte the following commands to create the source list for MongoDB packages
```
wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
```
once it finishes downloading
```
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
```
now let us update the local packages
```
sudo apt-get update
```
If everything completes, it is time to install mongodb. Execute the following command
```
sudo apt-get install -y mongodb-org
```
we will use the default setup of mongodb where we will find he log file at
**/var/log/mongodb** and data file at **/var/lib/mongodb**. Let us start the mongodb service
```
sudo systemctl start mongod.service
```
you can check if it has come up correctly by issuing the following or checking the logs at path mentioned few lines above
```
sudo systemctl status mongod
```
if the mongodb service has come up, it will show something like this
```
● mongod.service - MongoDB Database Server
     Loaded: loaded (/lib/systemd/system/mongod.service; disabled; vendor prese>
     Active: active (running) since Sat 2021-01-09 18:04:22 IST; 2min 59s ago
       Docs: https://docs.mongodb.org/manual
   Main PID: 4269 (mongod)
     Memory: 59.6M
     CGroup: /system.slice/mongod.service
             └─4269 /usr/bin/mongod --config /etc/mongod.conf

Jan 09 18:04:22 e2e-73-206 systemd[1]: Started MongoDB Database Server.
```
You can check if mongodb has come up by going to mongo shell by typing
```
mongo
```
if it is working you will see mongo shell prompt. It will also show a message to enable free monitoring if you are interested, follow the steps.

Let us enable the mongodb service to start at boot. Execute the following 
```
sudo systemctl enable mongod
```

Now we will do things to make it fit for purpose for meteor. Let us set it for oplog.

First we need to edit the mongo config file. You have to change mongod.conf which is in /etc folder for this.

Take a backup of mongod.conf before you edit the file.

use any of your favourite editor to edit /etc/mongod.conf file. we will use nano
```
sudo nano /etc/mongod.conf 
```


go to line which shows **#replication:** you should replace it with below lines and save (you need to be careful with editing they are position dependent). 
```
replication:
  replSetName: meteor
```


now restart mongo service by issuing
```
sudo systemctl restart mongod
```
if you had edited your mongod.conf file correctly, you should be able to login to the mongo shell.

Log on to the mongo shell by typing
```
mongo
```
on the mongo shell prompt type the following command to setup variable for replication config
```
var config = {_id: "meteor", members: [{_id: 0, host: "127.0.0.1:27017"}]}
``` 
then type the following to use the variable to configure replication
```
rs.initiate(config)
```
if it is accepted you will see an out something similar to this
```
{
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1610201164, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1610201164, 1)
}
```
now exit the mongo shell and restart the mongo service to enable replication.
```
sudo systemctl restart mongod
```
let us now check if the replication is up. For this again login to mongo shell by issuing the following
```
mongo
```
at shell prompt issue
```
rs.status()
```
it should print information about replication. Now we are set for using oplog. Exit the mongo shell.

Since our meteor application is going to be on another server, to access database server we need to bind the server ip otherwise our application server will not be able to access mongodb as by default mongodb binds to 127.0.0.1 

for this we have to edit again the mongod.conf file ..execute
```
sudo nano /etc/mongod.conf 
```
go to line that has
```
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1
```
change it to 
```
# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1,xxx.xx.xx.xxx
```
here xxx.xx.xx.xxx is the database server ip

Note - you can use private ip if you want your server to be more secure. 

Now restart the mongod service to enable binding ip
```
sudo systemctl restart mongod
```

Now we are almost set, but our initial firewall rules will prevent accessing the server so we need to change the firewall to allow application server to access our database server. if **aaa.aa.aa.aaa** is the ip of application server, issue the following (mongodb listens on port 27017 by default)
```
udo ufw allow from aaa.aa.aa.aaa/24  to any port 27017
```

Now we have our database setup.

Note - Our setup of mongodb does not set userid for accessing it. So it is important to have the firewall set correctly. It can be made more secure by setting up userid for accessing. Refer to mongodb documentation.

If you are migrating your app you may need to dump the data from old mongodb using mongodump and restore it using mongorestore.

It is a good idea to back up your database data. One of the way you can do is to use mongodump to take snapshot of the data at set times and load the dumped data to a secure place. Here is a sample script that you can use. You can schedule this script using crontab 

```
#!/bin/sh
now="$(date +'%d_%m_%Y_%H_%M_%S')"
filename="mongo_db_backup_$now".gz
backupfolder="dump"
fullpathbackupfile="$MongoBackups"
logfile="mongodump"_"$(date +'%Y-%m-%d-%H-%M-%S')".tar.gz
cd $home
rm -rf dump
mongodump

touch "$logfile"
tar -czvf "$logfile" dump

s3cmd put "$logfile" s3://bucketname
rm "$logfile"

```
the script takes a snapshot of mongodb, zips the data to a file with timestamp and then tries to load to a S3 compatible storage. 



# Setting up Application Server

For the application server we need to setup quite a few things 
* Setting up redis
* Setting up nginx
* setting up nodejs
* Setting up letsencrypt
* Setting up pm2



after this we need to config and bring up our application
* Setting up Meteor App
* Preparing pm2.json file
* Preparing nginx

## Setting up redis

We will use the default redis package of ubuntu. 

First let us update the package list
```
sudo apt update
```
then install redis server by executing
```
sudo apt install redis-server
```
Now let us configure redis to use systemd to start. For this you need to change redis.conf. We will use nano to edit
```
sudo nano /etc/redis/redis.conf
```
go to the line showing **supervised no** and change it to **supervised systemd** 

It should look like this after edit
```
# If you run Redis from upstart or systemd, Redis can interact with your
# supervision tree. Options:
#   supervised no      - no supervision interaction
#   supervised upstart - signal upstart by putting Redis into SIGSTOP mode
#   supervised systemd - signal systemd by writing READY=1 to $NOTIFY_SOCKET
#   supervised auto    - detect upstart or systemd method based on
#                        UPSTART_JOB or NOTIFY_SOCKET environment variables
# Note: these supervision methods only signal "process is ready."
#       They do not enable continuous liveness pings back to your supervisor.
supervised systemd

```

Now restart redis service 
```
sudo systemctl restart redis.service
```

you can check the status of the redis service
```
sudo systemctl status redis.service
```
you should see something like this as an output if redis is running fine
```
● redis-server.service - Advanced key-value store
     Loaded: loaded (/lib/systemd/system/redis-server.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2021-01-09 20:29:57 IST; 28s ago
       Docs: http://redis.io/documentation,
             man:redis-server(1)
    Process: 4069 ExecStart=/usr/bin/redis-server /etc/redis/redis.conf (code=exited, status=0/SUCCESS)
   Main PID: 4086 (redis-server)
      Tasks: 4 (limit: 14007)
     Memory: 1.9M
     CGroup: /system.slice/redis-server.service
             └─4086 /usr/bin/redis-server 127.0.0.1:6379
```

You can further check if redis is working by using redis-cli. type
```
redis-cli
```
you will get a redis-cli prompt. Now type 
```
ping
```
if everything is fine you should get a response back as
```
PONG
```
That should be good for our needs for redis-oplog for our meteor app.

## Setting up nginx
We will use the default nginx package for our needs from ubuntu. Which ideally should be the latest.

Install nginx by using the following command
```
sudo apt install nginx
```
We have to change our firewal rules to allow http and https access ( Currently setup to access only port 22). This can be done by
```
sudo ufw allow 'Nginx Full'
```
Note: 'Nginx Full' is rules configured for nginx. You can check he other rules available by using **sudo ufw app list**
You can also check if the firewall is set correctly by using **sudo ufw status**

We need to configure the dhparam this is Diffie-Hellman key to help in secure communication with server and clients. (openssl is used here and it should come with ubuntu vps, if not you may need to install it). We will create the key in **/etc/nginx** folder.
```
sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096
```
This key will be used in nginx config file for the domain.

We need to configure nginx further, which we will do later.

## Install nodejs
We will use nvm to install nodejs. 

First let us set up nvm. Execute command
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash
```
to start using nvm first you have to execute
```
source ~/.bashrc
```
now let us check if nvm is setup correctly. type
```
nvm list-remote
```
if setup of nvm is correct, you will see a list of node versions available to install.

We know that each meteor vesion requires a specific nodejs version. Meteor 1.12.1 requires nodejs version v12.20.1 ( you can check the nodejs version for your meteor app by going to your project folder on your development machine and executing **meteor node -v**)

Let us install nodejs v12.20.1
```
nvm install v12.20.1
```
This will install nodejs. You can check if it is installed by checking
```
node -v
```

Now we are setup with node and npm, let us go and install pm2
## Install pm2 
We will use pm2 to manage running of our node app. It allows to start multiple instances of our application very easily that can be helpful to scale up.

Install pm2
```
npm install pm2@latest -g
```
This will do the setup of pm2. You can test if it has etup by typing
```
pm2
```
You should see pm2 status.

We will come back for further configuration of pm2 later.

## Other install

My meteor app also does image processing on the server which requires imagemagick to be present. So let us install it as well. 

```
sudo apt install imagemagick
```

## Setting up the meteor app

We will setup meteor app in **/opt** folder on the application server. 

This folder needs to have the right permission for **applid** to use it. So let us first give it the correct access for this we will use setfacl command. This requires some setup
```
sudo apt install acl
```
now give the permission to **applid**
```
sudo setfacl -m u:applid:rwx /opt
```
now **applid** can read write to **/opt** folder on application server

Let us build the meteor app in our development machine for deployment to the Application server. So go to your development laptop or machine. Since our application server is ubuntu/linux based we need to ensure that correct build parameter is given. This is to be done in your meteor app project folder. Execute the following command replacing

bld-yourapp - the output folder where you want to store the build file.
https://yourdomain - with the domain name you have for your app

```
meteor build ../bld-yourapp --architecture os.linux.x86_64 --server=https://yourdomain
```
once this command is run it will build a file in the **bld-yourapp** with the format **yourapp.tar.gz** assuming **yourapp** is your project folder name.

now copy this file to your application server. If you are in the **bld-yourapp** folder you can issue the following command(replace it with correct build file name and ip address)

```
scp yourapp.tr.gz applid@aaa.aa.aa.aaa:/opt
```
this should copy the file to the /opt folder on the application server. 

Now go back to your application server and go to the **/opt** folder. 

Now you need to build it for the app server again. For this you will need to do the next few steps. Assuming you are in **/opt** folder, first unpack the loaded build file **youapp.tar.gz**
```
tar -xvzf yourapp.tar.gz
```
on successful completion it will create a folder under **/opt** named **bundle**

to build go to the server folder inside the bundle folder
```
cd bundle/program/server
```
once inside the server folder isuue 
```
npm install --production
```

now go back to **/opt** folder and rename the bundle to **appname** (this allows for new checkins)
```
mv bundle appname
``` 

Now our app is ready to be run. For this we will use pm2. Pm2 requires a json file to be given as a parameter so let us build one. A sample is given below you will need to customise it for your need. Create this file in your **applid** folder which is **/home/applid** and let us name it as **pm2.json**

```
{
  "apps": [
    {
      "name": "appname",
      "cwd": "/opt/appname",
      "script": "main.js",
      "instances":1,
      "env": {
        "NODE_ENV": "production",
        "WORKER_ID": "0",
        "PORT": "3000",
        
        "ROOT_URL": "https://yourdomain",
        "MONGO_URL": "mongodb://xxx.xx.xx.xxx:27017/appnamedb",
        "MONGO_OPLOG_URL": "mongodb://xxx.xx.xx.xxx:27017/local",
        "HTTP_FORWARDED_COUNT": "1",
        "MAIL_URL": "",
        "METEOR_SETTINGS": {
  	"aaaa":"aaaval",
          -------------
          "redisOplog": {
            "redis": {
              "port": 6379, 
              "host": "127.0.0.1"  
            },
            "retryIntervalMs": 10000, 
            "mutationDefaults": {
                "optimistic": false,  
                "pushToRedis": true  
            },
            "debug": true
          }
        }
      }
    }

  ]
}
```

You will have to carefully create the pm2.json file. Hopefully if you have your app running in development machine this is going to be very simple. Some hints
* replace **appname** with the a unique name you want to give to your application. This is shown when enquiring pm2 for status
* replace **/opt/appname** with the name you have given to the bundle folder earlier
* instances declares the number of copies of nodejs app you want to run, for now it can be one but can be increased. pm2 will automatically increment the port number starting from the number mentioned for the next instance.
* **https://yourdomain** with your root url i.e, your domain name as given also during your build time.
* **mongodb://xxx.xx.xx.xxx:27017/appnamedb** and **mongodb://xxx.xx.xx.xxx:27017/local** with xxx.xx.xx.xxx with database server ip. **appnamedb** can be a name given for your database name for your app (you can keep it as meteor if you want to be similar to development, but use unique if you want to host multiple apps using the same mongodb instance)
* if you have a MAIL_URL set it up here. Refer meteor documents
* Define your METEOR_SETTINGS using the settings.json file used in development. You will notice in the example the redis oplog part is defined, others you will need to create.
* if you are hosting more than one app you can create another instance of this.

If you have setup the PM2 file now you can invoke your node app. This is done by following command from the folder where **pm2.json** is created -- assuming **/home/applid**
```
pm2 start pm2.json all
```
if your pm2 json is correct it should have invoded your program - node app, and it will give you that information while invoking.

The meteor node app will bind to port defined on the pm2.json file -- 3000 in our case. Though the app is up, it is not accessible from outside as the firewall will prevent access to port 3000. To test you can set rules for ufw (refer what we did for database server) to open up your app. But do not forget to remove the rule after test. In the next step we will setup nginx to open the app for HTTP/HTTPS access using your domain name.

to enable pm2 to start on restart you can save the config
```
pm2 save
``` 
refer pm2 site for further documentation and features of it -- like restart, kill, status, logs, etc.

By default pm2 will store the logs in the .pm2 folder of user home. You can also view the logs using
```
pm2 logs
```
monitor your app by using
```
pm2 monit
```
status of the apps by using
```
pm2 status
```

## Certificates for Domain using Letsencrypt

We already have installed letsencrypt. If you have pointed your domain(as mentioned earlier) to your application server, you should be able to get the certificates required for SSL for your domain. To check if you have pointed correctly try pinging your domain and it should return your Application IP address.

Now next step is to get the certificate using
```
sudo certbot certonly -d domain-name  
```
You can use the 1st option presented and answer the default questions, letsencrypt will generate the certificate and store it in **/etc/letsencrypt/live/**.

Now with the certificate available we can define nginx config for the domain.

## Configure nginx

create a file **yourdomainname** using the template below replacing as mentioned below in **/etc/nginx/sites-available**

```
upstream appname {
  	hash $remote_addr consistent;
  	server 127.0.0.1:3000 fail_timeout=0;
  	server 127.0.0.1:3001 fail_timeout=0;
	 
}

map $http_upgrade $connection_upgrade {
  	default upgrade;
  	''      close;
}

server {
	listen 80;
	server_name yourdomainname;
      	return 301 https://$server_name$request_uri?;
}
server {
	listen 443 ssl http2;
  	server_name yourdomainname;
	ssl on;
	ssl_certificate /etc/letsencrypt/live/yourdomainname/fullchain.pem;
  	ssl_certificate_key /etc/letsencrypt/live/yourdomainname/privkey.pem;
  	ssl_dhparam /etc/nginx/dhparam.pem;
  	ssl_session_cache shared:SSL:10m;
  	ssl_session_timeout 10m;
  	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  	ssl_ciphers "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:
ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:A
ES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
	ssl_prefer_server_ciphers on;
	ssl_session_cache shared:SSL:10m;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains";
	add_header X-Frame-Options DENY;
	add_header X-Content-Type-Options nosniff;
	ssl_stapling on;
	ssl_stapling_verify on;
	resolver 8.8.8.8 8.8.4.4;
	# end of SSL block
	error_log /var/log/nginx/appname.log;
	## performance boost using gzip
	gzip on;
	gzip_disable "msie6";
	gzip_vary on;
	gzip_proxied any;
	gzip_comp_level 6;
	gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
	# end of GZIP block
location / {
        #resolve using Google's DNS server to force DNS resolution and prevent caching of IPs
        resolver 8.8.8.8;
        set $proxy_host $host;
		proxy_redirect off;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header Host $http_host;
		# Make sure to use WebSockets
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
        proxy_pass http://appname; # the name used in upstreams, substituted for any of the defined instances
    }
}

```
**upstream** block declares the instances of the apps initiated by pm2. Nginx loadbalances across the servers mentioned in this block. Make sure you list the ones you want the connections to be load balanced. Since application server and nginx is on the same server the application servers can be referenced by 127.0.0.1, but you can point to application server running on another server as lond as it can be accessed. 

Replace **yourdomanname** with the application domain name.

**appname** with your choice of appname

You may have to use sudo to create the config file. For eg
```
sudo nano /etc/nginx/sites-available/yourdomainname
```
Once the congifg file is created a symbolic link is to be created to sites-enabled
```
sudo ln -s /etc/nginx/sites-available/yourdomainname /etc/nginx/sites-enabled/yourdomainname
```

Once this is done we can now restart nginx for the configuration to take effect.
```
sudo systemctl restart nginx
```
If the config is correct, nginx should come up correctly. Now you should be able to access your application from any broswer.

Good luck with implementing this.......


