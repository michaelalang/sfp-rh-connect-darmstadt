# Longtime storage Lab

### configuration an S3 Bucket

```
podman volume create minio
podman run -d \
  --name minio \
  --restart unless-stopped \
  -v minio1:/data \
  -p 9000:9000 \
  -p 9001:9001 \
  -e MINIO_DEFAULT_BUCKETS=us-east-1,us-west-1 \
  -e MINIO_ROOT_USER=admin \
  -e MINIO_ROOT_PASSWORD=changeme \
  bitnami/minio:latest
``` 

```
CFG="$(podman volume inspect prometheus1-cfg | jq -r '.[0].Mountpoint')/bucket.yml"
REGION=us-east-1
cat <<EOF> ${CFG}
type: S3
config:
  bucket: ${REGION}
  endpoint: s3.example.com:9000
  access_key: admin
  secret_key: changeme
  insecure: true
EOF

CFG="$(podman volume inspect prometheus2-cfg | jq -r '.[0].Mountpoint')/bucket.yml"
REGION=us-west-1
cat <<EOF> ${CFG}
type: S3
config:
  bucket: ${REGION}
  endpoint: s3.example.com:9000
  access_key: admin
  secret_key: changeme
  insecure: true
EOF
```

* Update DNS to know the new records required

``` 
export IP=# <<< IP of the MinIO Service

cat <<EOF> /etc/dnsmasq.hosts/hosts
127.0.0.4 store1-gateway.example.com
127.0.0.5 store2-gateway.example.com
${IP} s3.example.com
EOF

systemctl restart dnsmasq
``` 

```
export NAME=prometheus1
export CFG="${NAME}-cfg"
podman rm -f ${NAME}-sidecar
podman run --pod=${NAME} -d \
    --restart unless-stopped \
    -v ${CFG}:/etc/prometheus \
    -v ${NAME}:/prometheus \
    --name ${NAME}-sidecar \
    quay.io/thanos/thanos:v0.35.1 \
      sidecar \
      --reloader.config-file=/etc/prometheus/prometheus.yml \
      --tsdb.path /prometheus \
      --objstore.config-file /etc/prometheus/bucket.yml \
      --http-address :9091 \
      --grpc-address :9191 \
      --prometheus.url http://127.0.0.1:9090

podman run -d \
    --net=host \
    --restart unless-stopped \
    -v ${CFG}:/etc/prometheus \
    -v ${NAME}-cache:/data \
    --name ${NAME}-store1-gw \
    quay.io/thanos/thanos:v0.35.1 \
    store \
      --objstore.config-file /etc/prometheus/bucket.yml \
      --http-address 127.0.0.4:9090 \
      --grpc-address 127.0.0.4:9191 

export NAME=prometheus2
export CFG="${NAME}-cfg"
podman rm -f ${NAME}-sidecar
podman run --pod=${NAME} -d \
    --restart unless-stopped \
    -v ${CFG}:/etc/prometheus \
    -v ${NAME}:/prometheus \
    --name ${NAME}-sidecar \
    quay.io/thanos/thanos:v0.35.1 \
      sidecar \
      --reloader.config-file=/etc/prometheus/prometheus.yml \
      --tsdb.path /prometheus \
      --objstore.config-file /etc/prometheus/bucket.yml \
      --http-address :9091 \
      --grpc-address :9191 \
      --prometheus.url http://127.0.0.1:9090

podman run -d \
    --net=host \
    --restart unless-stopped \
    -v ${CFG}:/etc/prometheus \
    -v ${NAME}-cache:/data \
    --name ${NAME}-store2-gw \
    quay.io/thanos/thanos:v0.35.1 \
    store \
      --objstore.config-file /etc/prometheus/bucket.yml \
      --http-address 127.0.0.5:9090 \
      --grpc-address 127.0.0.5:9191 
``` 

* remove the current configure Querier pod

    ```
    podman rm -f querier
    ```

* configure the querier pod with a new Bucket storage gateway

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
        --store store1-gateway.example.com:9191 \
        --store store2-gateway.example.com:9191
    ```

