# GAP Stack 

Guide to install GAP Stack on CentOs8.x

Let’s say we have a 3 node setup on which we want to deploy GAP Stack for monitoring and alerting.
Let’s call the nodes as- node1, node2, node3.

Operating system- CentOS Linux release 8.1.1911

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

To reset grafana password in future use the following command.

~~~shell
grafana-cli admin reset-admin-password admin
~~~
Now on Grafana dasboard navigate to Setting->Data Sources-> Add data source -> Prometheus.
Add http://localhost:9090 in URL and add the following data source like this. 

![Screenshot (95)](https://user-images.githubusercontent.com/28900470/118795927-baa80980-b8b8-11eb-8471-5b47b13e2a09.png)

Now all the components are installed let's see how we can configure and customize them.

<h2>PART 2- Configuring the setup.</h2>

<h4> Creating custom node-exporter metrics</h4>

In this part we”ll learn how to create custom node-exporter rules.
Let’s say we have to run a script in crontab and pushed the output of it to node-exporter on regular intervals and create a rule for that value in Prometheus.

Suppose we have a script script.sh on **node1** that has a variable **some_exitval** based on some logic and we want this variable to be pushed to node-exporter. We”ll run the script in crontab according to our requirement.

The script looks like this.

**[on node1]**
~~~shell
vi script.sh

EXITVAL=0
# while installing node-exporter this is the default value of textfile-collector
TEXTFILE_COLLECTOR_DIR=/var/lib/node_exporter/
cat << EOF > "$TEXTFILE_COLLECTOR_DIR/some_prov.prom.$$"
some_exitval $((EXITVAL))
EOF
mv "$TEXTFILE_COLLECTOR_DIR/some_prov.prom.$$" \
  "$TEXTFILE_COLLECTOR_DIR/some_prov.prom"
~~~
Now each file in /var/lib/node-exporter/ that has .prom extension will be pushed to node-exporter.
In this case we”ll see a variable in node-exporter metrics named nova_exitval that has value 0. Similarly we can create custom metrics for other scripts.

<h4> Adding targets to Prometheus and creating rules</h4>

Let’s create and understand prometheus.yml file.

~~~shell
cd /etc/prometheus/
vi prometheus.yml
~~~

~~~yaml

# my global config
global:
  scrape_interval:     60s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 60s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - localhost:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'group1'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['<node1_ip>:9100']

  - job_name: 'group2'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['<node2_ip>:9100','<node3_ip>:9100']
~~~

We have created 2 jobs and put different targets in them. Similarly any no. of jobs and targets can be added accordingly. This will be used in referring in rules.yml file. 

Also in the alerting section we have added localhost:9093 because AlertManager is running on same node at port 9093.

Let’s create and understand rules.yml file.

~~~shell
cd /etc/prometheus/
vi rules.yml
~~~

~~~yaml
groups:
- name: Power-On Status
  rules:
  - alert: HostDown-group1
    expr: up{job="group1"}==0
    for: 1m
    labels:
      severity: critical
      source: 1
    annotations:
      summary: group 1 not-active

  - alert: HostDown-group2
    expr: up{job="group2"}==0
    for: 1m
    labels:
      severity: critical
      source: 2
    annotations:
      summary: group 2 not-active
    
- name: Services status
  rules:
  - alert: ChronyServiceDownGroup1
    expr: node_systemd_unit_state{job="group1",name="chronyd.service",state="active"}==0
    for: 60m
    labels:
      severity: error
      source: 3
    annotations:
      summary: chrony service not-active on group1

  - alert: HaproxyServiceDownNode3
    expr: node_systemd_unit_state{instance="<node3_ip>:9100",job="group2",name="haproxy.service",state="active"}==0
    for: 60m
    labels:
      severity: error
      source: 4
    annotations:
      summary: haproxy service is not active on node3
 
  - alert: custom_parameter check
    expr: nova_exitval{instance="<node1_ip>:9100",job="group1"}!=0
    for: 10m
    labels:
      severity: error
      source: 5
    annotations:
      summary: nova_exital is not zero

- name: Warnings
  rules:
  - alert: CPU_Used_Threshold
    expr: (100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100))>=50
    for: 5m
    labels:
      severity: warning
      source: 6
    annotations:
      summary: host cpu usage is high
      
  - alert: RAM_Used_Threshold
    expr: (100 * (1 - ((avg_over_time(node_memory_MemFree_bytes[1h]) + avg_over_time(node_memory_Cached_bytes[1h]) + avg_over_time(node_memory_Buffers_bytes[1h])) / avg_over_time(node_memory_MemTotal_bytes[1h]))))>=70
    for: 5m
    labels:
      severity: warning
      source: 14
    annotations:
      summary: host memory usage is high

~~~

In rules.yml we are making use of job filter as defined in prometheus.yml. We have first written rule to check if group 1 and group 2 nodes are up. Then we have written a rule to check chrony service on group1. Then we have used combination of instance and job filter to check haproxy service on node3. 

We have also made a rule to check custom parameter i.e. nova_exitval. The script was running on node1 in crontab and value is pushed to node_exporter so we have mentioned node1_ip and 9100 port to get that value. If the value is not zero then a alert will be sent to AlertManager.

In the end we have created a rule to check the cpu and memory usage percentages on all nodes if the value is more than 50 (or 70) then a alert would be sent.

We have also used 2 labels here 'severity' and 'severity' that can be utilized in inhibition in AlertManager. Inhibition is a concept of suppressing notifications for certain alerts if certain other alerts are already firing. 

<h4> Setup AlertManager </h4>

~~~shell
cd /etc/alertmanager/
vi alertmanager.yml
~~~

~~~yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname','severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'email'
receivers:
- name: 'email'
  email_configs:
  - to: '<email_to_send_alerts_to>'
    from: '<from_email>'
    smarthost: <smarthost>:25
inhibit_rules:
  - source_match:
      severity: 'Critical'
    target_match:
      severity: 'Error'
    equal: ['source']   #this should be same label:value in both source and target alert

  - source_match:
      severity: 'Critical'
    target_match:
      severity: 'Warning'
    equal: ['source']
~~~

This is a sample alertmanager.yml. In this rules inhibition is done. If a 'Critical' alert is fired then 'Error' and 'Warning' will not be fired if they have same 'source' label match in rules.yml.

To check alerts go to http://<node2_ip>:9093. This type of alerts can be seen. If email is configured then following alerts will be pushed to email based on the intervals defined in rules.yml.


![Screenshot (97)](https://user-images.githubusercontent.com/28900470/118823068-509e5d00-b8d6-11eb-9b05-96d94936131b.png)

