ELK (Elasticsearch, Logstash and Kibana)
===

All about my ELK configurations + Nxlog as a log forwarder

The Elasticsearch, Logstash, Kibana (ELK) stack has become very popular recently for cheap and easy centralized logging. We will go over the installation of Elasticsearch, Logstash and Kibana, and how to configure them to gather and visualize the CFEngine outputs of my systems in a centralized location.
* Logstash is an open source tool for collecting, parsing, and storing logs for future use. 
* Kibana is a web interface that can be used to search and view the logs that Logstash has indexed. 
* Both of these tools are based on Elasticsearch. Elasticsearch, Logstash, and Kibana, when used together is known as an ELK stack.

It is possible to use Logstash to gather logs of all types, but we will limit the scope of this tutorial to CFEngine related log gathering. For a log forwarder, we are going to use NXlog instead of Logstash Forwarder. NXlog is an open source log management tool available at no cost. Why? you would better follow this page for a clear veiw. :-) http://nxlog.org/products/nxlog-community-edition/features

I'm setting ELK up on a fresh install of Ubuntu Server 14.04 LTS. virtual machine with 2GB memory and 2 CPUs.

## Let's get started!

### Install Logstash and Elasticsearch
**Note**: Logstash 1.4.2 recommends Elasticsearch 1.1.1

First, I'm adding the repos of Elasticsearch and Logstash. Run the following command to import Elasticsearch public GPG key into apt:
```
wget -O - http://packages.elasticsearch.org/GPG-KEY-elasticsearch | sudo apt-key add -
```
Edit `/etc/apt/sources.list` and add the Elasticsearch and Logstash repositories to the end of the file:
```
deb http://packages.elasticsearch.org/elasticsearch/1.1/debian stable main  
deb http://packages.elasticsearch.org/logstash/1.4/debian stable main
```
Update your apt package database:
```
sudo apt-get update
```
Install Logstash this command:
```
sudo apt-get -y install logstash
```
*(Logstash and Elasticsearch require Java. It would take some times depending on how fast your internet is.)*

Then install Elasticsearch 1.1.1
```
sudo apt-get -y install elasticsearch=1.1.1
```
Elasticsearch is now installed. Let's start the process:
```
sudo service elasticsearch restart
```
and run the following command to start Elasticsearch on boot up:
```
sudo update-rc.d elasticsearch defaults 95 10
```
Now that Elasticsearch is up and running. A quick test to make sure that Elasticsearch was installed correctly and works. To do that, run `curl http://localhost:9200`. Curl should output something like:
```
{
  "status" : 200,
  "name" : "Rl'nnd",
  "version" : {
    "number" : "1.1.1",
    "build_hash" : "f1585f096d3f3985e73456debdc1a0745f512bbc",
    "build_timestamp" : "2014-04-16T14:27:12Z",
    "build_snapshot" : false,
    "lucene_version" : "4.7"
  },
  "tagline" : "You Know, for Search"
}
```
Let's install Kibana

### Install Kibana (+ Nginx as a web server)

**Note**: Logstash 1.4.2 recommends Kibana 3.1.2

Download Kibana to /tmp directory with the following command:
```
cd /tmp; wget https://download.elasticsearch.org/kibana/kibana/kibana-3.1.2.tar.gz
```
Extract Kiban tarball with tar:
```
tar xvfz kibana-3.1.2.tar.gz
```
We will be using Nginx to serve our Kibana installation, so let's make it more standadisation to where web-related stuff should be located:
```
sudo mkdir -p /var/www/kibana3
```
Now copy the Kibana files into the directory:
```
sudo cp -R /tmp/kibana-3.1.2/* /var/www/kibana3/
```
Last, but definitely not least, before we can use the Kibana web interface, we have to install Nginx. So:
```
sudo apt-get -y install nginx
```
We better should ensure ownership of Kibana to `www-data` as well:
```
sudo chown -R www-data:www-data /var/www
```
### Install Nxlog

