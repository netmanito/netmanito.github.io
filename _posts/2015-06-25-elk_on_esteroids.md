---
layout: post
title: ELK on esteroids
date: 2015-06-26
categories: jekyll elasticsearch
---

Ok, ELK is fashion now and you can find many documentation on how to install and configure ELK environments and more.
But the problem comes when you want to make it work daily.

Here are some ELK Tips, but first, 

### Some important questions


1. You cannot have more than one ES (ElasticSearch) master node.

2. Consider you have 3 nodes n1, n2 and n3 that all contain data, and currently n1 is selected as the master master node. If you query in n2 node the query will be distributed to all corresponding shards of indexes[replica shard or primary shard]. The result from each shards are combined and return back to you (see the query phase docs).
It's not necessary to distribute the query by master node. Any node data or master or non data node can act as router[Distributing search queries].

3. The master node can be small if the node does not contain data because it need not take care of data management.Its only work is to just route the queries to corresponding nodes and return the result to you. If the master node contains data then you should have configuration more than an data node. because it have 2 works [data management,routing query]..


Now, some help,

### HQ Plugin

This is a great plugin to manage your Elasticsearch Cluster.

Install it with 

	/usr/share/elasticsearch/bin/plugin -i royrusso/elasticsearch-HQ

Then, point your browser to the below address and you'll be able to see and graphically manage your ES cluster.
	
	http://your.elasticsearch.host:9200/_plugin/HQ/

#### GrokDebug
Grok will help you parse your own app logs or whatever you want to index.

	https://grokdebug.herokuapp.com/

### Logstash Configtest

If you use more than one file in your config, you can check the hole directory configuration. 

	/opt/logstash/bin/logstash --configtest -f /etc/logstash/conf.d/

If you prefer you can check a certain file configuration too.

	/opt/logstash/bin/logstash --configtest -f /etc/logstash/conf.d/05-file.cfg

You should see something like that on your screen

	You are using a deprecated config setting "type" set in elasticsearch. Deprecated settings will continue to work, but are scheduled for removal from logstash in the future. You can achieve this same behavior with the new conditionals, like: `if [type] == "sometype" { elasticsearch { ... } }`. If you have any questions about this, please visit the #logstash channel on freenode irc. {:name=>"type", :plugin=><LogStash::Outputs::ElasticSearch --->, :level=>:warn}
	Configuration OK

**Be careful as different versions may change their configuration syntax!** 

### Fake SSL Certs

This is an extract of the following [url](http://serverfault.com/questions/633681/logstash-forwarder-is-throwing-ssl-errors "serverfault")

The certificate you are using does not have any valid IP SAN's as mentioned in the message: 

	Failed to tls handshake with x.x.x.x x509: cannot validate certificate for x.x.x.x because it doesn't contain any IP SANs

If you connect using an IP address then your certificate must contain a matching IP SAN to pass validation with Go 1.3 and higher. This is not (yet?) mentioned in any README file or documentation.

To permit IP address as the server name, the SSL cert must include IP address as a **subjectAltName** field.

To solve that you can use following procedure for the creation of the SSL cert and key:

- Create a file **notsec.cfr** (or any other name) containing output like:


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

If you connect via host names, you can remove the IP SAN's, otherwise add your **logstash** server IP address.

- Create the certificate and key with following command (using the file from point 1):

		openssl req -x509 -batch -nodes -newkey rsa:2048 -keyout notsecure.key -out notsecure.crt -config notsec.cnf -days 1825

This will create a kind of wildcard certificate accepting any hostname and the IP addresses mentioned it that file. Of course this is just a simple example and you will need to adjust the settings to your needs.

### Delete Index

The delete index API allows to delete an existing index.


	$ curl -XDELETE 'http://localhost:9200/indextoremove/'
	
You can delete also all indices with 

	curl -XDELETE ‘http://localhost:9200/_all


### Old Indices ES-Curator

With Curator you can manage automatically old Indices to close or remove them.  

Then use pip to install elasticsearch-curator

	pip install elasticsearch-curator

Add the following line to run curator at 20 minutes past midnight (system time) and connect to the elasticsearch node on 127.0.0.1 and delete all indexes older than 120 days and close all indexes older than 90 days.

	20 0 * * * /usr/local/bin/curator --host elasticsearch -d 120 -c 90

### Manage ES Throttle

Set throttle to unlimited in *ES* if you need to bulk import your data.

	curl -XPUT 'http://172.17.0.142:9200/_cluster/settings' -d '
	{
    "transient" : {
        "indices.store.throttle.type" : "none"
    }
	}'
	
### Fancy Queries


	curl -XGET ‘http://172.17.0.142:9200/_count?pretty' -d '
	
	curl http://localhost:9200/_cluster/health?pretty
	
	curl http://localhost:9200/_nodes/process?pretty

	
### Some usefull links

*	ES

https://www.elastic.co/guide/en/elasticsearch/guide/current/root-object.html

https://blog.codecentric.de/en/2014/05/elasticsearch-indexing-performance-cheatsheet/

https://www.elastic.co/blog/performance-considerations-elasticsearch-indexing

*	Kibana

https://www.elastic.co/guide/en/kibana/3.0/queries.html

https://www.elastic.co/guide/en/beats/packetbeat/current/_kibana_query.html

https://www.mjt.me.uk/posts/kibana-101/

*	ELK

https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-centos-7