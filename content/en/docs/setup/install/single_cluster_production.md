---
title: Install in a Single Cluster
linkTitle: "Standalone"
description: Install LLMariner in a single Kubernetes cluster.
weight: 20
---

We provide a Helm chart for installing LLMariner. You can obtain the Helm chart from our repository and install.

``` bash
# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws

helm upgrade --install \
  --namespace <namespace> \
  --create-namespace \
  llmariner oci://public.ecr.aws/cloudnatix/llmariner-charts/llmariner \
  --values <values.yaml>
```

Once installation completes, you can interact with the API endpoint using the [OpenAI Python library](https://github.com/openai/openai-python), running our CLI, or directly hitting the endpoint. To download the CLI, run:

{{< include "../../../includes/cli-install.md" >}}

## EKS Installation

Let\'s go through the details of the installation. Here we use EKS as a target K8s cluster, but it can be AKS, GKE, or any other K8s cluster.

### Prerequisites

LLMariner requires the following resources:

-   [Nvidia GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html)
-   Ingress controller (to route API requests)
-   SQL database (to store jobs/models/files metadata)
-   S3-compatible object store (to store training files and models)
-   [Milvus](https://milvus.io/) (for RAG, optional)

LLMariner can process inference requests on CPU nodes, but it can be best used with GPU nodes. Nvidia GPU Operator is required to install the device plugin and make GPUs visible in the K8s cluster.

Preferably the ingress controller should have a DNS name or an IP that is reachable from the outside of the EKS cluster. If not, you can rely on port-forwarding to reach the API endpoints.

{{% alert title="Note" color="primary" %}}
When port-forwarding is used, the same port needs to be used consistently as the port number will be included the OIDC issuer URL. We will explain details later.
{{% /alert %}}

You can provision RDS and S3 in AWS, or you can deploy Postgres and [MinIO](https://min.io/) inside your EKS cluster. The following permissions are required for S3:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
       "Action": [
         "s3:PutObject",
         "s3:GetObject",
         "s3:DeleteObject",
         "s3:ListBucket"
       ],
       "Resource": [
         "arn:aws:s3:::<bucket name>",
         "arn:aws:s3:::<bucket name>/*"
       ]
     }
   ]
 }
```

The rest of the section go through concrete steps to create an EKS cluster, create necessary resources, and install LLMariner. You can skip some of the steps if you have already made necessary installation/setup.

### Step 1. Provision an EKS cluster

#### Step 1.1. Create a new cluster with Karpenter

Either follow the [Karpenter getting started guide](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/) and create an EKS cluster with Karpenter, or run the following simplified installation steps.

``` bash
export CLUSTER_NAME="llmariner-demo"
export AWS_DEFAULT_REGION="us-east-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"

export KARPENTER_NAMESPACE="kube-system"
export KARPENTER_VERSION="1.0.1"
export K8S_VERSION="1.30"
export TEMPOUT="$(mktemp)"

curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > "${TEMPOUT}" \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"

eksctl create cluster -f - <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "${K8S_VERSION}"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}

iam:
  withOIDC: true
  podIdentityAssociations:
  - namespace: "${KARPENTER_NAMESPACE}"
    serviceAccountName: karpenter
    roleName: ${CLUSTER_NAME}-karpenter
    permissionPolicyARNs:
    - arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}

iamIdentityMappings:
- arn: "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes

managedNodeGroups:
- instanceType: m5.large
  amiFamily: AmazonLinux2
  name: ${CLUSTER_NAME}-ng
  desiredCapacity: 2
  minSize: 1
  maxSize: 10
addons:
- name: eks-pod-identity-agent
EOF

# Create the service linked role if it does not exist. Ignore an already-exists error.
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

# Logout of helm registry to perform an unauthenticated pull against the public ECR.
helm registry logout public.ecr.aws

# Deploy Karpenter.
helm upgrade --install --wait \
  --namespace "${KARPENTER_NAMESPACE}" \
  --create-namespace \
  karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi
```

#### Step 1.2. Provision GPU nodes

Once Karpenter is installed, we need to create an `EC2NodeClass` and a `NodePool` so that GPU nodes are provisioned. We configure `blockDeviceMappings` in the `EC2NodeClass` definition so that nodes have sufficient local storage to store model files.

``` bash
export GPU_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2-gpu/recommended/image_id --query Parameter.Value --output text)"

