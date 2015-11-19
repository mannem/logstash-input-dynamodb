# Logstash Plugin for Amazon DynamoDB

The Logstash plugin for Amazon DynamoDB gives you a nearly real-time view of the data in your DynamoDB table. The Logstash plugin for DynamoDB uses DynamoDB Streams to parse and output data as it is added to a DynamoDB table. After you install and activate the Logstash plugin for DynamoDB, it scans the data in the specified table, and then it starts consuming your updates using Streams and then outputs them to Elasticsearch, or a Logstash output of your choice.

Logstash is a data pipeline service that processes data, parses data, and then outputs it to a selected location in a selected format. Elasticsearch is a distributed, full-text search server. For more information about Logstash and Elasticsearch, go to https://www.elastic.co/products/elasticsearch.

Amazon Elasticsearch Service is a managed service that makes it easy to deploy, operate, and scale Elasticsearch in the AWS Cloud.  https://aws.amazon.com/elasticsearch-service/

## The following sections walk you through the process to:

1. Create a DynamoDB table and enable a new stream on the table.
2. Download, build, and install the Logstash plugin for DynamoDB on Amazon Linux Ec2 Instance.
3. Configure Logstash to output to AWS Elasticsearch endpoint and the command line.
4. Run the Logstash plugin for DynamoDB.
5. Test Logstash by adding DynamoDB items to the table.

When this process is finished, you can search your data in the AWS Elasticsearch cluster using Kibana.

### Prerequisites

**The following items are required to use the Logstash plugin for Amazon DynamoDB:**

1. Amazon Web Services (AWS) account with DynamoDB
2. A running Elasticsearch cluster—To download Elasticsearch, go to https://www.elastic.co/products/elasticsearch. Or Spin up your own AWS Elasticsearch cluster at http://console.aws.amazon.com/es
3. You may need to install development tools which incldes some pre-reqs like gcc,git etc. You can skip this option though, ```sudo yum groupinstall "Development Tools" ```
4. Download all the pre-requisites. 

 a. JRuby—To download JRuby, you may install this with RVM : https://rvm.io/rvm/install . 
```
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
\curl -sSL https://get.rvm.io | bash -s stable --ruby
# add the following line to '/home/ec2-user/.bash_profile': source ~/.profile  
exec bash
# Install and use ruby inside JRuby instead of system provided ruby
rvm install jruby-9.0.1.0
/bin/bash --login
rvm --default use jruby-9.0.1.0
```
 b. Logstash—To download. You may clone logstash at https://github.com/awslabs/logstash-input-dynamodb, however this is minimalistic. Instead you may also download a stable logstash release from elastic.co with some added plugins for future use. https://www.elastic.co/downloads/logstash 
```
wget https://download.elastic.co/logstash/logstash/logstash-1.5.4.zip 
unzip logstash-1.5.4.zip
```
 c. Git—To download Git, go to http://git-scm.com/downloads. ```sudo yum install git```

 d. Apache Maven—To get Apache Maven, go to http://maven.apache.org/. 
 ```
# Install maven
 sudo wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
 sudo sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo
 sudo yum install -y apache-maven 
 ```
 e. Install JAVA 8 because Logstash support for Java 7 is on best efforts.  
Update java from a sample script here: https://github.com/mannem/Scripts/blob/master/java8UpdateScript.sh

```\curl https://raw.githubusercontent.com/mannem/Scripts/master/java8UpdateScript.sh | bash```  

### Before You Begin: Create a Source Table

In this step, you will create a DynamoDB table with DynamoDB Streams enabled. This will be the source table and writes to this table will be processed by the Logstash plugin for DynamoDB.

**To create the source table**

1. Open the DynamoDB console at https://console.aws.amazon.com/dynamodb/.
2. Choose **Create Table**.
3. On the **Create Table** page, enter the following settings:
   1. **Table Name** — SourceTable
   2. **Primary Key Type** — Hash
   3. **Hash attribute data type** — Number
   4. **Hash Attribute Name** — Id
   5. Choose **Continue**.
4. On the **Add Indexes** page, choose **Continue**. You will not need any indexes for this exercise.
5. On the **Provisioned Throughput** page, choose **Continue**.
6. On the **Additional Options** page, do the following:
    1. Select **Enable Streams**, and then set the **View Type** to **New and Old Images**.
    2. Clear **Use Basic Alarms**. You will not need alarms for this exercise.
    3. When you are ready, choose **Continue**.
7. On the **Summary** page, choose **Create**.

The source table will be created within a few minutes.

### Setting Up the Logstash Plugin for Amazon DynamoDB

To use the Logstash plugin for DynamoDB, you need to build, install, run the plugin, and then you can test it.

**IMPORTANT: in order to successfully build and install Logstash, you must have previously installed ```MAVEN``` to satisfy jar dependencies, and ```JRUBY``` to build and run the logstash gem.**

**To build the Logstash plugin for DynamoDB**

At the command prompt, change to the directory where you want to install the Logstash plugin for DynamoDB and demo project.

In the directory where you want the Git project, clone the (Updated) Git project:

```
git clone https://github.com/mannem/logstash-input-dynamodb.git
```

**Install the Bundler gem by typing the following:**

```
gem install bundler
```

The Bundler gem checks dependencies for Ruby gems and installs them for you.

To install the dependencies for the Logstash plugin for DynamoDB, type the following command (Inside the DynamoDB logstash directory):
This command may take a while to install. 
```
cd logstash-input-dynamodb/
bundle install
```

To build the gem, type the following command:

```
gem build logstash-input-dynamodb.gemspec
```

### To install the Logstash plugin for DynamoDB

Now that you have built the plugin gem, you can install it.

