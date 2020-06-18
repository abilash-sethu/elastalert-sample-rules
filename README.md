# elastalert-sample-rules
#Please check the pdf documentation for more details which i have committed along with the rules sample
This contains some sample rules to work with elastalert https://elastalert.readthedocs.io/en/latest/index.html
Step-by-step guide
Step 1:- Installation
Windows 10:-

Setup Elasticsearch Instance  --> I used version 6.8.2 the instance is hosted in https://www.elastic.co/cloud/
Install Python 3.6 -----> Install python 3.6.x from https://www.python.org/downloads/windows/
Install ---> Microsoft Visual Build Tools
Download Elastalert ----> git clone https://github.com/Yelp/elastalert.git
From the Elastalert folder --> pip install -r requirements.txt
From the Elastalert folder --> python setup.py install
For Linux:- https://elastalert.readthedocs.io/en/latest/running_elastalert.html

Note:- We can also run Elastalert on a container

 Step 2:- Create Elastalert indexes in Elasticsearch to store alerts and errors
From Elastalert Folder use this command to create required meta indexes --> elastalert-create-index 
This will create following indexes try list indexes from the cluster using → https://host:port/_cat/indices?v


Step 3:- Configure The global Elastalert configuration
Rename the config.yaml.example file to config.yaml
Provide the instance informations like host,port,ssl_required,username,password,etc..
config.yaml

