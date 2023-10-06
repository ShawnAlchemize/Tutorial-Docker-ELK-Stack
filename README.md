#How To Install Elasticsearch, Logstash, and Kibana (Elastic Stack) in a WSL Docker container (Ubuntu 20.04)

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
Pull an ubuntu:20.04 image from [Docker's IOfficial Images](https://hub.docker.com/_/ubuntu "Docker's IOfficial Images"):
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
```bash
	network.host: localhost
	http.port:9200
	discovery.type: single-node
```
<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/c0795d9d-a965-47b3-b58d-022cfbbc8286" height="250" >

As we're using `discovery.type: single-node`, comment out the following line at the bottom of the file:
```bash
#cluster.initial_master_nodes: ["……"]
```
By default, JVM heap size is set at 1GB.
```bash
nano /etc/elasticsearch/jvm.options
```
It is recommended to set it to no more than half the size of your total memory. 
Set the heap size by uncommenting the following lines. Here, I've configured it to be 4GB:
```bash
-Xms4g
-Xmx4g
```
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

### Add a Superuser account:
As the generated password might be difficult to remember and inconvenient to log in with later, we'll create a supeuser role account. 
```bash
/usr/share/elasticsearch/bin/elasticsearch-users useradd <your-username> -p <your-username> -r superuser
```
There are many more possible roles available, but we're sticking with the same role as `elastic` user. 

### Test Elasticsearch
In  `/etc/elasticsearch/elasticsearch.yml`, if `xpack.security.enabled: true`: (default)
This means that security is enabled.

<img src="https://github.com/ShawnAlchemize/Docker-ELK-Stack-Tutorial/assets/33109120/b02e433b-b9d3-4c66-aeff-a194263f579b" height="250" >

```bash
curl -k -u elastic:<generated-password> https://localhost:9200
# OR
curl -k -u <your-username>:<password> https://localhost:9200
```
Else if, in  `/etc/elasticsearch/elasticsearch.yml`, if `xpack.security.enabled: false`: 
This means that security is disabled.
```bash
curl -X GET "localhost:9200"
```
The name of your system should display, and elasticsearch for the cluster name. This indicates that Elasticsearch is functional and is listening on port 9200.












