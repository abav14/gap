# gap
Guide to install GAP Stack on CentOs8.x

Let’s say we have a 3 node setup on which we want to deploy GAP Stack for monitoring and alerting.
Let’s call the nodes as- node1, node2, node3.

Operating system- Cent0S 8

<h2>PART 1- Installation and setup</h2>
<h4>1.	Install Node exporter</h4>


**[on all nodes]**

Run the following script on all the nodes. This will install node exporter that will be running on port 9100. 
~~~shell
yum install net-tools -y
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz -P /tmp
 
cd /tmp
tar xzf node_exporter-1.0.1.linux-amd64.tar.gz
cp node_exporter-1.0.1.linux-amd64/node_exporter /usr/local/bin/
chown root:root /usr/local/bin/node_exporter
 
touch /etc/systemd/system/node_exporter.service
cat << EOF > /etc/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
After=network-online.target
 
[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/local/bin/node_exporter \
    --collector.systemd \
    --collector.textfile \
    --collector.textfile.directory=/var/lib/node_exporter \
    --web.listen-address=0.0.0.0:9100 \
    --web.telemetry-path=/metrics
 
SyslogIdentifier=node_exporter
Restart=always
RestartSec=1
StartLimitInterval=0
 
ProtectHome=yes
NoNewPrivileges=yes
 
ProtectSystem=strict
ProtectControlGroups=true
ProtectKernelModules=true
ProtectKernelTunables=yes
 
[Install]
WantedBy=multi-user.target
EOF
 
systemctl daemon-reload
systemctl enable --now node_exporter.service

netstat -nltp
~~~

You’ll be able to see node exporter running on port 9100.

To check the metrics go to http://<node_ip>:9100/metrics.

<h4>2. Installing Prometheus</h4>

We”ll be installing Prometheus on node2. It will be collecting metrics from all the nodes with the help of node exporter running on all those nodes.

**[on node2]**
~~~shell
cd ~
mkdir prometheus
cd prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.11.2/prometheus-2.11.2.linux-amd64.tar.gz
tar xvzf prometheus-2.11.2.linux-amd64.tar.gz

sudo useradd -rs /bin/false prometheus
cd prometheus-2.11.2.linux-amd64/
sudo cp prometheus promtool /usr/local/bin
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo mkdir /etc/prometheus
sudo cp -R consoles/ console_libraries/ prometheus.yml /etc/prometheus
sudo mkdir -p /data/prometheus
sudo chown -R prometheus:prometheus /data/prometheus /etc/prometheus/*
sudo chown -R prometheus:prometheus /data/
~~~
Prometheus Service file
~~~shell
cat << EOF > /lib/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path="/data/prometheus" \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-admin-api

Restart=always

[Install]
WantedBy=multi-user.target
EOF
~~~
Start and enable the service
~~~shell
sudo systemctl enable --now prometheus
~~~
<h4>3.	Installing AlertManager</h4>

We"ll be installing AlertManager on node2. It will be pushing alerts based on rules written in prometheus.

**[on node2]**
~~~shell
cd
mkdir alertmanager
cd alertmanager
wget https://github.com/prometheus/alertmanager/releases/download/v0.18.0/alertmanager-0.18.0.linux-amd64.tar.gz
tar xvzf alertmanager-0.18.0.linux-amd64.tar.gz
cd alertmanager-0.18.0.linux-amd64/
sudo mv amtool alertmanager /usr/local/bin

sudo mkdir -p /etc/alertmanager
sudo mv alertmanager.yml /etc/alertmanager
sudo mkdir -p /data/alertmanager
sudo useradd -rs /bin/false alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/amtool /usr/local/bin/alertmanager
sudo chown -R alertmanager:alertmanager /data/alertmanager /etc/alertmanager/*
~~~
 AlertManager Service file
~~~shell
cat << EOF > /lib/systemd/system/alertmanager.service
[Unit]
Description=Alert Manager
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=alertmanager
Group=alertmanager
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/data/alertmanager

Restart=always

[Install]
WantedBy=multi-user.target
EOF
~~~
Start and enable the service
~~~shell
sudo systemctl enable --now prometheus
~~~
<h4>4.	Setup Grafana </h4>
We"ll be setting up grafana on node2.

**[on node2]**

~~~shell
cat << EOF >  /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt

yum install grafana
systemctl daemon-reload
systemctl enable --now grafana-server
~~~

Go to browser and access http://<node2_ip>:3000
username=admin
password=admin

