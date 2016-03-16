---
layout: post
title: ELK the definitive guide
date: 2016-03-15
categories: jekyll elasticsesarch update
---
# INSTALL LOGSTASH-ELASTICSEARCH-KIBANA + LOGSTASH FORWARDERS

In this tutorial, we will go over the installation of the Elasticsearch ELK Stack on CentOS 7-that is, Elasticsearch 1.4.4, Logstash 1.5.0, and Kibana 4. We will also show you how to configure it to gather and visualize the syslogs of your systems in a centralized location. Logstash is an open source tool for collecting, parsing, and storing logs for future use. Kibana 4 is a web interface that can be used to search and view the logs that Logstash has indexed. Both of these tools are based on Elasticsearch.

Our Logstash / Kibana setup has four main components:

*Logstash: The server component of Logstash that processes incoming logs*

*Elasticsearch: Stores all of the logs*

*Kibana: Web interface for searching and visualizing logs, which will be proxied through Nginx*

*Logstash Forwarder: Installed on servers that will send their logs to Logstash, Logstash Forwarder serves as a log forwarding agent that utilizes the lumberjack networking protocol to communicate with Logstash*

Note: This tutorial can be applied to install versions 2.X from elasticsearch and logstash, just keep in mind that there might be little changes, checkout [elasitc.co](https://elastic.co) to see changes.


## Install needed elements

### EPEL

You might need EPEL repositories enabled on your system

[Follow instructions from  https://fedoraproject.org/wiki/EPEL/](https://fedoraproject.org/wiki/EPEL/)

### JAVA

You need to install java in order to make all this work. JAVA openjdk-7 is fine and oracle one is not really needed.

	yum install java-1.7.0-openjdk.x86_64

### ELASTICSEARCH

Run the following command to import the Elasticsearch public GPG key into rpm:

	sudo rpm --import http://packages.elasticsearch.org/GPG-KEY-elasticsearch

Create and edit a new yum repository file for Elasticsearch:

	sudo vi /etc/yum.repos.d/elasticsearch.repo

Add the following repository configuration:

	[elasticsearch-1.4]
	name=Elasticsearch repository for 1.4.x packages
	baseurl=http://packages.elasticsearch.org/elasticsearch/1.4/centos
	gpgcheck=1
	gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
	enabled=1

Save and exit.

Install Elasticsearch 1.4.4 with this command:

	sudo yum -y install elasticsearch-1.4.4

Elasticsearch is now installed. Let's edit the configuration:

	sudo vi /etc/elasticsearch/elasticsearch.yml

You will want to restrict outside access to your Elasticsearch instance (port 9200), so outsiders can't read your data or shutdown your Elasticsearch cluster through the HTTP API. Find the line that specifies network.host, uncomment it, and replace its value with "localhost" so it looks like this:

	network.host: localhost

Save and exit elasticsearch.yml.

Note: If you work with a elasticsearch cluster you'll need to use FQDN or IP addresses visible from the rest of the nodes. 


Now start Elasticsearch:

	sudo systemctl start elasticsearch.service

Then run the following command to start Elasticsearch automatically on boot up:

	sudo systemctl enable elasticsearch.service

Now that Elasticsearch is up and running, let's install logstash

### Install Logstash

The Logstash package shares the same GPG Key as Elasticsearch, and we already installed that public key, so let's create and edit a new Yum repository file for Logstash:

	sudo vi /etc/yum.repos.d/logstash.repo

Add the following repository configuration:

	[logstash-1.5]
	name=logstash repository for 1.5.x packages
	baseurl=http://packages.elasticsearch.org/logstash/1.5/centos
	gpgcheck=1
	gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
	enabled=1

Save and exit.

Install Logstash 1.5 with this command:

	sudo yum -y install logstash

Logstash is installed but it is not configured yet. We need to do a few things before

### Certificate generation

If the certificate you are using does not have any valid IP SAN's as mentioned in the message:

	Failed to tls handshake with x.x.x.x x509: cannot validate certificate for x.x.x.x because it doesn't contain any IP SANs

If you connect using an IP address then your certificate must contain a matching IP SAN to pass validation with Go 1.3 and higher. This is not (yet?) mentioned in any README file or documentation but there are some issues ( #226 , #221 ) open at the project github repo.
To permit IP address as the server name, the SSL cert must include IP address as a subjectAltName field.

To solve that you can use following procedure for the creation of the SSL cert and key:

Create a file notsec.cfr (or any other name) containing output like:

    [req]
    distinguished_name = req_distinguished_name
    x509_extensions = v3_req
    prompt = no

    [req_distinguished_name]
    C = TG
    ST = Togo
    L =  Lome
    O = Private company
    CN = *

    [v3_req]
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid,issuer
    basicConstraints = CA:TRUE
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = *
    DNS.2 = *.*
    DNS.3 = *.*.*
    DNS.4 = *.*.*.*
    DNS.5 = *.*.*.*.*
    DNS.6 = *.*.*.*.*.*
    DNS.7 = *.*.*.*.*.*.*
    IP.1 = 192.168.1.1
    IP.2 = 192.168.69.14  

If you connect via host names, you can remove the IP SAN's, otherwise add your logstash server IP address.

Create the certificate and key with following command (using the file from point 1):

	openssl req -x509 -batch -nodes -newkey rsa:2048 -keyout notsecure.key -out notsecure.crt -config notsec.cnf -days 1825

This will create a kind of wildcard certificate accepting any hostname and the IP addresses mentioned it that file. Of course this is just a simple example and you wil need to adjust the settings to your needs.

The notsecure.crt file will be copied to all of the servers that will send logs to Logstash but we will do that a little later. Let's complete our Logstash configuration. If you went with this option, skip option 2 and move on to Configure Logstash.


## Configure Logstash

Lets see what we can find in configuration file.
This is a basic configuration file for logstash indexer. We'll see logstash-forwarder config later on.
We can use [https://grokdebug.herokuapp.com/](https://grokdebug.herokuapp.com/) to create patterns for our logs if they're not standard based.
By default, logstash can understand most of typicall logs with included patterns like apache, nginx, tomcat, etc. 

### Basic Configuration

This configuration example uses **lumberjack** as input source, we create a custom *type* to define a name and we use a certificate to authorize the input.
Once received, we parse our data with **grok**. We define a filter for our *type* and make it match with 2 different messages, anything that wont match these patterns, will be parsed as a *_grokparsefailure* which will be dropped from our filter as we don't want this information.
Keep in mind that for debbuging purposes, you should not drop this tag until you get correctly the data you want from your logs.

You can see 3 different blocks in the configuration that are mandatory and must follow their order. **Input**, **Filter/parse** and **Output**

	root@ESJC-HAHP-MS01S:~# cat /etc/logstash/conf.d/config.cfg
	##########################################################
	## Input is from we'll receive the information, we're opening port 5000 and we'll read for syslog data

	input {
	lumberjack {
	 port => 5000
	 type => "mylogs" 
	 ssl_certificate => "/etc/pki/tls/certs/notsecure.crt" 
	 ssl_key => "/etc/pki/tls/private/notsecure.key" 
 	 }
	}
	############################################
	## we can define many filters, but at first we start with syslog and define it's grok patern

	filter {
  	 if [type] == "mylogs" {
    	   grok {
      	     patterns_dir => "/etc/logstash/conf.d/patterns" 
      	     match => { "message" => "%{WORD:DEBUG} %{TIMESTAMP_ISO8601} \[] %{JAVACLASS}:%{WORD:parseRequest} - %{GREEDYDATA}=%{GREEDYDATA}" 
	                "message" => "%{WORD:DEBUG} %{TIMESTAMP_ISO8601} \[] %{JAVACLASS}:%{WORD:parseRequest} - %{DATA:message}:%{GREEDYDATA}" 
 	     }
    	   }  
    	 }
	}
	#############################################
	## this filter will drop all failed parsed information, just to clean data.
	
	filter {
	 if "_grokparsefailure" in [tags] {
      	  drop { }
    	 }
	}
	##################################################
	## output to elasticsearch for data explotation
	output {
	  elasticsearch {
	   host => localhost
	  }
	  stdout { 
	    codec => rubydebug 
  	  }
	}
	##################################################

### Check configuration

If you need to check logstash configuration changes please run the following:

	root@server:/etc/logstash/conf.d# /opt/logstash/bin/logstash --configtest -f /etc/logstash/conf.d/config.cfg

## KIBANA

You can use NginX to bypass kibana or apache,

We'll first install kibana

### Install kibana3

Download kibana3, decompress and place it in a directory.

	wget https://download.elastic.co/kibana/kibana/kibana-3.1.2.tar.gz

Kibana3 is quite simple compared to Kibana4 which is a whole app. In this case, kibana3 is just a buch of files to make logstash easier to use and we've figured out to make geoip work on in, but not on Kibana4

Check whether the following line is uncommented in config.js

	    elasticsearch: "http://"+window.location.hostname+":80",
You don't need to touch anything if you have an elasticsearch node installed and available in the same host.

On Kibana3 we'll use dashboards that will be stored in kibana3/app/dashboards/ with .json extension.
We can create many dashboards and have them placed at this directory and later on access through

	http://logstash-server/#/dashboard/file/dashboard-N.json

### Install kibana4

News on kibana4, no more static web system, now it starts all on listening to port 5601, so we'll configure the webserver to proxypass this address.

	cd ~; wget https://download.elasticsearch.org/kibana/kibana/kibana-4.0.1-linux-x64.tar.gz

Extract Kibana archive with tar:

	tar xvf kibana-*.tar.gz

Open the Kibana configuration file for editing:

	vi ~/kibana-4*/config/kibana.yml

In the Kibana configuration file, find the line that specifies host, and replace the IP address ("0.0.0.0" by default) with "localhost":

	host: "localhost"

Save and exit. This setting makes it so Kibana will only be accessible to the localhost. This is fine because we will use an Nginx reverse proxy to allow external access.

Let's copy the Kibana files to a more appropriate location. Create the /opt directory with the following command:

	sudo mkdir -p /var/www/kibana4

Now copy the Kibana files into your newly-created directory:

	sudo cp -R ~/kibana-4*/* /var/www/kibana4/

Kibana can be started by running /opt/kibana/bin/kibana, but we want it to run as a service. Create the Kibana systemd init file using vi:

	sudo vi /etc/systemd/system/kibana4.service

Now paste in this Kibana init file:

	[Service]
	ExecStart=/var/www/kibana4/bin/kibana
	Restart=always
	StandardOutput=syslog
	StandardError=syslog
	SyslogIdentifier=kibana4
	User=root
	Group=root
	Environment=NODE_ENV=production

	[Install]
	WantedBy=multi-user.target

Save and exit.

Now start the Kibana service, and enable it:

	sudo systemctl start kibana4
	sudo systemctl enable kibana4

Before we can use the Kibana web interface, we have to set up a reverse proxy. Let's do that now, with Nginx.

### NGINX

Install nginx to proxy pass kibana

	yum -y install nginx httpd-tools

We have default config and we'll use port 80 for kibana3.

Comment the following part in /etc/nginx/nginx.conf and we'll create our own servers.

	 ... server {
	       listen 80;
	       location / { 
	       root   html;
	       index  index.html index.htm;
       	}
	 ...
 	}


#### kibana3 with nginx
Create a configuration in nginx to serve the kibana3 service.

	root@ESJC-HAHP-MS01S:/etc/nginx/conf.d# cat nginx.kibana3.conf 
    	server {
	        listen       80 default_server;
	        server_name  localhost;
	        location / {
	          root         /var/www/kibana3;
	          index  index.html  index.htm;
	
	location ~ ^/_aliases$ {
	  proxy_pass http://127.0.0.1:9200;
	  proxy_read_timeout 90;
	        }
	location ~ ^/.*/_aliases$ {
	  proxy_pass http://127.0.0.1:9200;
	  proxy_read_timeout 90;
	}
	location ~ ^/_nodes$ {
	  proxy_pass http://127.0.0.1:9200;
	  proxy_read_timeout 90;
	}
	location ~ ^/.*/_search$ {
	  proxy_pass http://127.0.0.1:9200;
	  proxy_read_timeout 90;
	}
	location ~ ^/.*/_mapping {
	  proxy_pass http://127.0.0.1:9200;
	  proxy_read_timeout 90;
	}
	}
	       # redirect server error pages to the static page /40x.html
	       #
	error_page  404              /404.html;          
	location = /40x.html {                           
	}                                                
	# redirect server error pages to the static page /50x.html
	#                                                
	error_page   500 502 503 504  /50x.html;         
	location = /50x.html {
	}
	}

#### kibana4 with nginx

We'll create an Nginx server block in a new file:

	sudo vi /etc/nginx/conf.d/nginx.kibana4.conf

Paste the following code block into the file. Be sure to update the server_name to match your server's name:


	server {
	    listen 80;
	    server_name example.com;
	    location / {
	        proxy_pass http://localhost:5601;
	        proxy_http_version 1.1;
	        proxy_set_header Upgrade $http_upgrade;
	        proxy_set_header Connection 'upgrade';
	        proxy_set_header Host $host;
	        proxy_cache_bypass $http_upgrade;        
    		}
	}

Save and exit. This configures Nginx to direct your server's HTTP traffic to the Kibana application, which is listening on localhost:5601.

Now start and enable Nginx to put our changes into effect:

	sudo systemctl start nginx
	sudo systemctl enable nginx

### Proxy_passing Kibana over APACHE

Maybe you've an APACHE running and don't want to use nginx or you're not able to, therefore, you still can proxy pass kibana with apache, please follow this configuration.

	admin@ESJC-HAHP-AS07P ~ $ cat /etc/httpd/conf.d/kibana.conf 
	# Proxypass kibana4
	ProxyPass /kibana/ http://monitor01:5601/
	ProxyPassReverse /kibana/ http://monitor01:5601/
        # Proxypass kibana3
	ProxyPass /kibana3/ http://elasticserver/
	ProxyPassReverse /kibana3/ http://elasticserver/

	    # Proxy for _aliases and .*/_search
	  <LocationMatch "^/(_nodes|_aliases|.*/_aliases|_search|.*/_search|_mapping|.*/_mapping)$">
	    ProxyPassMatch http://elasticserver:9200/$1
	    ProxyPassReverse http://elasticserver:9200/$1
	  </LocationMatch>

	  # Proxy for kibana-int/{dashboard,temp} stuff (if you don't want auth on /, then you will want these to be protected)
	  <LocationMatch "^/(kibana-int/dashboard/|kibana-int/temp)(.*)$">
	    ProxyPassMatch http://elasticserver:9200/$1$2
	    ProxyPassReverse http://elasticserver:9200/$1$2
	  </LocationMatch>


## Logstash Forwarder

This plugin is deprecated on version 2.X and over and it's been substituted by *BEATS*.
 
Instructions from https://github.com/elastic/logstash-forwarder have been used to build rpm packages.
Once we have installed the forwarder, we need two important files, autogenerated certificate used to communicate with logstash-server and configuration file described below.

If you built the package, you'll need to install it on the forwarder hosts and use the certificate to connect.

 logstash-forwarder-0.4.0-1.x86_64.rpm
 logstash-forwarder.crt

	[admin@ESJC-HAHP-AS05P ~]$ cat /etc/logstash-forwarder
	{
	  "network": {
	    "servers": [ "logstash-server:5000" ],
	    "timeout": 15,
	    "ssl ca": "/etc/pki/tls/certs/logstash-forwarder.crt" 
	  },
	  "files": [
    	{
	      "paths": [
	        "/var/log/mobtel/mobtel-mimov/mimov.log" 
	       ],
	      "fields": { "type": "syslog" }
	    },
	    {
	      "paths": [
	        "/var/log/mobtel/mobtel-devicehandler/devicehandler.log" 
	       ],
	      "fields": { "type": "oysta" }
	    }
	   ]
	}

Run it with the following bash script or edit /etc/init.d/logstash-forwarder to addapt the options as in here.
You can manage spool-size from 1-100+ depending how much logs you index.

	 root@ESJC-HAHP-AS01P:~# cat logstash-forwarder.sh
	 nohup /opt/logstash-forwarder/bin/logstash-forwarder -config=/etc/logstash-forwarder -spool-size=10 -log-to-syslog &

You can check if it's working fine by looking at messages log.

	root@ESJC-HAHP-AS05P:/home/admin#  tailf /var/log/messages
	Apr 13 09:54:32 esjc-hahp-as05p logstash-forwarder[11095]: 2015/04/13 09:54:32.669446 Registrar: processing 66 events
	Apr 13 09:55:00 esjc-hahp-as05p logstash-forwarder[11095]: 2015/04/13 09:55:00.126108 Registrar: processing 34 events


# ELK  Architecture


ELK organization is defined as you want or need. In our case, we've 3 nodes to manage the elasticsearch cluster with logstash and kibana included.
We've 2 hosts LSF sending via lumberjack over a master node with 2 ES instances in slave mode, one with store data and the input one has no store.
Then we've 2 more ES hosts, both data stored and one, acting as master and the third slave.
 

	ESJC-HAHP-AS05P - LSF ---> |                                                                             |--> ESJC-HAHP-MS01P ES NODE2 MASTER
                	           | --- > ESJC-HAHP-MS01S|hahpremsc - ELK (slave:slave) | ES NODE0 + NODE1 <--- |                                     
	ESJC-HAHP-AS06P - LSF ---> |                                                                             |--> ESJC-HAHP-AS08S ES NODE3 - SLAVE 

Update: we've added a new Logstash input service to separate live and stage environment as a segmentation network done.

	ESJC-HAHP-AS05P - LSF ---> |                                                                             |--> ESJC-HAHP-MS01P ES NODE2 MASTER
                	           | --- > ESJC-HAHP-MS01S|hahpremsc - ELK (slave:slave) | ES NODE0 + NODE1 <--- |                                     
	ESJC-HAHP-AS06P - LSF ---> |                                                                             |--> ESJC-HAHP-AS08S ES NODE3 - SLAVE 

	ESJC-HAHP-AS05S - LSF ---> |                                                                             
                	           | --- > ESJC-HAHP-MS01P|hahpromsc - LGS -->  ESJC-HAHP-MS01S | ES NODE0 + NODE1                                      
	ESJC-HAHP-AS06S - LSF ---> |                                                                             

## Logstash configurations

### INPUTS

### 01-input.cfg

Input examples for lumberjack and redis.

	###########################################################
	input {
	  lumberjack {
	    port => 5000
	    type => "mimov, oysta" 
	    ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt" 
	    ssl_key => "/etc/pki/tls/private/logstash-forwarder.key" 
	  }
	}
	##########################################################

	input {
	  lumberjack {
	    port => 5001
	    type => "mimov, oysta" 
	    tags => "prepro" 
	    ssl_certificate => "/etc/pki/tls/certs/logstash-forwarder.crt" 
	    ssl_key => "/etc/pki/tls/private/logstash-forwarder.key" 
  	  }
	}
	##########################################################
	#input {
	#  redis {
	#    host => "127.0.0.1" 
	#    data_type => "list" 
	#    key => "logstash" 
	#    threads => 2
	#    batch_count => 1000
	#  }
	#}
	##########################################################

### PARSERS

#### 05-mimov.cfg

This parser uses grok and a pattern file detailed below

	######################################################
	filter {
	  if [type] == "mimov" {
	    grok {
	    # grok plugin
	    # %{MIMOV_MSG}=%{MIMOV_NOTIFY} is defined in /etc/logstash/conf.d/patterns/mimov
	    
	     patterns_dir => "/etc/logstash/conf.d/patterns" 
	     match => { "message" => "%{MIMOV_MSG}=%{MIMOV_NOTIFY}" 
	                }
	    # we add a tag and remove unwanted fields, this optimices data storage
	      add_tag => [ "mimov" ]
	      remove_field => [ 	"message","DATA1","DATA2","DATA4","DATA6","DATA8","DATA9","DATA10","DATA11","DATA12"]
	      }
	  }
	}
	################################  
	# this mutate replaces LOC and puts the word 'cycle'
	
	filter {
	 if [type] == "mimov" and [alarmType] == "LOC" {
	  mutate {
	  replace => [ "alarmType", "cycle" ]
	        }
	 }
	}
	################################  
	# this mutate filter replaces NEW and LAS to match other inputs with same names as above
	
	filter {
	 if [type] == "mimov" and [hasFix] == "NEW" {
	  mutate {
	  replace => [ "hasFix", "true" ]
	        }
	  } else if [type] == "mimov" and [hasFix] == "LAS" {
	  mutate {
	  replace => [ "hasFix", "false" ]
	        }
	  }
	}
	################################  
	# we don't want anything incorrectly parsed so we drop it
	
	filter {
	if "_grokparsefailure" in [tags] {
	      drop { }
	    }
	}
	#############################################

#### 10-oysta.cfg

This example uses **XML** and **Grok** filters.

	#############################################
	filter {
	  if [type] == "oysta" {
	     multiline {
	                       pattern => "^\s|</slevent>|^[A-Za-z].*" 
	                       what => "previous" 
                	}
	      xml {
	           store_xml => "false" 
	           source => "message" 
	           xpath => [ 
                	      "/slevent/event/text()", "alarmType",
	                      "/slevent/device/@id", "deviceID",
                	      "/slevent/device/name/text()", "deviceName",
	                      "/slevent/device/msisdn/text()", "deviceMsisdn",
                	      "/slevent/device/imei/text()", "deviceIMEI",
	                      "/slevent/device/type/text()", "deviceType",
	                      "/slevent/device/battery/text()", "deviceBatteryLevel",
	                      "/slevent/device/mileage/text()", "deviceMileage",
	                      "/slevent/logtime/@stamp", "timestamp",
	                      "/slevent/logtime/text()", "externalPlatformReceptionTimeStamp",
	                      "/slevent/eventtime/@stamp", "locationTimeStamp",
	                      "/slevent/eventtime/text()", "locationTime",
                	      "/slevent/quality/@hasfix", "hasFix",
	                      "/slevent/quality/@hdop", "hdop",
                	      "/slevent/quality/text()", "quality",
	                      "/slevent/location/@type", "locationType",
                	      "/slevent/location/latitude/text()", "latitude",
	                      "/slevent/location/longitude/text()", "longitude",
                	      "/slevent/location/speed/text()", "speed",
	                      "/slevent/location/datetime/@stamp", "receptionDate",
                	      "/slevent/location/datetime/text()", "receptionDate",
	                      "/slevent/location/address/text()", "address" 
                	    ]
	           }
	       # Remove the intermediate fields before output. "message" contains the
	       # original message (XML). You may or may-not want to keep that.
	        mutate {
	            remove_field => ["offset"]
	            remove_field => ["deviceMileage"]
	            #remove_field => ["message"]
	            add_tag => ["oysta"]
	       }
	    }
	}
	#############################################
	filter {
	    # multiline filter adds the tag "multiline" only to lines spanning multiple lines
	    # We _only_ want those here.
	    if "multiline" in [tags] {
	       grok {
	            match => { "message" => "%{WORD:} %{TIMESTAMP_ISO8601} \[]" }
	            add_tag => [ "delete" ]
	       }
	    }
	}
	#############################################
	filter {
	if "oysta" in [tags] {
	     grok { 
	            match => { "message" => "^<\?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"yes\"\?>"  }
	            add_tag => [ "delete" ]
	    }
	 }
	}
	#############################################
	filter {
	if "oysta" in [tags] {
	     grok { 
	            match => { "message" => ".*<.*smart35.*>"  }
	            add_tag => [ "xml" ]
	            remove_tag => [ "_grokparsefailure" ]
	    }
	 }
	}
	#############################################
	filter {
	 if "oysta" in [tags] {
	   mutate {
	     remove_field => [ "message" ]
	   }
	 }
	}
	#############################################
	filter {
	if "delete" in [tags] {
	      drop { }
	    }
	}
	#############################################
	filter {
	if "_grokparsefailure" in [tags] {
	      drop { }
	    }
	}	
	#############################################

#### 12-mtc.cfg

This is **json** plugin example

	filter {
	  if [type] == "mtc" {
	######################################################
		if [measures] and [measures][externalId] { 
		  mutate {
		    add_tag => [ "measure" ]
		    add_field => [ "measureDate", "%{[measures][0][measureDate]}" ]
		    add_field => [ "hasFix", "%{[measures][0][origin]}" ]
		    add_field => [ "latitude", "%{[measures][0][latitude]}" ]
		    add_field => [ "longitude", "%{[measures][0][longitude]}" ]
		    add_field => [ "deviceBatteryLevel", "%{[measures][0][batteryLevel]}" ]
		    add_field => [ "alarmType", "cycle" ]
		    remove_field => [ "ack","alarms" ]
		    # remove_tag => [ "enest" ]
		  }
		}
	else
	#############################################
	 if [alarms] and [alarms][pkey] {
		mutate {
		   add_field => [ "alarmType", "%{[alarms][0][type]}" ]
		   add_field => [ "msisdn", "%{[alarms][0][msisdn]}" ]
		   add_field => [ "alarmDate", "%{[alarms][0][alarmDate]}" ]
		   add_field => [ "measureDate", "%{[alarms][0][measure][measureDate]}" ]
		   add_field => [ "hasFix", "%{[alarms][0][measure][origin]}" ]
		   add_field => [ "latitude", "%{[alarms][0][measure][latitude]}" ]
		   add_field => [ "longitude", "%{[alarms][0][measure][longitude]}" ]
		   add_field => [ "deviceBatteryLevel", "%{[alarms][0][measure][batteryLevel]}" ]
		   add_tag => [ "alarm" ]
		   remove_field => [ "ack","measures" ]
	  }
	 }   
	else
	#############################################
	 if [measureDate] {
		date { 
		  match => [ "measureDate","UNIX_MS" ]
		  target => [ "measureDate" ]
		}
	 }
	#############################################
	 }
	}
	#############################################
	filter {

	if "_grokparsefailure" in [tags] {
	      drop { }
	    }
	}

	filter {
	if "_jsonparsefailure" in [tags] {
	      drop { }
	    }
	}
	#############################################

#### 14-enest.cfg

This example uses **grok** and **json**

	#############################################
	filter {
	  if [type] == "enest" {
	    grok {
		patterns_dir => "/etc/logstash/conf.d/patterns" 
		match => { "message" => "- %{WORD}   %{TIMESTAMP_ISO8601} %{JAVACLASS}:%{WORD} - %{WORD}\/%{WORD} %{WORD}:%{GREEDYDATA:message}" }
		overwrite => [ "message" ] 
		add_tag => [ "enest" ]
		remove_tag => [ "_grokparsefailure" ]
	    }   
	 }  
	}
	#############################################
	filter {
	  if "enest" in [tags] {
	    json {
		source => "message" 
		add_field => ["latitude", "%{[location][latitude]}" ] 
		add_field => ["longitude", "%{[location][longitude]}" ] 
		remove_field => [ "message","host" ]
	    }   
	  } 
	} 
	#############################################
	filter {
	if "_jsonparsefailure" in [tags] {                                                        
	      drop { }
	    } 
	}   
	#############################################                                             
	filter {
	 if [event] == "HEARTBEAT" {                                                              
	      drop { }
	 }                                                                                        
	}
	#############################################                                             
	filter {
	if "_grokparsefailure" in [tags] {                                                        
	      drop { }
	    } 
	}   
	#############################################                                             

#### 15-geoip.cfg

The **geoip** plugin to manage locations.

	################################ GEOIP 
	filter {
	if [latitude] and [longitude] {
	    mutate {

	    add_field => [ "[geoip.location]", "%{longitude}" ]
	    add_field => [ "[geoip.location]", "%{latitude}" ]
	    }
		mutate {
	       convert => [ "[geoip.location]", "float" ]
	    }
	  }
	}
	#############################################
	filter {
	       geoip {
		     source => "geoip.location" 
		     target => "geoip" 
		     database =>"/etc/logstash/GeoLiteCity.dat" 
		     }
		 #Delete 0,0 in SourceGeo.location if equal to 0,0
	    if ([geoip.location] and [geoip.location] =~ "0,0") {
	      mutate {
		replace => [ "geoip.location", "" ]
	      }
	    }
	}
	#############################################

### OUTPUT

Output example, basically elasticsearch, but remember you can output to many other things.

#### 20-output.cfg

	##################################################
	#output {
	#
	#  if [type] == "mimov" {
	#    elasticsearch {
	#      host => ["server1:9300","server2:9300"]
	#      protocol => transport
	#      cluster => mycluster
	#    }
	#  }
	#  if [type] == "oysta" {
	#    elasticsearch {
	#      host => "server3" 
	#      protocol => transport
	#    }
	# stdout { 
	#      codec => rubydebug 
	#    }
	#
	#  }
	#}

	output {
	  elasticsearch {
	    host =>  "elastic-host" 
	    protocol => transport
	    cluster => mycluster
	    #manage_template => true
	    #workers => 1
	    flush_size => 2000
	    idle_flush_time => 5
	    index => "logstash-%{+YYYY.MM.dd}" 
	  }
	}
	##################################################

### PATTERNS
This patterns have been created at  [https://grokdebug.herokuapp.com/](https://grokdebug.herokuapp.com/) 

#### mimov

	MIMOV_MSG %{WORD:} %{TIMESTAMP_ISO8601} \[] %{JAVACLASS}:%{WORD:} - %{WORD:} %{WORD:} %{WORD:} %{WORD:} :%{WORD:MIMOV}
	MIMOV_NOTIFY %{DATA:TransactionID}\_%{DATA:deviceIMEI}\_%{DATA:locationDate}\_%{DATA:locationTime}\_%{DATA:deviceType}\_%{DATA:alarmType}\_%{DATA:locationDate}\_%{DATA:locationTime}\_%{DATA:longitude}\_%{DATA:latitude}\_%{DATA:DATA1}\_%{DATA:DATA2}\_%{DATA:speed}\_%{DATA:DATA4}\_%{DATA:quality}\_%{DATA:DATA6}\_%{DATA:hasFix}\_%{DATA:DATA8}\_%{DATA:DATA9}\_%{DATA:DATA10}\_%{DATA:DATA11}\_%{DATA:hdop}\_%{DATA:deviceBatteryLevel}\_%{GREEDYDATA}

#### oysta

	OYSTA_MSG %{WORD:} %{TIMESTAMP_ISO8601} \[] %{JAVACLASS}:%{WORD:} - %{WORD:} %{WORD:} = %{WORD:} \[ 
	OYSTA_NOTIFICATION %{WORD:}=%{DATA:MSG}, %{WORD:}=%{DATA:alarmType}, %{WORD:}=%{DATA:measureOrigin}, %{WORD:}=%{DATA:command}, %{WORD:}=%{DATA:ackId}, %{WORD:}=%{DATA:safezoneId}, %{WORD:}=%{DATA:deviceID}, %{WORD:}=%{DATA:deviceName}, %{WORD:}=%{DATA:deviceMsisdn}, %{WORD:}=%{DATA:deviceIMEI}, %{WORD:}=%{DATA:deviceType}, %{WORD:}=%{DATA:deviceBatteryLevel}, %{WORD:}=%{DATA:deviceMileage}, %{WORD:}=%{TIMESTAMP_ISO8601}, %{WORD:}=%{DATA:externalPlatformReceptionTimeStamp}, %{WORD:}=%{DATA:quality}, %{WORD:}=%{DATA:hasFix}, %{WORD:}=%{DATA:hdop}, %{WORD:}=%{DATA:cellinfoMCC}, %{WORD:}=%{DATA:cellinfoMNC}, %{WORD:}=%{DATA:cellinfoLAC}, %{WORD:}=%{DATA:cellinfoCellID}, %{WORD:}=%{DATA:locationType}, %{WORD:}=%{DATA:locationTimeStamp}, %{WORD:}=%{DATA:locationTime}, %{WORD:}=%{DATA:latitude}, %{WORD:}=%{DATA:longitude}, %{WORD:}=%{DATA:speed}, %{WORD:}=%{DATA:address}, %{WORD:}=%{DATA:receptionDate}\]

## Elasticsearch Plugins


#### Plugin HQ

Install with

	/usr/share/elasticsearch/bin/plugin -install royrusso/elasticsearch-HQ

Then point your browser to

	http://myserver:9200/_plugin/HQ/#cluster

#### ES Curator

Automatic index cleaning via Curator

You can use the curator program to delete indexes. See more information in the github repository: https://github.com/elasticsearch/curator

Youfll need pip in order to install curator:

	$ sudo apt-get install python-pip

Once itfs done, you can install curator:

	$ sudo pip install elasticsearch-curator

Now, itfs easy to setup a cron to delete the indexes older than 30 days in /etc/cron.d/elasticsearch_curator:

	@midnight     root    curator --host $host delete indices --time-unit days --timestring "\%Y.\%m.\%d" --older-than 30 >> /var/log/curator.log 2>&1

#### Kibana queries

*Kibana Reference:*

	https://www.elastic.co/guide/en/kibana/3.0/queries.html

Type this very simple query into the search bar:

	to be or not to be

You will notice, in the table, your first hit is Hamlet as expected. However, look at the next line by Sir Andrew, it does not contain "to be", nor does it contain "not to be". The search we have actually performed is: to OR be OR or OR not OR to OR be.

We can also match the entire phrase:

	"to be or not to be"

Or in specifc fields:

	line_id:86169

We can express complex searches with AND/OR, note these words must be capitalized:

	food AND love

Or parantheses:

	("played upon" OR "every man") AND stage

Numeric ranges can also be easily searched:

	line_id:[30000 TO 80000] AND havoc

And of course to search everything:

	*

Also at [https://www.elastic.co/guide/en/kibana/3.0/multiple-queries.html](https://www.elastic.co/guide/en/kibana/3.0/multiple-queries.html) you'll find howto make multiple queries.
Here's a very good web on howto use basic kibana [https://www.mjt.me.uk/posts/kibana-101/](https://www.mjt.me.uk/posts/kibana-101/)

## Other Usuefull Information

### Fancy Queries

**Show cluster health**

	curl http://localhost:9200/_cluster/health?pretty

#### Show cluster processes

	curl http://localhost:9200/_nodes/process?pretty

#### Get index mappings

	curl -XGET 'http://localhost:9200/_mapping'

#### Do a search in a certain index

	curl -XGET 'http://localhost:9200/logstash-2015.04.16/_search?smart359464032046919=true' -d '' | xargs -p

#### Show indices available

	curl -XGET 'http://localhost:9200/_cat/indices?v'


### Can I have multiple Master NODES in my ES Cluster?

Answer 1) You cannot have more than one master node.

Answer 2) Consider you have 3 nodes n1, n2 and n3 that all contain data, and currently n1 is selected as the master master node. If you query in n2 node the query will be distributed to all corresponding shards of indexes[replica shard or primary shard]. The result from each shards are combined and return back to you (see the query phase docs).

It's not necessary to distribute the query by master node. Any node data or master or non data node can act as router[Distributing search queries].

Answer 3) yes the master node can be small if the node does not contain data because it need not take care of data management.Its only work is to just route the queries to corresponding nodes and return the result to you. If the master node contains data then you should have configuration more than an data node. because it have 2 works [data management,routing query]..

### Setting throttle to unlimited.

If you need to make bulk import from your data, ensure to free ES limit throttle. You can later put it back as it was.
Please follow this instructions on how tunning your cluster https://www.elastic.co/guide/en/elasticsearch/guide/master/indexing-performance.html

	curl -XPUT 'http://localhost:9200/_cluster/settings' -d '
	{
	    "transient" : {
		"indices.store.throttle.type" : "none" 
	    }
	}'

### Setting ES to DEBUG mode

	https://www.elastic.co/guide/en/elasticsearch/guide/master/logging.html


### ES, when to use TCP (transport) module or HTTP module in your cluster

The transport module is used for internal communication between nodes within the cluster. Each call that goes from one node to the other uses the transport module (for example, when an HTTP GET request is processed by one node, and should actually be processed by another node that holds the data).

The transport mechanism is completely asynchronous in nature, meaning that there is no blocking thread waiting for a response. The benefit of using asynchronous communication is first solving the C10k problem, as well as being the ideal solution for scatter (broadcast) / gather operations such as search in ElasticSearch.

### Vagrant set box 

https://github.com/comperiosearch/vagrant-elk-box-ansible

http://blog.comperiosearch.com/blog/2015/11/26/elk-stack-deployment-with-ansible/
