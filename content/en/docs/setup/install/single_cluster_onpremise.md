---
title: Install in a Single On-premise Cluster
linkTitle: "Standalone (on-premise)"
description: Install LLMariner in an on-premise Kubernetes cluster with the standalone mode.
weight: 21
---

This page goes through the concrete steps to install LLMariener on a on-premise K8s cluster (or a local K8s cluster).
You can skip some of the steps if you have already made necessary installation/setup.

{{% alert title="Note" color="primary" %}}
Installation of Postgres, MinIO, SeawoodFS, and Milvus are just example purposes, and
they are not intended for the production usage.
Please configure based on your requirements if you want to use LLMariner for your production environment.
{{% /alert %}}


## Step 1. Install Nvidia GPU Operator

{{< include "../../../includes/install-nvidia-operator.md" >}}

## Step 2. Install an ingress controller

An ingress controller is required to route HTTP/HTTPS requests to the LLMariner components. Any ingress controller works, and you can skip this step if your EKS cluster already has an ingress controller.

Here is an example that installs [Kong](https://konghq.com/) and make the ingress controller:

``` bash
helm repo add kong https://charts.konghq.com
helm repo update

cat <<EOF > kong-values.yaml
proxy:
 type: NodePort
 http:
   hostPort: 80
 tls:
   hostPort: 443
 annotations:
   service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "300"

nodeSelector:
  ingress-ready: "true"

tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Equal
  effect: NoSchedule
- key: node-role.kubernetes.io/master
  operator: Equal
  effect: NoSchedule

fullnameOverride: kong
EOF

helm upgrade --install --wait \
  --namespace kong \
  --create-namespace \
  kong-proxy kong/kong \
  -f kong-values.yaml
```


## Step 3. Install a Postgres database

Run the following to deploy an Postgres deployment:

```bash
export POSTGRES_USER="admin_user"
export POSTGRES_PASSWORD="secret_password"

helm upgrade --install --wait \
  --namespace postgres \
  --create-namespace \
  postgres oci://registry-1.docker.io/bitnamicharts/postgresql \
  --set nameOverride=postgres \
  --set auth.database=ps_db \
  --set auth.username="${POSTGRES_USER}" \
  --set auth.password="${POSTGRES_PASSWORD}"
```

Set the environmental variables so that LLMariner can later access the Postgres database.

``` bash
export POSTGRES_ADDR=postgres.postgres
export POSTGRES_PORT=5432
```


Create a K8s secret in the `llmariner` namespace so that later LLMariner can retrieve the database password from the secret.

```bash
export LLMARINER_NAMESPACE=llmariner
export POSTGRES_SECRET_NAME=postgres

kubectl create namespace "${LLMARINER_NAMESPACE}"
kubectl create secret generic -n "${LLMARINER_NAMESPACE}" "${POSTGRES_SECRET_NAME}" --from-literal=password="${POSTGRES_PASSWORD}"
```

## Step 4. Install an S3-compatible object store

LLMariner requires an S3-compatible object store such as [MinIO](https://min.io/) or [SeaweedfS](https://seaweedfs.com).

First set environmental variables to specify installation configuration:

```bash
# Bucket name and the dummy region.
export S3_BUCKET_NAME=llmariner
export S3_REGION=dummy

# Credentials for accessing the S3 bucket.
export AWS_ACCESS_KEY_ID=llmariner-key
export AWS_SECRET_ACCESS_KEY=llmariner-secret
```

Then install an object store. Here are the example installation commands for MinIO and SeaweedFS.

{{< tabpane text=true >}}
  {{% tab header="**Install**:" disabled=true /%}}
  {{% tab header="MinIO" %}}
```bash
helm upgrade --install --wait \
  --namespace minio \
  --create-namespace \
  minio oci://registry-1.docker.io/bitnamicharts/minio \
  --set auth.rootUser=minioadmin \
  --set auth.rootPassword=minioadmin \
  --set defaultBuckets="${S3_BUCKET_NAME}"

kubectl port-forward -n minio service/minio 9001 &

# Wait until the port-forwarding connection is established.
sleep 5

# Obtain the cookie and store in cookies.txt.
curl \
  http://localhost:9001/api/v1/login \
  --cookie-jar cookies.txt \
  --request POST \
  --header 'Content-Type: application/json' \
  --data @- << EOF
{
  "accessKey": "minioadmin",
  "secretKey": "minioadmin"
}
EOF

# Create a new API key.
curl \
  http://localhost:9001/api/v1/service-account-credentials \
  --cookie cookies.txt \
  --request POST \
  --header "Content-Type: application/json" \
  --data @- << EOF >/dev/null
{
  "name": "LLMariner",
  "accessKey": "$AWS_ACCESS_KEY_ID",
  "secretKey": "$AWS_SECRET_ACCESS_KEY",
  "description": "",
  "comment": "",
  "policy": "",
  "expiry": null
}
EOF

rm cookies.txt

kill %1
```

  {{% /tab %}}
  {{% tab header="SeaweedFS" %}}

```bash
kubectl create namespace seaweedfs

# Create a secret.
# See https://github.com/seaweedfs/seaweedfs/wiki/Amazon-S3-API#public-access-with-anonymous-download for details.
cat <<EOF > s3-config.json
{
  "identities": [
    {
      "name": "me",
      "credentials": [
        {
          "accessKey": "${AWS_ACCESS_KEY_ID}",
          "secretKey": "${AWS_SECRET_ACCESS_KEY}"
        }
      ],
      "actions": [
        "Admin",
        "Read",
        "ReadAcp",
        "List",
        "Tagging",
        "Write",
        "WriteAcp"
      ]
    }
  ]
}
EOF

kubectl create secret generic -n seaweedfs seaweedfs --from-file=s3-config.json
rm s3-config.json

# deploy seaweedfs
cat << EOF | kubectl apply -n seaweedfs -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: seaweedfs-volume
  labels:
    type: local
    app: seaweedfs
spec:
  storageClassName: manual
  capacity:
    storage: 500Mi
  accessModes:
  - ReadWriteMany
  hostPath:
    path: /data/seaweedfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: seaweedfs-volume-claim
  labels:
    app: seaweedfs
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seaweedfs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: seaweedfs
  template:
    metadata:
      labels:
        app: seaweedfs
    spec:
      containers:
      - name: seaweedfs
        image: chrislusf/seaweedfs
        args:
        - server -s3 -s3.config=/etc/config/s3-config.json -dir=/data
        ports:
        - name: master
          containerPort: 9333
          protocol: TCP
        - name: s3
          containerPort: 8333
          protocol: TCP
        volumeMounts:
        - name: seaweedfsdata
          mountPath: /data
        - name: config
          mountPath: /etc/config
      volumes:
      - name: seaweedfsdata
        persistentVolumeClaim:
          claimName: seaweedfs-volume-claim
      - name: config
        secret:
          secretName: seaweedfs
---
apiVersion: v1
kind: Service
metadata:
  name: seaweedfs
  labels:
    app: seaweedfs
spec:
  type: NodePort
  ports:
  - port: 9333
    targetPort: master
    protocol: TCP
    name: master
    nodePort: 31238
  - port: 8333
    targetPort: s3
    protocol: TCP
    name: s3
    nodePort: 31239
  selector:
    app: seaweedfs
EOF

kubectl wait --timeout=60s --for=condition=ready pod -n seaweedfs -l app=seaweedfs

kubectl port-forward -n seaweedfs service/seaweedfs 8333 &

# Wait until the port-forwarding connection is established.
sleep 5

# Create the bucket.
aws --endpoint-url http://localhost:8333 s3 mb s3://${S3_BUCKET_NAME}

kill %1
```
  {{% /tab %}}
{{< /tabpane >}}


Then set environmental variable `S3_ENDPOINT_URL` to the URL of the object store. The URL should be accessible from LLMariner pods that will run on the same cluster.

{{< tabpane text=true >}}
  {{% tab header="**Set up the endpoint URL with**:" disabled=true /%}}
  {{% tab header="MinIO" %}}
```bash
export S3_ENDPOINT_URL=http://minio.minio:9000
```
  {{% /tab %}}
  {{% tab header="SeaweedFs" %}}
```bash
export S3_ENDPOINT_URL=http://seaweedfs.seaweedfs:8333
```
  {{% /tab %}}
{{< /tabpane >}}


Finally create a secret for LLMariner to access the bucket.

```bash
export S3_ACCESS_SECRET_NAME=aws

kubectl create secret generic -n "${LLMARINER_NAMESPACE}" "${S3_ACCESS_SECRET_NAME}" \
  --from-literal=accessKeyId="${AWS_ACCESS_KEY_ID}" \
  --from-literal=secretAccessKey="${AWS_SECRET_ACCESS_KEY}"
```


## Step 5. Install Milvus

Install [Milvus](https://milvus.io/) as it is used a backend vector database for RAG.

```bash
cat << EOF > milvus-values.yaml
cluster:
  enabled: false
etcd:
  enabled: false
pulsar:
  enabled: false
minio:
  enabled: false
  tls:
    enabled: false
extraConfigFiles:
  user.yaml: |+
    etcd:
      use:
        embed: true
      data:
        dir: /var/lib/milvus/etcd
    common:
      storageType: local
EOF

helm repo add zilliztech https://zilliztech.github.io/milvus-helm/
helm repo update
helm upgrade --install --wait \
  --namespace "${LLMARINER_NAMESPACE}" \
  milvus zilliztech/milvus \
  -f milvus-values.yaml
```


## Step 6. Install LLMariner

Run the following command to set up a `values.yaml` and install LLMariner with Helm.

``` bash
# Set the endpoint URL of LLMariner. Please change if you are using a different ingress controller.
export INGRESS_CONTROLLER_URL=http://localhost:8080

cat << EOF | envsubst > llmariner-values.yaml
global:
  # This is an ingress configuration with Kong. Please change if you are using a different ingress controller.
  ingress:
    ingressClassName: kong
    # The URL of the ingress controller. this can be a port-forwarding URL (e.g., http://localhost:8080) if there is
    # no URL that is reachable from the outside of the EKS cluster.
    controllerUrl: "${INGRESS_CONTROLLER_URL}"
    annotations:
      # To remove the buffering from the streaming output of chat completion.
      konghq.com/response-buffering: "false"

  database:
    host: "${POSTGRES_ADDR}"
    port: ${POSTGRES_PORT}
    username: "${POSTGRES_USER}"
    ssl:
      mode: disable
    createDatabase: true

  databaseSecret:
    name: "${POSTGRES_SECRET_NAME}"
    key: password

  objectStore:
    s3:
      endpointUrl: "${S3_ENDPOINT_URL}"
      bucket: "${S3_BUCKET_NAME}"
      region: "${S3_REGION}"

  awsSecret:
    name: "${S3_ACCESS_SECRET_NAME}"
    accessKeyIdKey: accessKeyId
    secretAccessKeyKey: secretAccessKey

dex-server:
  staticPasswords:
  - email: admin@example.com
    # bcrypt hash of the string: $(echo password | htpasswd -BinC 10 admin | cut -d: -f2)
    hash: "\$2a\$10\$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
    username: admin-user
    userID: admin-id

inference-manager-engine:
  model:
    default:
      runtimeName: vllm
      preloaded: true
      resources:
        limits:
          nvidia.com/gpu: 1
    overrides:
      meta-llama/Meta-Llama-3.1-8B-Instruct-q4_0:
        contextLength: 16384
      google/gemma-2b-it-q4_0:
        runtimeName: ollama
        resources:
         limits:
           nvidia.com/gpu: 0
      sentence-transformers/all-MiniLM-L6-v2-f16:
        runtimeName: ollama
        resources:
         limits:
           nvidia.com/gpu: 0

inference-manager-server:
  service:
    annotations:
      # These annotations are only meaningful for Kong ingress controller to extend the timeout.
      konghq.com/connect-timeout: "360000"
      konghq.com/read-timeout: "360000"
      konghq.com/write-timeout: "360000"

job-manager-dispatcher:
  serviceAccount:
    create: false
    name: "${LLMARINER_SERVICE_ACCOUNT_NAME}"
  notebook:
    # Used to set the base URL of the API endpoint. This can be same as global.ingress.controllerUrl
    # if the URL is reachable from the inside cluster. Otherwise you can change this to the
    # to the URL of the ingress controller that is reachable inside the K8s cluster.
    llmarinerBaseUrl: "${INGRESS_CONTROLLER_URL}/v1"

model-manager-loader:
  baseModels:
  - meta-llama/Meta-Llama-3.1-8B-Instruct-q4_0
  - google/gemma-2b-it-q4_0
  - sentence-transformers/all-MiniLM-L6-v2-f16

# Required when RAG is used.
vector-store-manager-server:
  vectorDatabase:
    host: milvus
  llmEngineAddr: ollama-sentence-transformers-all-minilm-l6-v2-f16:11434
EOF

helm upgrade --install \
  --namespace llmariner \
  --create-namespace \
  llmariner oci://public.ecr.aws/cloudnatix/llmariner-charts/llmariner \
  -f llmariner-values.yaml
```

{{% alert title="Note" color="primary" %}}
Starting from Helm v3.8.0, the OCI registry is supported by default. If you are using an older version, please upgrade to v3.8.0 or later. For more details, please refer to [Helm OCI-based registries](https://helm.sh/docs/topics/registries/).
{{% /alert %}}

{{% alert title="Note" color="primary" %}}
If you are getting a 403 forbidden error, please try `docker logout public.ecr.aws`. Please see [AWS document](https://docs.aws.amazon.com/AmazonECR/latest/public/public-troubleshooting.html) for more details.
{{% /alert %}}

If you would like to install only the control-plane components or the worker-plane components, please see `multi_cluster_deployment`{.interpreted-text role="doc"}.

## Step 7. Verify the installation

{{< include "../../../includes/verify-installation.md" >}}

## Optional: Monitor GPU utilization

{{< include "../../../includes/optional-setup-monitoring.md" >}}

## Optional: Enable TLS

{{< include "../../../includes/optional-setup-tls.md" >}}
