This documentation describes how to setup the ELK stack and the Apache access log generator within AWS to demonstrate its basic functionalities and key features.

# Installation of Logstash and the required plugins

Launch an EC2 instance with Linux AMI 2017.09 and SSH to it.

$ java -version  
$ sudo yum install java-1.8.0 -y  
$ sudo yum remove java-1.7.0-openjdk -y  
$ java -version  
$ sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch  
$ sudo vi /etc/yum.repos.d/logstash.repo  
```
[logstash-6.x]
name=Elastic repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
$ sudo yum install logstash -y  
$ cd /usr/share/logstash/  
$ ll  
$ sudo bin/logstash-plugin install logstash-output-amazon_es  
$ vi /etc/logstash/conf.d/logstash.conf  
```
input {
    file {
       path => "/var/log/apache/*"
    }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    remove_field => "message"
  }
  date {
    match => [ "timestamp", "dd/MMM/YYYY:HH:mm:ss Z" ]
    locale => en
    remove_field => ["timestamp"]
  }
  geoip {
    source => "clientip"
  }
  useragent {
    source => "agent"
    target => "useragent"
  }
}

output {
    amazon_es {
      hosts => “<AWS-ES-DOMAIN-ENTPOINT-URL>"
      port => "443"
      region => “<AWS-REGION-ID>"
      index => "web-access-log"
      document_type => "apache"
    }
}
```
### Start Logstash
$ sudo bin/logstash -f /etc/logstash/conf.d/logstash.conf &  

### Shutdown Logstash
$ ps -ef | grep logstash  
$ sudo kill <process-id>  

To know more about the filtering options in Logstash take a look at this documentation URL  
https://www.elastic.co/guide/en/logstash/current/filter-plugins.html

# Installation of Fake Apache Log Generation

For simulating Apache access logs we are going to use the generator available below  
https://github.com/kiritbasu/Fake-Apache-Log-Generator

$ sudo yum install git -y  
$ git clone https://github.com/kiritbasu/Fake-Apache-Log-Generator.git  
$ ll  
$ cd Fake-Apache-Log-Generator/  
$ ll  
$ sudo pip install -r requirements.txt  
$ python apache-fake-log-gen.py  
$ sudo mkdir -p /var/log/apache/  
$ sudo python apache-fake-log-gen.py -n 0 -s 1 -o LOG -p /var/log/apache/ &  
$ cd /var/log/apache/  
$ ls -alh  

### Shutdown
$ ps -ef | grep python  
$ sudo kill <process-id>  
$ sudo rm -R /var/log/apache/*  

# Setup an AWS Elasticsearch Domain

Setup an AWS Elasticsearch domain either from the AWS Management Console or via aws-cli command below.

Required parameters:
```
<aws-region-id>
<aws-account-id>
<public-IP-address-of-your-machine>
```
```
% aws es create-elasticsearch-domain --domain-name demo-es --elasticsearch-version 6.2 --elasticsearch-cluster-config InstanceType=m3.medium.elasticsearch,InstanceCount=2 --access-policies '{"Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Principal": {"AWS": "*"}, "Action":"es:*", "Resource": "arn:aws:es:<aws-region-id>:<aws-account-id>:domain/demo-es/*", "Condition": { "IpAddress": { "aws:SourceIp": "<public-IP-address-of-your-machine>" } } } ] }' --region <aws-region-id>
```

Once created and the status is shown as Active, then check whether you can access the Kibana URL in your local browser.

Confirm that the “web-access-log” has been created in Elasticsearch
```
GET _cat/indices?v
```

# Modify the auto-created index mapping to create an index template with appropriate field types
Create an index pattern for the “web-access-log” index in Kibana by navigating to Management —> Index Patterns

Check the field types for the geo.* fields that were generated by Logstash 

Download the index mapping into the local machine and modify the below field types accordingly
```
% curl -XGET '<AWS-ES-DOMAIN-ENTPOINT-URL>/web-access-log/_mapping?pretty' > web-access-log_mapping.json
```
```
"dma_code" : { "type" : "short" },
"ip" : { "type" : "ip" },
"latitude" : { "type" : "half_float" },
"location" : { "type" : "geo_point" },
"longitude" : { "type" : "half_float" },
```
```
"bytes" : { "type" : “long" },
```
Create a new file and Save As “web-access-log_template.json” 

Remove the first two lines and add the followings
```
{
  "template": "web-access-log",
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1
  },
 ```
Remove the last '}'

Save the modified template file and go ahead delete the existing index from Elasticsearch, then upload the template file
```
DELETE web-access-log?pretty
```
```
% curl -XPUT '<AWS-ES-DOMAIN-ENTPOINT-URL>/_template/web-access-log?pretty' -H 'Content-Type: application/json' -d @web-access-log_template.json
```

Confirm the template creation in Elasticsearch
```
GET _template/web-access-log?pretty
```

# Kibana Visualisations
After importing the visualisation JSON file if you see errors in your imported dashboard and visualisation, then go back to Kibana Management —> Index Patterns and specify the "Custom index pattern ID" as "17bf4a60-4158-11e8-9430-f7edb27c70a5" in the second step under the "Hide advanced options". This id value can be found inside the "web-access-log_export.json" file.

Below is the list of visualisations that are created as part of this demo.

Average Response Size in Bytes — Vertical Bar  
HTTP Response Codes over Time — Vertical Bar  
HTTP Response Status — Pie  
HTTP Response Status by OS — Pie  
HTTP Response Status by REST Calls — Pie  
HTTP Response Status by Web Browsers — Pie  
Response Time Statistic — Metric  
Web Access by User Devices — Vertical Bar  
Web Access Count — Metric  
Web Access Distribution — Region Map  
Web Access Geo-location — Coordinate Map  

# Cleanup
```
% aws ec2 terminate-instances --instance-ids <value> --region-name <aws-region-id>
% aws es delete-elasticsearch-domain --domain-name demo-es --region-name <aws-region-id>
```
# Helpful ES queries
```
GET _cat/indices?v
GET web-access-log?pretty
GET web-access-log/_mapping?pretty
DELETE web-access-log?pretty

GET _template?pretty
GET _template/web-access-log?pretty
DELETE _template/web-access-log?pretty
```
# References
https://www.elastic.co/guide/en/logstash/current/installing-logstash.html  
https://www.elastic.co/blog/logstash_lesson_elasticsearch_mapping  
https://github.com/kiritbasu/Fake-Apache-Log-Generator  
https://github.com/awslabs/logstash-output-amazon_es  
