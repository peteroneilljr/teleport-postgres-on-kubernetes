## Demo Self Hosted Postgres and Teleport on Kubernetes


### Create Teleport Certificates
```sh
HOSTNAME=postgres.pon-postgres.svc.cluster.local
KEYNAME=postgres-teleport
NAMESPACE=pon-postgres
tctl auth sign --format=db --host=$HOSTNAME --out=./certs/$KEYNAME --ttl=2190h
kubectl create secret generic postgres-certs  -n $NAMESPACE --from-file=./certs/$KEYNAME.crt --from-file=./certs/$KEYNAME.key --from-file=./certs/$KEYNAME.cas
```

### Create Postgres Credentials
```sh
kubectl create secret generic postgres-secret \
    --from-literal=POSTGRES_USER=developer \
    --from-literal=POSTGRES_DB=teleport_db \
    --from-literal=POSTGRES_PASSWORD='mysecretdeveloperpassword'
```

### Deploy Postgres
```sh
NAMESPACE=pon-postgres
kubectl apply -f deploy-postgres.yaml -n $NAMESPACE
```

### Get Teleport Agent Helm Chart
```sh
helm repo add teleport https://charts.releases.teleport.dev
helm repo update
```

### Create values, token is valid for 1 hour
```sh
PROXY="peter.teleport.sh:443"
TOKEN="$(tctl tokens add --type=db --format=text)"
DB_NAME="k8s-postgres"
DB_URI="postgres.pon-postgres.svc.cluster.local:5432"

cat > values.yaml <<EOF
roles: db
proxyAddr: $PROXY
# Set to false if using Teleport Community Edition
enterprise: true
authToken: $TOKEN
databases:
  - name: $DB_NAME
    uri: $DB_URI
    protocol: postgres
    static_labels:
      env: dev
      deploy: k8s
EOF
```

### Install Teleport Agent
```sh
NAMESPACE=pon-postgres
CHART_NAME="pon-postgres-agent"
helm install $CHART_NAME teleport/teleport-kube-agent \
  --namespace $NAMESPACE \
  --version 16.4.2 \
  -f values.yaml
```