Download packages from here.
* [Nxlog Community Edition](http://nxlog.org/products/nxlog-community-edition/download)

We have to install the dependencies first. To list the dependencies, use the following command:
```
sudo dpkg-deb -f nxlog_1.4.581_amd64.deb Depends
```
Then make sure all listed dependencies are installed. Alternatively you can run `apt-get install -f` after trying
to install the package with dpkg and getting an error due to the missing dependencies.

To install the deb package, issue the following command as root:
```
sudo dpkg -i nxlog_1.4.581_amd64.deb
```
To install the rpm package with the following command:
```
rpm –ivh nxlog-1.4.581-1.x86_64.rpm
```
or can be do with YUM
```
yum localinstall nxlog-1.4.581-1.x86_64.rpm
```
After the package is installed check and edit the configuration file located at `/etc/nxlog/nxlog.conf`. It contains an example configuration which you will likely want to modify to suit your needs.


## Quick overview of important locations for files
### Elasticsearch:
* Binaries and stuff: `/usr/share/elasticsearch`
* Plugin manager: `/usr/share/elasticsearch/bin/plugin`
* Configuration: `/etc/elasticsearch/elasticsearch.yml`
* Data: `/var/lib/elasticsearch/<cluster-name>`

### Logstash:
* Binaries and stuff: `/opt/logstash`
* Configuration: `/etc/logstash/conf.d`
* Logs: `/var/log/logstash`

### Nginx:
* Binaries, configuration and stuff: `/etc/nginx`
* Logs: `/var/log/nginx`

### Kibana
* Everything is in: `/var/www/kibana3`

### Nxlog
* Binaries and stuff: `/usr/bin/nxlog`
* Configuration: `DEB:/etc/nxlog/nxlog.conf, RPM:/etc/nxlog.conf`
* Logs: `/var/log/nxlog`

## Configurations
### Elasticsearch

Elasticsearch has sane defaults that work well for Logstash, so I'm pretty much going to leave it alone except for someone who would like to improve security. Let's edit the configuration to make it more secure:
```
sudo vim /etc/elasticsearch/elasticsearch.yml
```
Add the following line somewhere in the file, to disable dynamic scripts:
```
script.disable_dynamic: true
```
**Optional**: *If you want to restrict outside access to your Elasticsearch instance (port 9200), so outsiders cannot read your data or shutdown your Elasticseach cluster through the HTTP API. Find the line that specifies network.host and uncomment it so it looks like this:*
```
network.host: localhost
```
Save and exit `elasticsearch.yml`. Now start Elasticsearch:
```
sudo service elasticsearch restart
```
There are some plugins that I like to have installed: `Bigdesk` and `elasticsearch-head`
#### Bigdesk 
> In simple words bigdesk makes it very easy to see how your Elasticsearch cluster is doing. Just install it as an Elasticsearch plugin, download locally or run online from the web, then point it to the Elasticsearch node REST endpoint and have fun.

To install, run:
```
sudo /usr/share/elasticsearch/bin/plugin -install lukas-vlcek/bigdesk/2.4.0 
```
Bigdesk can be accessible at `http://localhost:9200/_plugin/bigdesk/`

#### elasticsearch-head
> elasticsearch-head is a web front end for browsing and interacting with an Elastic Search cluster.

To install, run:
```
sudo /usr/share/elasticsearch/bin/plugin -install mobz/elasticsearch-head
```
elasticsearch-head can be accessible at `http://localhost:9200/_plugin/head/`

### Nginx and Kibana

Kibana's defaults work great as well, so I won't change its configuration either (actually I don't know how to yet :-/). I, however, do have to configure Nginx where to find and Kibana on this server. Let's edit Nginx configuration:
```
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.orig
sudo vim /etc/nginx/sites-available/default
```
Find and change the values of the server_name to your FQDN (or localhost if you aren't using a domain name) and root to where we installed Kibana, so they look like the following entries:
```
server {
	listen 80 default_server;
	listen [::]:80 default_server ipv6only=on;

	root /usr/share/nginx/html;
	index index.html index.htm;

	server_name FQDN;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ =404;
		# Uncomment to enable naxsi on this location
		# include /etc/nginx/naxsi.rules
	}

  location /kibana3 {
    alias /var/www/kibana3/;
		try_files $uri $uri/ =404;
  }
```
Then, a quick `sudo service nginx reload` should have it working.

### Generate SSL Certificates
My intention of this setup is to transfer CFEngine logs via SSL (or TLS if you want to be super totally correct) to gain benifits of
* Privacy (stop looking at my password)
* Integrity (data has not been altered in flight)
* Trust (you are who you say you are)

So I have to use `OpenSSL` to create my own private certificate autority. OpenSSL is a free utility that comes with most installations of MacOS X, Linux, the *BSDs, and UNIXes. You can also download a binary copy to run on your Windows installation.

The process for creating your own certificate authority is pretty straight forward:

1. Create a private key
2. Self-sign
3. Install root CA on your various workstations

Once you do that, every device that you manage via HTTPS just needs to have its own certificate created with the following steps:

1. Create CSR for device
2. Sign CSR with root CA key

Let's do it!

##### Create the Root Certificate
Create the directories that will store the certificate and private key with the following commands:
```
sudo mkdir -p /etc/pki/tls/certs
sudo mkdir /etc/pki/tls/private
```
Create the Root Key: (keep this **very private!**. This is the basis of all trust for your certificates, and if someone gets a hold of it, they can generate certificates that your system will accept.)
```
sudo openssl genrsa -out /etc/pki/tls/private/rootCA.key 2048
```
Next step is to self-sign this certificate:
```
sudo openssl req -x509 -new -nodes -key /etc/pki/tls/private/rootCA.key -days 3650 -out /etc/pki/tls/certs/rootCA.pem
```
This will start an interactive script which will ask you for various bits of information. Fill it out as you see fit.
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:NO
State or Province Name (full name) [Some-State]:Oslo
Locality Name (eg, city) []:Oslo
Organization Name (eg, company) [Internet Widgits Pty Ltd]:CFEngineers.net
Organizational Unit Name (eg, section) []:IT
Common Name (eg, YOUR name) []:Nakarin Phooripoom
Email Address []:mynameisjeang@gmail.com
```
Once done, this will create an SSL certificate called `rootCA.pem`, signed by itself, valid for 3650 days, and it will act as our root certificate. **This is the one that we are going to distribute to all client hosts so Nxlog will use to be able to communicate back to Logstash**

#### Create A Certificate (Done Once for Logstash)
Only on Logstash server. You will need to create another private key.
```
sudo openssl genrsa -out /etc/pki/tls/private/logstash.key 2048
```
Then generate certificate signing request:
```
sudo openssl req -new -key /etc/pki/tls/private/logstash.key -out /etc/pki/tls/certs/logstash.csr
```
You will be asked various questions. The important question to answer though is **common-name**.
```
Common Name (eg, YOUR name) []: 10.0.1.11
```
If it doesn’t match your IP address, even a properly signed certificate will not validate correctly and you will get the `“cannot verify authenticity”` error. Once that’s done, you will sign the CSR, which requires the root CA key.
```
sudo openssl x509 -req -in /etc/pki/tls/certs/logstash.csr -CA /etc/pki/tls/certs/rootCA.pem -CAkey /etc/pki/tls/private/rootCA.key -CAcreateserial -out /etc/pki/tls/certs/logstash.crt -days 3650
```
This creates a signed certificate called `logstash.crt` which is valid for 3650 days. To be ready to use in Logstash configuration.

### Logstash
This is biggie! When logstash is started using its initscript, it'll simply check `/etc/logstash/conf.d` for configuration files and load them in. Recommended to read first.

Useful references:
* [Logstash's website](http://www.logstash.net)
* [Configuration overview page](http://logstash.net/docs/1.4.2/configuration)

Once you have written a configuration, you can test it by running `/opt/logstash/bin/logstash -t -f /etc/logstash/conf.d/` Then, when the config-test succeeds, just run `sudo service logstash start` to get going, and `tail -f /var/log/logstash/logstash.log` to make sure that everything is OK.

Here is an example of Logstash configuration.
* [logstash-cfe3-with-ssl-nxlog.conf](https://github.com/cfengineers-net/ELK/blob/master/logstash/conf.d/logstash-cfe3-with-ssl-nxlog.conf)
* [logstash-cfe3-with-udp-nxlog.conf](https://github.com/cfengineers-net/ELK/blob/master/logstash/conf.d/logstash-cfe3-with-udp-nxlog.conf)

### Nxlog
On every Nxlog clients, we need to have root CA locate locally. We can simply `scp` from Logstash server to client hosts.
```
logstash# scp /etc/pki/tls/certs/rootCA.pem root@CLIENT001:/root
```
Then on the client host, we move it to `/etc/pki/tls/certs` to be ready to add to Nxlog configuration.
```
sudo mkdir -p /etc/pki/tls/certs
sudo cp /root/rootCA.pem /etc/pki/tls/certs/rootCA.pem
```
Here is an example of Nxlog configuration.
* [ssl-nxlog.conf](https://github.com/cfengineers-net/ELK/blob/master/nxlog/ssl-nxlog.conf)
* [udp-nxlog.conf](https://github.com/cfengineers-net/ELK/blob/master/nxlog/udp-nxlog.conf)

Once all components are up and running. Then we can head on over to Kibana, choose the included `"Logstash Dashboard"` and look at all your pretty logs! or you can even inspect your performance and data directly from Elasticsearch Bigdesk or elasticsearch-head.

**"Have a lot of fun ..." :-P**

### ELK image example

* **Kibana**
![Kibana3](/images/kibana3.png)

* **Elasticsearch-Bigdesk**
![Elasticsearch-Bigdesk](/images/elasticsearch-bigdesk.png)

* **Elasticsearch-Head**
![Elasticsearch-Head](/images/elasticsearch-head.png)