Step 4:- Dump some data to the Elasticsearch 
Create a spring boot application from initializer
You can connect with Elasticsearch either by using Spring data jest/Spring data Elasticsearch
Using Spring Data Jest use (Deprecated)
Spring boot version---> 2.1.11.RELEASE
Spring data Jest version ---> 3.2.2.RELEASE
Required properties:-
spring.data.elasticsearch.properties.path.logs=target/elasticsearch/log
spring.data.elasticsearch.properties.path.data=target/elasticsearch/data
spring.data.elasticsearch.cluster-name=# clustername
spring.data.elasticsearch.cluster-nodes=#host:port
spring.data.jest.password=#password
spring.data.jest.username=#username
spring.data.jest.readTimeout=100000000
spring.data.jest.uri=#https://host:port
Using Spring Data Elasticsearch
You can use the latest upgrade 4.0 with spring boot 2.3.0
Required properties:-
spring.data.elasticsearch.cluster-name=elasticsearch # Elasticsearch cluster name.
spring.data.elasticsearch.cluster-nodes= # Comma-separated list of cluster node addresses.
spring.data.elasticsearch.properties.*= # Additional properties used to configure the client.
spring.data.elasticsearch.repositories.enabled=true # Whether to enable Elasticsearch repositories
Important Note :- The data we dump into an index need a date field with ISO8601 or Unix timestamped format
In spring boot we can acheive this by adding :-
@Field(type = FieldType.Date, store = true, format = DateFormat.custom, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
@JsonFormat (shape = JsonFormat.Shape.STRING, pattern ="yyyy-MM-dd'T'HH:mm:ss.SSS'Z'")
private Date timestamp;

Timestamp mapping should look like this in the index:- 

 "timestamp": {
                        "type": "date",
                        "store": true,
                        "format": "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"
                    }

Example, Create an index with name Event and dump some data and read using → https://host:port/event/_search?q=*



Step 5:- Create Alert Rules and configure Alert Types
Create rules
There are different types of rules you can create like any,new_term, blacklist, whitelist, frequency etc..,
Refer this for creating a rule --> https://elastalert.readthedocs.io/en/latest/running_elastalert.html#creating-a-rule
Refer this for rules configurations and types --> https://elastalert.readthedocs.io/en/latest/ruletypes.html
Refer this to know available alerts that you can configure with the alert ---> https://elastalert.readthedocs.io/en/latest/ruletypes.html#alerts

Timestamp configuration
If the timestamp is not a @timestamp you need to explicitly say which field to use as timestamp, provide the below configurations to the rule YAML file:-
timestamp_field: timestamp
timestamp_type: custom
timestamp_format: '%Y-%m-%dT%H:%M:%S.%fZ'
timestamp_format_expr: 'ts[:23] + ts[26:]

Email SMTP Configuration for rules
alert:
- "email"
email:
- "xxx.s@xxx.com" (to list)
email_from_field: "alerts"
email_add_domain: "@ucc.com"
smtp_host: "smtp.gmail.com"
smtp_port: 465
smtp_ssl: true
smtp_auth_file: 'email_auth.yaml' (Authentication credentials file)

The email_auth.yaml should contain username and password in YAML format for example:-

user: xxxx@gmail.com
password: xxxxx

HTTP POST Alert Type
This will post the resultant alert as JSON, the required configurations are 

alert: post
http_post_url: "http://localhost:8080/api/event"

In our server, we need to have a rest endpoint which accepts the JSON body



Example Rules
blacklist.yaml
email_auth.yaml
events_term.yaml
change_rule_http_post.yaml
frequency_rule.yaml
flatline.yaml
any_rule.yaml
whitelist.yaml

Step 7:- Testing the alert rule
Use this command :- elastalert-test-rule your_rules/your_rule.yaml
This is only for test purpose this will log the details to the console, this will not send actual alerts if we want to send the real alerts we can specify the --alert flag, below are output logs of some rule types

New Term (Email alert) 
This rule matches when a new value appears in a field that has never been seen before. When ElastAlert starts, it will use an aggregation query to gather all known terms for a list of fields/query_key


Elastalert will poll the Elasticsearch according to the configured buffer_time and print the matched document to the console


At last, the server will write the results to the index for audit purpose


Change (HTTP POST Alert)
This rule will monitor a certain field and match if that field changes
Resultant log at the end (real alerts are sending used --alert flag)


Frequency
This rule matches when there are at least a certain number of events in a given time frame.
Resultant log at the end


Flatline
 This rule matches when the total number of events is under a given threshold for a time period.
Resultant log at the end


When using flatline rule or spike this won't work as we expected in a test environment, this needs a minimum_elapsed_timeframe to begin the alerts

 

In order to trigger the alerts, we can increase the timeframe with the configuration

timeframe:
     days: 2

Any
Any rule will match everything. Every hit that the query returns will generate an alert.
Resultant log at the end


Blacklist
The blacklisting rule will check a certain field against a blacklist, and match if it is in the blacklist.
Whitelist
Similar to blacklist, this rule will compare a certain field to a whitelist, and match if the list does not contain the term.
Apart from the blacklist, the whitelist had a required field ignore_null: If true, events without a compare_key field will not match. If not provided a validation error will throw
Resultant log at the end


Common Issues/concerns might face during the test
Configure Buffer Time
Elastalert checks the data in a range of 15 minutes(comparing with the timestamp we are provided to the document) we can customize this by passing --start and --end parameters

We can customize the polling time using "buffer_time" By default, ElastAlert will query large overlapping windows to ensure that it does not miss any events, even if they are indexed in real-time. In config.YAML, you can adjust "buffer_time to a smaller number to only query the most recent few minutes.



buffer_time:
       minutes: 5

Why did I only get one alert when I expected to get several?
Log:- "INFO:elastalert: Ignoring match for silenced rule Event rule"
There is a setting called realert which is the minimum time between two alerts for the same rule. Any alert that occurs within this time will simply be dropped. The default value for this is one minute. If you want to receive an alert for every single match, even if they occur right after each other, use, by default this property is 1
realert:
    minutes: 0

Issue With keyword type fields
Log: INFO:elastalert:Found no values for your_field (When pulling the data for the first time)

When you're working with term queries, for example writing a new_term rule, by default the string fields mapping type will be the keyword,  So when you specify a field for filter/querying you need to postfix .keyword to the fields. Elastalert provides use_keyword_postfix boolean configuration to solve this issue, by default this property is set to true. another solution here is to change the field type itself in Elasticsearch you can do this by annotating the field  @Field( type = FieldType.Text, fielddata = true)

To see the mapping for the index:- https://host:port/index/_mapping/

For a String field by default, the mapping will be 

 "name": {
                        "type": "text",
                        "fields": {
                            "keyword": {
                                "type": "keyword",
                                "ignore_above": 256
                            }
                        }
                    },



A Very helpful resource:- https://awesomeopensource.com/project/Yelp/elastalert

Issue when testing the HTTP Post Alert type
Issue:- AttributeError: '_thread._local' object has no attribute 'alerts_sent'

Fix:- Open elatalert.py on the local AppData, make the below change

self.replace_dots_in_field_names = self.conf.get('replace_dots_in_field_names', False)
self.thread_data.num_hits = 0
self.thread_data.num_dupes = 0
self.thread_data.alerts_sent = 0 #add
self.scheduler = BackgroundScheduler()

Step 8:- Run the Elastalert server
Use this command:- python -m elastalert.elastalert --verbose --rule your_rules_dir/your_rule.yaml
Output Log
INFO:elastalert:Queried rule Event rule blacklist from 2020-05-21 16:02 India Standard Time to 2020-05-21 16:05 India Standard Time: 0 / 0 hits
INFO:elastalert:Ran Event rule blacklist from 2020-05-21 16:02 India Standard Time to 2020-05-21 16:05 India Standard Time: 0 query hits (0 already seen), 0 matches, 0 alerts sent
INFO:elastalert:Disabled rules are: []
INFO:elastalert:Sleeping for 59.9995 seconds
INFO:elastalert:Background configuration change check run at 2020-05-21 16:06 India Standard Time
INFO:elastalert:Sent email to ['abhilash.s@infospica.com']
INFO:elastalert:Sent email to ['abhilash.s@infospica.com']
INFO:elastalert:Queried rule Event rule blacklist from 2020-05-21 16:02 India Standard Time to 2020-05-21 16:06 India Standard Time: 1 / 1 hits
INFO:elastalert:Background alerts thread 2 pending alerts sent at 2020-05-21 16:06 India Standard Time
INFO:elastalert:Sent email to ['abhilash.s@infospica.com']
INFO:elastalert:Ran Event rule blacklist from 2020-05-21 16:02 India Standard Time to 2020-05-21 16:06 India Standard Time: 1 query hits (0 already seen), 1 matches, 1 alerts sent
INFO:elastalert:Disabled rules are: []
INFO:elastalert:Sleeping for 60.0 seconds
INFO:elastalert:Background configuration change check run at 2020-05-21 16:07 India Standard Time
INFO:elastalert:Background alerts thread 0 pending alerts sent at 2020-05-21 16:07 India Standard Time
INFO:elastalert:Queried rule Event rule blacklist from 2020-05-21 16:02 India Standard Time to 2020-05-21 16:07 India Standard Time: 1 / 1 hits
INFO:elastalert:Ran Event rule blacklist from 2020-05-21 16:02 India Standard Time to 2020-05-21 16:07 India Standard Time: 1 query hits (1 already seen), 0 matches, 0 alerts sent
INFO:elastalert:Disabled rules are: []
INFO:elastalert:Sleeping for 60.0 seconds
INFO:elastalert:Background configuration change check run at 2020-05-21 16:08 India Standard Time
INFO:elastalert:Background alerts thread 0 pending alerts sent at 2020-05-21 16:08 India Standard Time
INFO:elastalert:Queried rule Event rule blacklist from 2020-05-21 16:02 India Standard Time to 2020-05-21 16:08 India Standard Time: 2 / 2 hits
INFO:elastalert:Sent email to ['abhilash.s@infospica.com']
INFO:elastalert:Ran Event rule blacklist from 2020-05-21 16:02 India Standard Time to 2020-05-21 16:08 India Standard Time: 2 query hits (1 already seen), 1 matches, 1 alerts sent
INFO:elastalert:SIGINT received, stopping ElastAlert...

Email Message:- 
The email alert looks like this (We can customize the subject and content)


