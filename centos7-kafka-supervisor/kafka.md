# Install Python and Java
```
sudo yum update -y
sudo yum install -y python3 java-1.8.0-openjdk wget
sudo yum install -y epel-release python-pip
#python -m pip install --upgrade pip
```

# Get Kafka and Supervisor installed
```
#user vagrant
wget https://downloads.apache.org/kafka/3.5.1/kafka_2.13-3.5.1.tgz

python -m pip install --user supervisor
mkdir -p ~/supervisor/conf.d
cd ~/supervisor
echo_supervisord_conf > supervisord.conf
echo "[include]
files = conf.d/*.conf" >> ~/supervisor/supervisord.conf
```

## Set up a sample Python web server to run under supervisord
```
mkdir -p ~/htdocs

echo "[program:simplehttpserver]
command=python -m SimpleHTTPServer 8080
directory=/home/vagrant/htdocs
autostart=true
autorestart=true
stdout_logfile=~/supervisor/log/simplehttpserver_stdout.log
stdout_errfile=~/supervisor/log/simplehttpserver_stderr.log
redirect_stderr=true" >> ~/supervisor/conf.d/simplehttpserver.conf

supervisorctl -c ~/supervisor/supervisord.conf reread
supervisorctl -c ~/supervisor/supervisord.conf update
```
## Using supervisor
```
supervisord -c ~/supervisor/supervisord.conf
```

### supervisor status
```
supervisorctl -c ~/supervisor/supervisord.conf status
```
### Stop supervisor jobs
```
supervisorctl -c ~/supervisor/supervisord.conf stop all
```
### Shutdown supervisor
```
supervisorctl -c ~/supervisor/supervisord.conf shutdown
```

## Setting up a cronjob to start a supervisor
### Manually
```
crontab -e
```
Put in the file:
```
@reboot /home/vagrant/.local/bin/supervisord -c /home/vagrant/supervisor/supervisord.conf
```

### Automated
```
(crontab -l 2>/dev/null; echo "@reboot /home/vagrant/.local/bin/supervisord -c /home/vagrant/supervisor/supervisord.conf") | crontab -
```

# Add kafka to supervisor
```
echo "[program:zookeeper]
command=/home/vagrant/kafka_2.13-3.5.1/bin/zookeeper-server-start.sh /home/vagrant/kafka_2.13-3.5.1/config/zookeeper.properties
directory=/home/vagrant/kafka_2.13-3.5.1
autostart=true
autorestart=true
stdout_logfile=~/supervisor/log/zookeeper_stdout.log
stdout_errfile=~/supervisor/log/zookeeper_stderr.log
redirect_stderr=true" > /home/vagrant/supervisor/conf.d/zookeeper.conf

echo "[program:kafka]
command=/home/vagrant/kafka_2.13-3.5.1/bin/kafka-server-start.sh /home/vagrant/kafka_2.13-3.5.1/config/server.properties
directory=/home/vagrant/kafka_2.13-3.5.1
autostart=true
autorestart=true
stdout_logfile=~/supervisor/log/kafka_stdout.log
stdout_errfile=~/supervisor/log/kafka_stderr.log
redirect_stderr=true" > /home/vagrant/supervisor/conf.d/kafka.conf

supervisorctl -c ~/supervisor/supervisord.conf reread
supervisorctl -c ~/supervisor/supervisord.conf update
```

## Testing Kafka
supervisorctl -c ~/supervisor/supervisord.conf stop kafka
#### Create a topic
```
/home/vagrant/kafka_2.13-3.5.1/bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --topic test-topic --partitions 1 --replication-factor 1
```

#### Send a message to the test-topic (interactively)
```
/home/vagrant/kafka_2.13-3.5.1/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic
```

#### Send a message (automation)a
```
echo "hello" | /home/vagrant/kafka_2.13-3.5.1/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic --property "parse.key=true" --property "key.separator"
```

#### Consume messages from the test-topic (interactively)
```
/home/vagrant/kafka_2.13-3.5.1/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test-topic --from-beginning
```

# Make vagrant user supervisord start with the system using systemd
```
sudo vi /etc/systemd/system/supervisord-vagrant.service
```
Put in the file
```
[Unit]
Description=Supervisor Process Control System (as vagrant user)
Documentation=http://supervisord.org
After=network.target

[Service]
ExecStart=/home/vagrant/.local/bin/supervisord -c /home/vagrant/supervisor/supervisord.conf
ExecStop=/home/vagrant/.local/bin/supervisorctl -c /home/vagrant/supervisor/supervisord.conf stop all && /home/vagrant/.local/bin/supervisorctl -c /home/vagrant/supervisor/supervisord.conf shutdown
ExecReload=/home/vagrant/.local/bin/supervisorctl -c /home/vagrant/supervisor/supervisord.conf reload && /home/vagrant/.local/bin/supervisorctl -c /home/vagrant/supervisor/supervisord.conf update
User=vagrant
Group=vagrant
Environment="HOME=/user/vagrant"

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl start supervisord-vagrant
sudo systemctl enable supervisord-vagrant
```
Remember to disable the cronjob by running `crontab -e` as vagrant and commenting the job out (using `#`).
