# Setup the Lab

### install depencendies
```
dnf -y install podman dnsmasq bind-utils
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

IP=$(hostname -I | awk ' { print $1 }')
mkdir /etc/dnsmasq.hosts/
cat <<EOF> /etc/dnsmasq.hosts/hosts
127.0.0.2      prometheus1.example.com
127.0.0.3      prometheus2.example.com
${IP} prometheus.example.com
${IP} node1.dc1.example.com
${IP} node2.dc2.example.com
EOF

systemctl restart dnsmasq
```

# configure the host to use the local dnsmasq 

```
nmcli c mod 'System eth0' ipv4.ignore-auto-dn yes
nmcli c mod 'System eth0' ipv4.dns $(hostname -I | awk ' { print $1 }')
nmcli c up 'System eth0'

host -t srv _prometheus._tcp.dc1.example.com
host -t srv _prometheus._tcp.dc2.example.com
``` 

### create two prometheus instance pods 

``` 
podman pod create -n prometheus1 -p 127.0.0.2:9090:9090 -p 127.0.0.2:9191:9191
podman pod create -n prometheus2 -p 127.0.0.3:9090:9090 -p 127.0.0.3:9191:9191
```

### create two volumes for the prometheus comfigurations

```
podman volume create prometheus1-cfg
podman volume create prometheus2-cfg
```

### create two prometheus configurations 

``` 
CFG="$(podman volume inspect prometheus1-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'> ${CFG}
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

CFG="$(podman volume inspect prometheus2-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'> ${CFG}
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
``` 

### start the prometheus instances

```
export NAME=prometheus1
export CFG="${NAME}-cfg"
podman run --pod=prometheus1 -d \
    --restart unless-stopped \
    -v ${CFG}:/etc/prometheus \
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
export CFG="${NAME}-cfg"
podman run --pod=${NAME} -d \
    --restart unless-stopped \
    -v ${CFG}:/etc/prometheus \
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
export CFG="${NAME}-cfg"
podman run --pod=${NAME} -d \
    --restart unless-stopped \
    -v ${CFG}:/etc/prometheus \
    -v ${NAME}:/prometheus \
    --name ${NAME}-sidecar \
    quay.io/thanos/thanos:v0.35.1 \
      sidecar \
      --reloader.config-file=/etc/prometheus/prometheus.yml \
      --http-address :9091 \
      --grpc-address :9191 \
      --prometheus.url http://127.0.0.1:9090

export NAME=prometheus2
export CFG="${NAME}-cfg"
podman run --pod=${NAME} -d \
    --restart unless-stopped \
    -v ${CFG}:/etc/prometheus \
    -v ${NAME}:/prometheus \
    --name ${NAME}-sidecar \
    quay.io/thanos/thanos:v0.35.1 \
      sidecar \
      --reloader.config-file=/etc/prometheus/prometheus.yml \
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
     --grpc-address $(hostname -I | awk ' { print $1 }'):9191 \
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

# update the prometheus configuration to use DNS service discovery

```
CFG="$(podman volume inspect prometheus1-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'>> ${CFG}
  - job_name: "node"
    dns_sd_configs:
    - names:
        - "_prometheus._tcp.dc1.example.com"
EOF

CFG="$(podman volume inspect prometheus2-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'>> ${CFG}
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
export IP=$(hostname -I | awk ' { print $1 }')
for x in $(seq 3 2 10) ; do 
    echo "srv-host=_prometheus._tcp.dc1.example.com,node${x}.dc1.example.com,9100" >> /etc/dnsmasq.d/srv.conf
    echo "${IP} node${x}.dc1.example.com" >> /etc/dnsmasq.hosts/hosts
done

for x in $(seq 4 2 10) ; do
    echo "srv-host=_prometheus._tcp.dc2.example.com,node${x}.dc2.example.com,9100" >> /etc/dnsmasq.d/srv.conf
    echo "${IP} node${x}.dc2.example.com" >> /etc/dnsmasq.hosts/hosts
done


systemctl restart dnsmasq
``` 

# Custom monitoring implementations 


```
mkdir /home/cloud-user/exportme

cat <<'EOF'> /home/cloud-user/metrics.tmpl
# HELP linux_users_logged_in Total users logged in to the system
# TYPE linux_users_logged_in gauge
linux_users_logged_in ${USERS}
EOF

crontab -e
# add logged in users count
* * * * * export USERS=$(w -hu | wc -l) ; cat /home/cloud-user/metrics.tmpl | envsubst > /home/cloud-user/exportme/metrics

(python -m http.server 8080 -d /home/cloud-user/exportme >/home/cloud-user/exportme/access.log 2>&1 &)
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
CFG="$(podman volume inspect prometheus1-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'>> ${CFG}
  - job_name: "customapp"
    dns_sd_configs:
    - names:
        - "_customapp._tcp.example.com"
EOF

CFG="$(podman volume inspect prometheus2-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'>> ${CFG}
  - job_name: "customapp"
    dns_sd_configs:
    - names:
        - "_customapp._tcp.example.com"
EOF
```
 
```
curl -s prometheus.example.com:9090/api/v1/query \
  -d 'query=sum(linux_users_logged_in) / count(linux_users_logged_in)' | \
  jq -r 
```


## further integrations 

* https://prometheus.io/docs/instrumenting/exporters/
* https://opentelemetry.io/docs/languages/#status-and-releases
