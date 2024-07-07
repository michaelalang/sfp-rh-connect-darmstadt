# MultiRegion Lab Settup

### configuration second Region

The MultiRegion setup will clone the original Lab and adjust values accordingly so that copy&paste is sufficient.

```
dnf -y install podman dnsmasq bind-utils

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

cat <<'EOF'> /etc/dnsmasq.d/srv.conf
srv-host=_prometheus._tcp.dc3.example.com,node1.dc3.example.com,9100
srv-host=_prometheus._tcp.dc4.example.com,node2.dc4.example.com,9100
EOF

IP=$(hostname -I | awk ' { print $1 }')
mkdir /etc/dnsmasq.hosts/
cat <<EOF> /etc/dnsmasq.hosts/hosts
127.0.0.2      prometheus1.example.com
127.0.0.3      prometheus2.example.com
${IP} prometheus.example.com
${IP} node1.dc3.example.com
${IP} node2.dc4.example.com
EOF

systemctl restart dnsmasq

nmcli c mod 'System eth0' ipv4.ignore-auto-dn yes
nmcli c mod 'System eth0' ipv4.dns $(hostname -I | awk ' { print $1 }')
nmcli c up 'System eth0'

host -t srv _prometheus._tcp.dc3.example.com
host -t srv _prometheus._tcp.dc4.example.com

podman pod create -n prometheus1 -p 127.0.0.2:9090:9090 -p 127.0.0.2:9191:9191
podman pod create -n prometheus2 -p 127.0.0.3:9090:9090 -p 127.0.0.3:9191:9191

podman volume create prometheus1-cfg
podman volume create prometheus2-cfg

CFG="$(podman volume inspect prometheus1-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'> ${CFG}
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: us-east-2
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
    cluster: us-west-2
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']
  - job_name: 'sidecar'
    static_configs:
      - targets: ['127.0.0.1:9091']
EOF

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

CFG="$(podman volume inspect prometheus1-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'>> ${CFG}
  - job_name: "node"
    dns_sd_configs:
    - names:
        - "_prometheus._tcp.dc3.example.com"
EOF

CFG="$(podman volume inspect prometheus2-cfg | jq -r '.[0].Mountpoint')/prometheus.yml"
cat <<'EOF'>> ${CFG}
  - job_name: "node"
    dns_sd_configs:
    - names:
        - "_prometheus._tcp.dc4.example.com"
EOF

export IP=$(hostname -I | awk ' { print $1 }')
for x in $(seq 3 2 10) ; do 
    echo "srv-host=_prometheus._tcp.dc3.example.com,node${x}.dc3.example.com,9100" >> /etc/dnsmasq.d/srv.conf
    echo "${IP} node${x}.dc3.example.com" >> /etc/dnsmasq.hosts/hosts
done

for x in $(seq 4 2 10) ; do
    echo "srv-host=_prometheus._tcp.dc4.example.com,node${x}.dc4.example.com,9100" >> /etc/dnsmasq.d/srv.conf
    echo "${IP} node${x}.dc4.example.com" >> /etc/dnsmasq.hosts/hosts
done

systemctl restart dnsmasq
```

## Federate the two Regions

* **DO NOT** bi-directional. This is not what you are looking for.
  Choose either Reqion1 to Region2 fedration or the other way around

* update the DNS to reference both Virtual Systems as region1 and region2

    ```
    export region1=# << IP of Virtual system 1
    export region2=# << IP of Virtual system 2
    echo "${region1} region1.example.com" >> /etc/dnsmasq.hosts/hosts
    echo "${region2} region2.example.com" >> /etc/dnsmasq.hosts/hosts

    systemctl restart dnsmasq
    ``` 

### Update Region1 (us-east-1 and us-west-1)

* login to your initial Lab virtual System 
* remove the current configure Querier pod

    ```
    podman rm -f querier
    ```

* configure the querier pod with a new endpoint pointing to Region 2

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
        --store prometheus2.example.com:9191 \
        --store region2.example.com:9191
    ```

### Update Region2 (us-east-2 and us-west-2)

* login to your initial Lab virtual System 
* remove the current configure Querier pod

    ```
    podman rm -f querier
    ```

* configure the querier pod with a new endpoint pointing to Region 1

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
        --store prometheus2.example.com:9191 \
        --store region1.example.com:9191
    ```
