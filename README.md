# gap
Guide to install GAP Stack on CentOs8.x

Let’s say we have a 3 node setup on which we want to deploy GAP Stack for monitoring and alerting.
Let’s call the nodes as- node1, node2, node3.

Operating system- Cent0S 8

<h3>PART 1- Installation and setup</h3>
1.	Install Node exporter


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

2.	Setup Prometheus
We”ll be installing Prometheus on node2 only. It will be collecting metrics from all the nodes with the help of node exporter running there.
