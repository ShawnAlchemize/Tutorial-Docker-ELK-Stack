# How To Install Elasticsearch, Logstash, and Kibana (Elastic Stack) in a WSL Docker container (Ubuntu 20.04)

In this document, you'll learn how to set up an ELK stack in a docker container running Ubuntu 20.04. These steps are applicable to setup ELK on WSL2 as well, with minimal adjustments - just skip installing docker.

## Pre-requisites:
- WSL2
- [Install Docker on WSL2](https://docs.docker.com/engine/install/ubuntu/ "Install Docker on WSL2")
```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
# Add the repository to Apt sources:
echo \
"deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
"$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update 
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```


## Open a terminal in WSL:
Pull an ubuntu:20.04 image from [Docker's Official Images](https://hub.docker.com/_/ubuntu "Docker's IOfficial Images"):
```bash
sudo docker pull ubuntu:20.04
```

Create a directory to mount with docker container: (optional)
```bash
cd security-analytics
mkdir elk_share
```
Create a Docker container with a mounted directory (elk_share) & expose the 9200 and 5601 ports:
```bash
sudo docker run -it \
--name elk-stack \
-d -p 9200:9200 -p 5601:5601 \
--shm-size=1g \
--ulimit memlock=-1 \
--ulimit stack=67108864 \
-v ~/security-analytics/elk_share:/elk_share \
ubuntu:20.04
```
Start the docker container:
```bash
sudo docker start elk-stack
```

# Open a terminal inside Docker container:
```bash
sudo docker exec -it elk-stack /bin/bash
```

#### Prepare the environment:
```bash
apt update && apt upgrade -y && apt-get install nginx -y && apt-get install wget -y && apt-get install gpg -y && apt-get install nano -y && apt install systemctl -y && apt install curl -y && apt install net-tools -y
```

#### Set up rsyslog service:
In this tutorial, we'll go through the steps to install `rsyslog` and configure it to log shell commands to the syslog. Then using filebeats to send the logs to Logstash.

##### Step 1: Install rsyslog
First, we need to install `rsyslog`, which is a powerful logging system for Linux.
```bash
sudo apt install rsyslog
```

##### Step 2: Enable Command Logging in the Shell
1. Open the `.bashrc` file in a text editor. You can use `nano` or any other text editor of your choice.
```bash
	nano /etc/bashrc
```
2. Add the following line at the end of the file to set the `PROMPT_COMMAND` environment variable. This command will log every command to syslog as it's executed.
```bash
	PROMPT_COMMAND='history -a >(logger -t "[$USER] $SSH_CONNECTION")'
```
3. Save the file and exit the text editor. (for nano: ctrl-s + ctrl-x)

4. To apply the changes immediately, you need to source the `.bashrc` file:
```bash
	source ~/.bashrc
```

#####  Step 3: View Command Logs
Now that you have enabled command logging, you can view the logged commands in the syslog. Use the `cat` command to display the syslog:
```bash
cat /var/log/syslog
```
You should see entries that look like:
```
Oct  7 12:34:56 hostname [username] [ip_address]: command_here
```
This format includes the date and time, the username, the IP address (if applicable), and the executed command.

# ELK Stack
## Install Elasticsearch 
```bash
apt-get install openjdk-8-jdk -y
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
apt-get install apt-transport-https -y
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-8.x.list
apt-get update
apt-get install elasticsearch -y
```
### Configure Elasticsearch
```bash
	nano /etc/elasticsearch/elasticsearch.yml
```
Add/uncomment the following lines:

`network.host: localhost`
`http.port:9200`
`discovery.type: single-node`


As we're using `discovery.type: single-node`, comment out the following line at the bottom of the file:
`#cluster.initial_master_nodes: ["……"]`

<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/c0795d9d-a965-47b3-b58d-022cfbbc8286" height="250" >

By default, JVM heap size is set at 1GB.
```bash
nano /etc/elasticsearch/jvm.options
```
It is recommended to set it to no more than half the size of your total memory. 
Set the heap size by uncommenting the following lines. Here, I've configured it to be 4GB:
`-Xms4g`
`-Xmx4g`

<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/f72aff41-499a-4ba2-aa3f-4eaa09d32793" height="250" >

### Start Elasticsearch

#### As ELK stack is being set up on Docker or if is being set up as Root user in WSL2, do this first to avoid file access permission errors:
```bash
mkdir -p /var/run/elasticsearch 
chown elasticsearch:elasticsearch /var/run/elasticsearch
```
When you start Elasticsearch for the first time, passwords are generated for the `elastic` user and TLS is automatically configured for you.
```bash
systemctl start elasticsearch.service
#Enable Elasticsearch to start on boot:
systemctl enable elasticsearch.service 
#Check Elasticsearch status:
systemctl status elasticsearch.service
```

### Reset Password
```bash
/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```
Press 'Y' to print the password in terminal when prompted. This will be your `elastic` user password.

### Add a Superuser account
As the generated password might be difficult to remember and inconvenient to log in with later, we'll create a supeuser role account. 
```bash
/usr/share/elasticsearch/bin/elasticsearch-users useradd <your-newacct-username> -p <your-newacct-password> -r superuser
```
There are many more possible roles available, but we're sticking with the same role as `elastic` user. 

### Test Elasticsearch
In  `/etc/elasticsearch/elasticsearch.yml`, if `xpack.security.enabled: true`: (default)
This means that security is enabled.

<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/b02e433b-b9d3-4c66-aeff-a194263f579b" height="250" >

```bash
curl -k -u elastic:<generated-password> https://localhost:9200
# OR
curl -k -u <your-newacct-username>:<your-newacct-password> https://localhost:9200
```
Else if, in  `/etc/elasticsearch/elasticsearch.yml`, if `xpack.security.enabled: false`: 
This means that security is disabled.
```bash
curl -X GET "localhost:9200"
```
The name of your system should display, and elasticsearch for the cluster name. This indicates that Elasticsearch is functional and is listening on port 9200.

## Install Kibana 
```bash
apt-get install kibana
nano /etc/kibana/kibana.yml
```
Edit the following lines:
`server.port: 5601`
`server.host: "0.0.0.0"`
`elasticsearch.host: ["http://localhost:9200"]`
<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/572ad956-3f44-408a-a8c8-9104de653d88" height="250" >


Then start Kibana:
```bash
systemctl start kibana
systemctl enable kibana
systemctl status kibana
```

### Test Kibana
Inside Docker terminal: `curl -X GET "localhost:5601"`
In WSL terminal: `curl -X GET "172.17.0.2:5601"`

### Enroll Kibana
**On Windows host machine:** In a local web browser -- visit `localhost:5601`
**Inside Docker terminal: **
```bash 
	/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token --scope kibana
		#<obtain an enrollment token>
```
Paste the enrollment token in the web browser on your local Windows machine.

**Inside Docker terminal:**
Run the following command for verification code:
```bash
/usr/share/kibana/bin/kibana-verification-code
```
Paste the verification code in the web browser on your local Windows machine.

### Login into Kibana
Use the `elastic` account or the `new superuser` account and its corresponding password that you've created previously.

## Install Logstash
```bash
apt-get install logstash
systemctl start logstash
systemctl enable logstash
systemctl status logstash
chmod 777 -R /usr/share/logstash/ # Optional; for file permissions issues.
```
### All custom Logstash configuration files are stored in:
`/etc/logstash/conf.d/`

### Example configuration files (separated):
`nano /etc/logstash/conf.d/02-beats-input.conf`
```bash
input {
  beats {
    port => 5044
  }
}
```
`nano /etc/logstash/conf.d/10-syslog-filter.conf`
```bash
filter {
    if [fileset][module] == "system" {
        grok {
            match => { "message" => "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} %{GREEDYMULTILINE:[system][auth][message]}" }
            pattern_definitions => { "GREEDYMULTILINE" => "(.|\n)*" }
            remove_field => "message"
        }
        date {
            match => [ "[system][auth][timestamp]", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        }
        geoip {
            source => "[system][auth][ssh][ip]"
            target => "[system][auth][ssh][geoip]"
        }
    }
    else if [fileset][name] == "syslog" {
        grok {
            match => { "message" => "%{SYSLOGTIMESTAMP:[system][syslog][timestamp]} %{SYSLOGHOST:[system][syslog][hostname]} %{GREEDYMULTILINE:[system][syslog][message]}" }
            pattern_definitions => { "GREEDYMULTILINE" => "(.|\n)*" }
            remove_field => "message"
        }
        date {
            match => [ "[system][syslog][timestamp]", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        }
    }
}
```

`nano /etc/logstash/conf.d/30-elasticsearch-output.conf`
Modify according to your username and password.
```bash
output {
  elasticsearch {
    hosts => [ "https://localhost:9200" ]
    ssl_certificate_verification => false
    user => "elastic" # or your new super user account
    password => "<your-password>" # enter your password
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

### Alternatively, if you prefer only 1 single config file:
`/etc/logstash/conf.d/logstash-simple.conf`
```bash
input {
    beats {
        port => "5044"
    }
}
filter {
    grok {
        match => { "message" => "%{COMBINEDAPACHELOG}"}
    }
}
output {
    elasticsearch {
        hosts => [ "https://localhost:9200" ]
        ssl_certificate_verification => false
        user => "elastic"
        password => "aBp71qQxflISz457T5W0"
    }
}
```
### Test Logstash
Running as logstash user:
```bash
su -s /bin/bash -c '/usr/share/logstash/bin/logstash --path.settings /etc/logstash -t' logstash
```

# Setting up Beats
There are many different beats available, such as Filebeat, Winlogbeat, Metricbeat and Packetbeat. For this tutorial, we'll be using Filebeat. We'll be sending logs to Logstash instead of Elasticsearch via Filebeats. 

## Install Filebeat
```bash
apt-get install filebeat
nano /etc/filebeat/filebeat.yml
```

##### Filebeats input section
<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/b427a3db-e171-4028-8856-615ac62dd765" height="250" >

##### Filebeat modules section
<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/caf4300c-d694-4ab1-8916-b9b6eebea632" height="250" >

##### Kibana section
<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/a8e6f6fb-b7df-45ae-a0fc-9ba3b2c9e91f" height="250" >

##### Outputs section
Under the Elasticsearch output section, comment out the following lines:
`# output.elasticsearch:`
`# Array of hosts to connect to.`
`# hosts: ["localhost:9200"]`
##### Logstash Output section
Under the Logstash output section, remove the hash sign (#) in the following two lines:
`output.logstash`
`hosts: ["localhost:5044"]`

##### *note: if sending to Elasticsearch instead of Logstash, requires the following commands:
```bash
filebeat setup -e
```
### List of enabled/disabled modules
```bash
filebeat modules list
# OR
ls -l /etc/filebeat/modules.d
```

### To disable a module
```bash
filebeat modules disable <module-name.yml>
# OR
mv /etc/filebeat/modules.d/<module-name.yml> /etc/filebeat/modules.d/ <module-name.yml>.disabled
```

### In our tutorial, we'll enable system.yml module
```bash
filebeat modules enable system
nano /etc/filebeat/filebeat.yml
```
In Filebeats Input section: `enabled: false`
```bash
nano /etc/filebeat/modules.d/system.yml
```
<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/46c0316e-dea7-42ab-9de4-0d806a84b5e8" height="250" >


```bash
filebeat modules enable system
filebeat test config
systemctl start filebeat
```

# Verifying ELK + Filebeat
If you’ve set up your Elastic Stack correctly, Filebeat will begin shipping your syslog and authorization logs to Logstash, which will then load that data into Elasticsearch.

To verify that Elasticsearch is indeed receiving this data, query the Filebeat index with this command:
```bash
curl -k -u elastic:aBp71qQxflISz457T5W0 https://localhost:9200/filebeat-*/_search?pretty
```

# Systemctl Commands
```bash
systemctl start nginx elasticsearch kibana logstash filebeat
systemctl status nginx elasticsearch kibana logstash filebeat
systemctl enable nginx elasticsearch kibana logstash filebeat
systemctl restart nginx elasticsearch kibana logstash filebeat
```

# Troubleshooting Commands:
## .yml configs
```bash
nano /etc/filebeat/filebeat.yml
nano /etc/kibana/kibana.yml
nano /etc/logstash/logstash.yml
nano /etc/elasticsearch/elasticsearch.yml
```
## Display error logs:
```bash
cat /var/log/filebeat/*.ndjson
cat /var/log/kibana/kibana.log
cat /var/log/logstash/logstash-plain.log
cat /var/log/elasticsearch/elasticsearch.log
```
## List indexes:
```bash
curl -k -u elastic:<password> https://localhost:9200/_cat/indices?v&s=index&pretty
```
Or in Kibana: Dev Tools > Console > `GET "/_cat/indices?v&s=index&pretty"`

## Possible Elastic account roles:
	1) reporting_user
	2) logstash_admin
	3) machine_learning_admin
	4) kibana_user
	5) rollup_user
	6) rollup_admin
	7) remote_monitoring_collector
	8) superuser
	9) transport_client
	10) viewer
	11) kibana_admin
	12) watcher_admin
	13) remote_monitoring_agent
	14) monitoring_user
	15) ingest_admin
	16) transform_admin
	17) logstash_system
	18) apm_user
	19) watcher_user
	20) beats_system
	21) data_frame_transforms_user
	22) snapshot_user
	23) beats_admin
	24) apm_system
	25) kibana_system
	26) enrich_user
	27) machine_learning_user
	28) transform_user
	29) data_frame_transforms_admin
	30) editor