Change directories to your local Logstash directory.

In the Logstash directory, open the Gemfile file in a text editor and add the following line.

```
cd logstash-1.5.4
vim Gemfile
gem "logstash-input-dynamodb"
```

To install the plugin, in your logstash folder type the command:

```
bin/plugin install --no-verify logstash-input-dynamodb
```

To list all the installed plugins type the following command:

```
bin/plugin list
```

If the logstash-output-elasticsearch or logstash-output-stdout plugins are not listed you need to install them. For instructions on installing plugins, go to the Working with Plugins page in the Logstash documentation.

### Running the Logstash Plugin for Amazon DynamoDB

**NOTE: First, make sure you have *Enabled Streams* (see above) for your DynamoDB table(s) before running logstash. The streams should also have required view type (like new_and_old_images) that is specified in configuration.  Logstash for DynamoDB requires that each table you are logging from have a streams enabled to work.**

In the local Logstash directory type a command similar to this:

```
bin/logstash -e 'input { 
    dynamodb{endpoint => "dynamodb.us-west-2.amazonaws.com" 
    streams_endpoint => "streams.dynamodb.us-west-2.amazonaws.com" 
    view_type => "new_and_old_images" 
    aws_access_key_id => "<access_key_id>" 
    aws_secret_access_key => "<secret_key>" 
    table_name => "DynamoDBTableName"} } 
output { 
    elasticsearch 
        { host => "search-imdb-pgjs6t52gf3m2c5xkxpragmaer.us-west-2.es.amazonaws.com" port => "443" protocol => "http" ssl => "true"} 
        stdout { } }'
```

**Important**

This is an example command for configuring and running the logstash plugin. 

You must replace ```<access_key_id>``` and ```<secret_key>``` with your access key and secret key. If you have credentials saved in a credentials file, you can omit these configuration values.

You must replace ```DynamoDBTableName``` with your DynamoDb table name 

You must change the  DyanmoDb ```endpoint``` and ```streams_enpoint``` according to http://docs.aws.amazon.com/general/latest/gr/rande.html#ddb_region

You must change the AWS Elasticsearch ```hostname``` to the domain ```endpoint``` that you see in AWS Elasticsearch Web console.

This command should run the logstash with this configuration.

Logstash should successfully start and begin indexing the records from your DynamoDB table.

You can also change the other configuration options to match your particular use case.  

You can also configure the plugin to index multiple tables by adding additional ```dynamodb { }``` sections to the ```input``` section.

**The following table shows the configuration values.**

### Setting Description

Settings Id |  Description
------- | --------
table_name | The name of the table to index. This table must exist.
endpoint | The DynamoDB endpoint to use.  If you are running DynamoDB on your computer, use http://localhost:8000 as the endpoint.
streams_endpoint  | The name of a checkpoint table. This does not need to exist prior to plugin activation.
view_type | The view type of the DynamoDB stream. ("new_and_old_images", "new_image", "old_image", "keys_only" Note: these must match the settings for your table's stream configured in the DynamoDB console.)
aws_access_key_id | Your AWS access key ID. This is optional if you have credentials saved in a credentials file. Note: If you are running DynamoDB on your computer, this ID must match the access key ID that you used to create the table. If it does not match, the Logstash plugin will fail because DynamoDB partitions data by access key ID and region.
aws_secret_access_key | Your AWS access key ID. Your AWS access key ID. This is optional if you have credentials saved in a credentials file.
perform_scan | A boolean flag to indicate whether or not Logstash should scan the entire table before streaming new records. Note: Set this option to false if your are restarting the Logstash plugin.
checkpointer | A string that uniquely identifies the KCL checkpointer name and CloudWatch metrics name.  This is used when one worker leaves a shard so that another worker knows where to start again.
publish_metrics | Boolean option to publish metrics to CloudWatch using the checkpointer name.
perform_stream | Boolean option to not automatically stream new data into Logstash from DynamoDB streams.
read_ops  | Number of read operations per second to perform when scanning the specified table.
number_of_scan_threads | Number of threads to use when scanning the specified table.
number_of_write_threads | Number of threads to write to the Logstash queue when scanning the table.
log_format | Log transfer format. "plain" - Returns the object as a DynamoDB object. "json_drop_binary" - Translates the item format to JSON and drops any binary attributes. "json_binary_as_text" - Translates the item format to JSON and represents any binary attributes as 64-bit encoded binary strings. For more information, see the JSON Data Format topic in the DynamoDB documentation.

### Testing the Logstash Plugin for Amazon DynamoDB

The Logstash plugin for DynamoDB starts scanning the DynamoDB table and indexing the table data when you run it. As you insert new records into the DynamoDB table, the Logstash plugin consumes the new records from DynamoDB streams to continue indexing.

To test this, you can add items to the DynamoDB table in the AWS console, and view the output (stdout) in the command prompt window. The items are also inserted into Elasticsearch and indexed for searching.

**To test the Logstash plugin for DynamoDB**

Open the DynamoDB console at https://console.aws.amazon.com/dynamodb/.

In the list of tables, open (double-click) **SourceTable**.

Choose **New Item**, add the following data, and then choose **PutItem**:

Id—1
Message—First item

Repeat the previous step to add the following data items:

Id—2 and Message—Second item
Id—3 and Message—Third item

Return to the command-prompt window and verify the Logstash output (it should have dumped the logstash output for each item you added to the console).

**(Optional) Go back to the SourceTable in us-east-1 and do the following:**

Update item 2. Set the Message to Hello world!
Delete item 3.

Go to the command-prompt window and verify the data output.

You can now search the DynamoDB items in Elasticsearch. 

For information about accessing and searching data in Elasticsearch, see the Elasticsearch documentation.
