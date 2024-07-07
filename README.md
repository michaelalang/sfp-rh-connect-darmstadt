# Setup the Lab

### install depencendies
```
dnf -y install podman dnsmasq
```

### create two prometheus instance pods 

``` 
podman pod create -n prometheus1 -p 127.0.0.2:9090:9090 -p 127.0.0.2:9191:9191
podman pod create -n prometheus2 -p 127.0.0.3:9090:9090 -p 127.0.0.3:9191:9191
```

### create two prometheus conifugrations 

``` 
cat <<'EOF'> prometheus1.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: us-east-1
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']
  - job_name: 'sidecar'
    static_configs:
      - targets: ['127.0.0.1:9091']
EOF

cat <<'EOF'> prometheus2.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: us-west-1
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']
  - job_name: 'sidecar'
    static_configs:
      - targets: ['127.0.0.1:9091']
EOF

chcon -t container_file_t prometheus*.yml
``` 

### start the prometheus instances

```
export NAME=prometheus1
export CFG="${NAME}.yml"
podman run --pod=prometheus1 -d \
    --restart unless-stopped \
    -v $(pwd)/${CFG}:/etc/prometheus/prometheus.yml \
    -v ${NAME}:/prometheus \
    --name ${NAME} \
    quay.io/prometheus/prometheus:latest \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/prometheus \
      --web.listen-address=:9090 \
      --web.external-url=http://prometheus.example.com:9090 \
      --storage.tsdb.min-block-duration=2h \
      --storage.tsdb.max-block-duration=2h \
      --storage.tsdb.retention.size 0 \
      --web.enable-lifecycle \
      --web.enable-admin-api 

export NAME=prometheus2
export CFG="${NAME}.yml"
podman run --pod=${NAME} -d \
    --restart unless-stopped \
    -v $(pwd)/${CFG}:/etc/prometheus/prometheus.yml \
    -v ${NAME}:/prometheus \
    --name ${NAME} \
    quay.io/prometheus/prometheus:latest \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/prometheus \
      --web.listen-address=:9090 \
      --web.external-url=http://prometheus.example.com:9090 \
      --storage.tsdb.min-block-duration=2h \
      --storage.tsdb.max-block-duration=2h \
      --storage.tsdb.retention.size 0 \
      --web.enable-lifecycle \
      --web.enable-admin-api
```

### start the sidecars 

```
export NAME=prometheus1
podman run --pod=${NAME} -d \
    --restart unless-stopped \
    -v $(pwd)/${CFG}:/etc/prometheus/prometheus.yml \
    -v ${NAME}:/prometheus \
    --name ${NAME}-sidecar \
    quay.io/thanos/thanos:v0.35.1 \
      sidecar \
      --http-address :9091 \
      --grpc-address :9191 \
      --prometheus.url http://127.0.0.1:9090

export NAME=prometheus2
podman run --pod=${NAME} -d \
    --restart unless-stopped \
    -v $(pwd)/${CFG}:/etc/prometheus/prometheus.yml \
    -v ${NAME}:/prometheus \
    --name ${NAME}-sidecar \
    quay.io/thanos/thanos:v0.35.1 \
      sidecar \
      --http-address :9091 \
      --grpc-address :9191 \
      --prometheus.url http://127.0.0.1:9090
``` 

### start the frontend

```
podman run -d \
   --restart unless-stopped \
   --net=host \
   --name querier \
   quay.io/thanos/thanos:v0.35.1 \
     query \
     --http-address $(hostname -I | awk ' { print $1 }'):9090 \
     --query.replica-label replica \
     --store prometheus1.example.com:9191 \
     --store prometheus2.example.com:9191
```

### start the node exporter on the host to scrape more real-life values

```
podman run -d \
   --restart unless-stopped \
   --net=host --pid=host \
   -v /:/host:ro,rslave \
   -v /proc:/host/proc:ro,rslave \
   -v /sys:/host/sys:ro,rslave \
   -v /var/run/dbus/system_bus_socket:/var/run/dbus/system_bus_socket \
   --name node-exporter \
   docker.io/prom/node-exporter:latest \
     --collector.processes \
     --collector.systemd \
     --collector.tcpstat \
     --collector.cgroups \
     --collector.interrupts \
     --web.listen-address=:9100 \
     --path.rootfs=/host
```