cat << EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: kubernetes.io/os
        operator: In
        values: ["linux"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand"]
      - key: karpenter.k8s.aws/instance-family
        operator: In
        values: ["g5"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 720h
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  role: "KarpenterNodeRole-${CLUSTER_NAME}"
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: "${CLUSTER_NAME}"
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: "${CLUSTER_NAME}"
  amiSelectorTerms:
  - id: "${GPU_AMI_ID}"
  blockDeviceMappings:
  - deviceName: /dev/xvda
    ebs:
      deleteOnTermination: true
      encrypted: true
      volumeSize: 256Gi
      volumeType: gp3
EOF
```

#### Step 1.3. Install Nvidia GPU Operator

Nvidia GPU Operator is required to install the device plugin and make GPU resources visible in the K8s cluster. Run:

``` bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm upgrade --install --wait \
  --namespace nvidia \
  --create-namespace \
  gpu-operator nvidia/gpu-operator \
  --set cdi.enabled=true \
  --set driver.enabled=false \
  --set toolkit.enabled=false
```

#### Step 1.4. Install an ingress controller

An ingress controller is required to route HTTP/HTTPS requests to the LLMariner components. Any ingress controller works, and you can skip this step if your EKS cluster already has an ingress controller.

Here is an example that installs [Kong](https://konghq.com/) and make the ingress controller reachable via AWS loadbalancer:

``` bash
helm repo add kong https://charts.konghq.com
helm repo update
helm upgrade --install --wait \
  --namespace kong \
  --create-namespace \
  kong-proxy kong/kong \
  --set proxy.annotations.service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout=300 \
  --set ingressController.installCRDs=false \
  --set fullnameOverride=false
```

### Step 2. Create an RDS instance

We will create an RDS in the same VPC as the EKS cluster so that it can be reachable from the LLMariner components. Here is an example command for creating a DB subnet group and an RDS instance.

``` bash
export DB_SUBNET_GROUP_NAME="llmariner-demo-db-subnet"
export EKS_SUBNET_IDS=$(aws eks describe-cluster --name "${CLUSTER_NAME}" | jq '.cluster.resourcesVpcConfig.subnetIds | join(" ")' --raw-output)
export EKS_SUBNET_ID0=$(echo ${EKS_SUBNET_IDS} | cut -d' ' -f1)
export EKS_SUBNET_ID1=$(echo ${EKS_SUBNET_IDS} | cut -d' ' -f2)

aws rds create-db-subnet-group \
  --db-subnet-group-name "${DB_SUBNET_GROUP_NAME}" \
  --db-subnet-group-description "LLMariner Demo" \
  --subnet-ids "${EKS_SUBNET_ID0}" "${EKS_SUBNET_ID1}"

export DB_INSTANCE_ID="llmariner-demo"
export POSTGRES_USER="admin_user"
export POSTGRES_PASSWORD="secret_password"
export EKS_SECURITY_GROUP_ID=$(aws eks describe-cluster --name "${CLUSTER_NAME}" | jq '.cluster.resourcesVpcConfig.clusterSecurityGroupId' --raw-output)

aws rds create-db-instance \
  --db-instance-identifier "${DB_INSTANCE_ID}" \
  --db-instance-class db.t3.small \
  --engine postgres \
  --allocated-storage 10 \
  --storage-encrypted \
  --master-username "${POSTGRES_USER}" \
  --master-user-password "${POSTGRES_PASSWORD}" \
  --vpc-security-group-ids "${EKS_SECURITY_GROUP_ID}" \
  --db-subnet-group-name "${DB_SUBNET_GROUP_NAME}"
```

You can run the following command to check the provisioning status.

``` bash
aws rds describe-db-instances --db-instance-identifier "${DB_INSTANCE_ID}" | jq '.DBInstances[].DBInstanceStatus'
```

Once the RDS instance is fully provisioned and its status becomes `available`, obtain the endpoint information for later use.

``` bash
export POSTGRES_ADDR=$(aws rds describe-db-instances --db-instance-identifier "${DB_INSTANCE_ID}" | jq '.DBInstances[].Endpoint.Address' --raw-output)
export POSTGRES_PORT=$(aws rds describe-db-instances --db-instance-identifier "${DB_INSTANCE_ID}" | jq '.DBInstances[].Endpoint.Port' --raw-output)
```

You can verify if the DB instance is reachable from the EKS cluster by running the `psql` command:

``` bash
kubectl run psql --image jbergknoff/postgresql-client --env="PGPASSWORD=${POSTGRES_PASSWORD}" -- -h "${POSTGRES_ADDR}" -U "${POSTGRES_USER}" -p "${POSTGRES_PORT}" -d template1 -c "select now();"
kubectl logs psql
kubectl delete pods psql
```

If `psql` can successfully connect to the RDS instance, create a K8s secret in the `llmariner` namespace so that later LLMariner can retrieve the database password from the secret.

``` bash
export LLMARINER_NAMESPACE=llmariner
export POSTGRES_SECRET_NAME=postgres

kubectl create namespace "${LLMARINER_NAMESPACE}"
kubectl create secret generic -n "${LLMARINER_NAMESPACE}" "${POSTGRES_SECRET_NAME}" --from-literal=password="${POSTGRES_PASSWORD}"
```

{{% alert title="Note" color="primary" %}}
LLMariner will create additional databases on the fly for each API service (e.g., `job_manager`, `model_manager`). You can see all created databases by running `SELECT count(datname) FROM pg_database;`.
{{% /alert %}}

### Step 3. Create an S3 bucket

We will create an S3 bucket where model files are stored. Here is an example

``` bash
# Please change the bucket name to something else.
export S3_BUCKET_NAME="llmariner-demo"
export S3_REGION="us-east-1"

aws s3api create-bucket --bucket "${S3_BUCKET_NAME}" --region "${S3_REGION}"
```

If you want to set up Milvus for RAG, please create another S3 bucket for Milvus:

``` bash
# Please change the bucket name to something else.
export MILVUS_S3_BUCKET_NAME="llmariner-demo-milvus"

aws s3api create-bucket --bucket "${MILVUS_S3_BUCKET_NAME}" --region "${S3_REGION}"
```

Pods running in the EKS cluster need to be able to access the S3 bucket. We will create an [IAM role for service account](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) for that.

``` bash
cat << EOF | envsubst > policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::${S3_BUCKET_NAME}/*",
        "arn:aws:s3:::${S3_BUCKET_NAME}",
        "arn:aws:s3:::${MILVUS_S3_BUCKET_NAME}/*",
        "arn:aws:s3:::${MILVUS_S3_BUCKET_NAME}"
      ]
    }
  ]
}
EOF

export LLMARINER_POLICY="LLMarinerPolicy"
aws iam create-policy --policy-name "${LLMARINER_POLICY}" --policy-document file://policy.json

export LLMARINER_SERVICR_ACCOUNT_NAME="llmariner"
eksctl create iamserviceaccount \
  --name "${LLMARINER_SERVICR_ACCOUNT_NAME}" \
  --namespace "${LLMARINER_NAMESPACE}" \
  --cluster "${CLUSTER_NAME}" \
  --role-name "LLMarinerRole" \
  --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${LLMARINER_POLICY}" --approve
```

### Step 4. Install Milvus

Install [Milvus](https://milvus.io/) as it is used a backend vector database for RAG.

Milvus creates Persistent Volumes. Follow <https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html> and install EBS CSI driver.

``` bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster "${CLUSTER_NAME}" \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

eksctl create addon \
  --cluster "${CLUSTER_NAME}" \
  --name aws-ebs-csi-driver \
  --version latest \
  --service-account-role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole" \
  --force
```

Then install the Helm chart. Milvus requires access to the S3 bucket. To use the same service account created above, we deploy Milvus in the same namespace as LLMariner.

``` bash
cat << EOF | envsubst > milvus-values.yaml
cluster:
  enabled: false

etcd:
  replicaCount: 1
  persistence:
    storageClass: gp2 # Use gp3 if available

pulsar:
  enabled: false

minio:
  enabled: false

standalone:
  persistence:
    persistentVolumeClaim:
      storageClass: gp2 # Use gp3 if available
      size: 10Gi

serviceAccount:
  create: false
  name: "${LLMARINER_SERVICR_ACCOUNT_NAME}"

externalS3:
  enabled: true
  host: s3.us-east-1.amazonaws.com
  port: 443
  useSSL: true
  bucketName: "${MILVUS_S3_BUCKET_NAME}"
  useIAM: true
  cloudProvider: aws
  iamEndpoint: ""
  logLevel: info
EOF

helm repo add zilliztech https://zilliztech.github.io/milvus-helm/
helm repo update
helm upgrade --install --wait \
  --namespace "${LLMARINER_NAMESPACE}" \
  milvus zilliztech/milvus \
  -f milvus-values.yaml
```

Please see the [Milvus installation document](https://milvus.io/docs/install-overview.md) and the [Helm chart](https://artifacthub.io/packages/helm/milvus/milvus) for other installation options.

### Step 5. Install LLMariner

``` bash
# Set the endpoint URL of LLMariner. Please change if you are using a different ingress controller.
export INGRESS_CONTROLLER_URL=http://$(kubectl get services -n kong kong-proxy-kong-proxy  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

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
      mode: require
    createDatabase: true

  databaseSecret:
    name: "${POSTGRES_SECRET_NAME}"
    key: password

  objectStore:
    s3:
      bucket: "${S3_BUCKET_NAME}"
      region: "${S3_REGION}"

file-manager-server:
  serviceAccount:
    create: false
    name: "${LLMARINER_SERVICE_ACCOUNT_NAME}"

inference-manager-engine:
  serviceAccount:
    create: false
    name: "${LLMARINER_SERVICE_ACCOUNT_NAME}"
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
    llmarinerBaseUrl: "${INGRESS_CONTROLLER_URL}"/v1

model-manager-loader:
  serviceAccount:
    create: false
    name: "${LLMARINER_SERVICE_ACCOUNT_NAME}"
  baseModels:
  - meta-llama/Meta-Llama-3.1-8B-Instruct-q4_0
  - google/gemma-2b-it-q4_0
  - sentence-transformers/all-MiniLM-L6-v2-f16

# Required when RAG is used.
vector-store-manager-server:
  serviceAccount:
    create: false
    name: "${LLMARINER_SERVICE_ACCOUNT_NAME}"
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

### Step 6. Verify the installation

You can verify the installation by sending sample chat completion requests.

``` bash
echo "This is your endpoint URL: ${INGRESS_CONTROLLER_URL}/v1"

llma auth login
# Type the above endpoint URL.

llma models list

llma chat completions create --model google-gemma-2b-it-q4_0 --role user --completion "what is k8s?"

llma chat completions create --model meta-llama-Meta-Llama-3.1-8B-Instruct-q4_0 --role user --completion "hello"
```

### Optional: Monitor GPU utilization

If you would like to install Prometheus and Grafana to see GPU utilization, run:

``` bash
# Add Prometheus
cat <<EOF > prom-scrape-configs.yaml
- job_name: nvidia-dcgm
  scrape_interval: 5s
  static_configs:
  - targets: ['nvidia-dcgm-exporter.nvidia.svc:9400']
- job_name: inference-manager-engine-metrics
  scrape_interval: 5s
  static_configs:
  - targets: ['inference-manager-server-http.llmariner.svc:8083']
EOF
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install --wait \
  --namespace monitoring \
  --create-namespace \
  --set-file extraScrapeConfigs=prom-scrape-configs.yaml \
  prometheus prometheus-community/prometheus

# Add Grafana with DCGM dashboard
cat <<EOF > grafana-values.yaml
datasources:
 datasources.yaml:
   apiVersion: 1
   datasources:
   - name: Prometheus
     type: prometheus
     url: http://prometheus-server
     isDefault: true
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: 'default'
      type: file
      disableDeletion: true
      editable: true
      options:
        path: /var/lib/grafana/dashboards/standard
dashboards:
  default:
    nvidia-dcgm-exporter:
      gnetId: 12239
      datasource: Prometheus
EOF
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install --wait \
  --namespace monitoring \
  --create-namespace \
  -f grafana-values.yaml \
  grafana grafana/grafana
```

### Optional: Enable TLS

First follow the [cert-manager installation document](https://cert-manager.io/docs/installation/) and install cert-manager to your K8s cluster if you don't have one. Then create a `ClusterIssuer` for your domain. Here is an example manifest that uses Let\'s Encrypt.

``` yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: user@mydomain.com
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - http01:
       ingress:
          ingressClassName: kong
    - selector:
        dnsZones:
        - llm.mydomain.com
      dns01:
        ...
```

Then you can add the following to `values.yaml` of LLMariner to enable TLS.

``` yaml
global:
  ingress:
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt
    tls:
      hosts:
      - api.llm.mydomain.com
      secretName: api-tls
```

The ingresses created from the Helm chart will have the following annotation and spec:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
...
spec:
  tls:
  - hosts:
    - api.llm.mydomain.com
    secretName: api-tls
  ...
```