### setup dnsmasq for the Lab 

```
cat <<'EOF'> /etc/dnsmasq.conf
no-resolv
strict-order
user=dnsmasq
group=dnsmasq
interface=*
domain=example.com
server=8.8.8.8
conf-dir=/etc/dnsmasq.d
hostsdir=/etc/dnsmasq.hosts/
EOF

systemctl enable --now dnsmasq
```

# configure DNS service discovery 

``` 
cat <<'EOF'> /etc/dnsmasq.d/srv.conf
srv-host=_prometheus._tcp.dc1.example.com,node1.dc1.example.com,9100
srv-host=_prometheus._tcp.dc2.example.com,node2.dc2.example.com,9100
EOF

mkdir /etc/dnsmasq.hosts/
cat <<'EOF'> /etc/dnsmasq.hosts/hosts
127.0.0.2      prometheus1.example.com
127.0.0.3      prometheus2.example.com
192.168.192.35 prometheus.example.com
192.168.192.35 node1.dc1.example.com
192.168.192.35 node2.dc2.example.com
EOF

systemctl restart dnsmasq
``` 

# configure the host to use the local dnsmasq 

```
nmcli c mod 'System eth0' ipv4.ignore-auto-dn yes
nmcli c mod 'System eth0' ipv4.dns 192.168.192.35
nmcli c up 'System eth0'

host -t srv _prometheus._tcp.dc1.example.com
host -t srv _prometheus._tcp.dc2.example.com
``` 

# update the prometheus configuration to use DNS service discovery

```
cat <<'EOF'>> prometheus1.yml
  - job_name: "node"
    dns_sd_configs:
    - names:
        - "_prometheus._tcp.dc1.example.com"
EOF

cat <<'EOF'>> prometheus2.yml
  - job_name: "node"
    dns_sd_configs:
    - names:
        - "_prometheus._tcp.dc2.example.com"
EOF
```

```
curl -s prometheus.example.com:9090/api/v1/targets | \
  jq -r '.data.activeTargets|length'
```

# extend the DNS discovery to more targets

```
for x in $(seq 3 2 10) ; do 
    echo "srv-host=_prometheus._tcp.dc1.example.com,node${x}.dc1.example.com,9100" >> /etc/dnsmasq.d/srv.conf
    echo "$(hostname -I | awk ' { print $1 }') node${x}.dc1.example.com" >> /etc/dnsmasq.hosts/hosts
done

for x in $(seq 4 2 10) ; do
    echo "srv-host=_prometheus._tcp.dc2.example.com,node${x}.dc2.example.com,9100" >> /etc/dnsmasq.d/srv.conf
    echo "$(hostname -I | awk ' { print $1 }') node${x}.dc2.example.com" >> /etc/dnsmasq.hosts/hosts
done


systemctl restart dnsmasq
``` 

# Custom monitoring implementations 


```
mkdir /root/exportme

crontab -e
# add logged in users count
* * * * * export USERS=$(w -hu | wc -l) ; cat metrics.tmpl | envsubst > /root/exportme/metrics

python -m http.server 8080 -d /root/exportme
``` 

## integrate your custom monitor into our prometheus

```
cat <<'EOF' > /etc/dnsmasq.d/customapp
srv-host=_customapp._tcp.example.com,node1.dc1.example.com,8080
srv-host=_customapp._tcp.example.com,node2.dc2.example.com,8080
EOF

systemctl restart dnsmasq
```

```
cat <<'EOF'> prometheus1.yml
  - job_name: "customapp"
    dns_sd_configs:
    - names:
        - "_customapp._tcp.example.com"
EOF

cat <<'EOF'> prometheus2.yml
  - job_name: "customapp"
    dns_sd_configs:
    - names:
        - "_customapp._tcp.example.com"
EOF

podman restart prometheus1 prometheus2
```
 
```
curl -s prometheus.example.com:9090/api/v1/query \
  -d 'query=sum(linux_users_logged_in) / count(linux_users_logged_in)' | \
  jq -r 
```


## further integrations 

* https://prometheus.io/docs/instrumenting/exporters/
* https://opentelemetry.io/docs/languages/#status-and-releases