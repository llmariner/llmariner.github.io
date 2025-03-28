---
title: Install in a Single EKS Cluster
linkTitle: "Standalone (EKS)"
description: Install LLMariner in an EKS cluster with the standalone mode.
weight: 20
---

This page goes through the concrete steps to create an EKS cluster, create necessary resources, and install LLMariner. You can skip some of the steps if you have already made necessary installation/setup.

## Step 1. Provision an EKS cluster

### Step 1.1. Create a new cluster with Karpenter

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

### Step 1.2. Provision GPU nodes

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

### Step 1.3. Install Nvidia GPU Operator

{{< include "../../../includes/install-nvidia-operator.md" >}}

### Step 1.4. Install an ingress controller

{{< include "../../../includes/install-kong.md" >}}

## Step 2. Create an RDS instance

We will create an RDS in the same VPC as the EKS cluster so that it can be reachable from the LLMariner components. Here are example commands for creating a DB subnet group:

``` bash
export DB_SUBNET_GROUP_NAME="llmariner-demo-db-subnet"
export EKS_SUBNET_IDS=$(aws eks describe-cluster --name "${CLUSTER_NAME}" | jq '.cluster.resourcesVpcConfig.subnetIds | join(" ")' --raw-output)
export EKS_SUBNET_ID0=$(echo ${EKS_SUBNET_IDS} | cut -d' ' -f1)
export EKS_SUBNET_ID1=$(echo ${EKS_SUBNET_IDS} | cut -d' ' -f2)

aws rds create-db-subnet-group \
  --db-subnet-group-name "${DB_SUBNET_GROUP_NAME}" \
  --db-subnet-group-description "LLMariner Demo" \
  --subnet-ids "${EKS_SUBNET_ID0}" "${EKS_SUBNET_ID1}"
```
and an RDS instance:

``` bash
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

{{% alert title="Note" color="primary" %}}
LLMariner will create additional databases on the fly for each API service (e.g., `job_manager`, `model_manager`). You can see all created databases by running `SELECT count(datname) FROM pg_database;`.
{{% /alert %}}

## Step 3. Create an S3 bucket

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
export LLMARINER_NAMESPACE=llmariner
export LLMARINER_POLICY="LLMarinerPolicy"
export LLMARINER_SERVICE_ACCOUNT_NAME="llmariner"
export LLMARINER_ROLE="LLMarinerRole"

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

aws iam create-policy --policy-name "${LLMARINER_POLICY}" --policy-document file://policy.json

eksctl create iamserviceaccount \
  --name "${LLMARINER_SERVICE_ACCOUNT_NAME}" \
  --namespace "${LLMARINER_NAMESPACE}" \
  --cluster "${CLUSTER_NAME}" \
  --role-name "${LLMARINER_ROLE}" \
  --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${LLMARINER_POLICY}" --approve
```

## Step 4. Install Milvus

Install [Milvus](https://milvus.io/) as it is used a backend vector database for RAG.

Milvus creates Persistent Volumes. Follow <https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html> and install EBS CSI driver.

``` bash
export EBS_CSI_DRIVER_ROLE="AmazonEKS_EBS_CSI_DriverRole"

eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster "${CLUSTER_NAME}" \
  --role-name "${EBS_CSI_DRIVER_ROLE}" \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

eksctl create addon \
  --cluster "${CLUSTER_NAME}" \
  --name aws-ebs-csi-driver \
  --version latest \
  --service-account-role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${EBS_CSI_DRIVER_ROLE}" \
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

pulsarv3:
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
  name: "${LLMARINER_SERVICE_ACCOUNT_NAME}"

externalS3:
  enabled: true
  host: s3.us-east-1.amazonaws.com
  port: 443
  useSSL: true
  bucketName: "${MILVUS_S3_BUCKET_NAME}"
  region: us-east-1
  useIAM: true
  cloudProvider: aws
  iamEndpoint: ""
  logLevel: info
EOF

helm repo add zilliztech https://zilliztech.github.io/milvus-helm/
helm repo update
helm upgrade --install --wait \
  --namespace "${LLMARINER_NAMESPACE}" \
  --create-namespace \
  milvus zilliztech/milvus \
  -f milvus-values.yaml
```

Please see the [Milvus installation document](https://milvus.io/docs/install-overview.md) and the [Helm chart](https://artifacthub.io/packages/helm/milvus/milvus) for other installation options.

Set the environmental variables so that LLMariner can later access the Postgres database.

``` bash
export MILVUS_ADDR=milvus.llmariner.svc.cluster.local
```


## Step 5. Install LLMariner

Run the following command to set up a `values.yaml` and install LLMariner with Helm.

``` bash
# Set the endpoint URL of LLMariner. Please change if you are using a different ingress controller.
export INGRESS_CONTROLLER_URL=http://$(kubectl get services -n kong kong-proxy-kong-proxy  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export POSTGRES_SECRET_NAME="db-secret"

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
      endpointUrl: ""

prepare:
  database:
    createSecret: true
    secret:
      password: "${POSTGRES_PASSWORD}"

dex-server:
  staticPasswords:
  - email: admin@example.com
    # bcrypt hash of the string: $(echo password | htpasswd -BinC 10 admin | cut -d: -f2)
    hash: "\$2a\$10\$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
    username: admin-user
    userID: admin-id

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
    llmarinerBaseUrl: "${INGRESS_CONTROLLER_URL}/v1"

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
    host: "${MILVUS_ADDR}"
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

## Step 6. Verify the installation

{{< include "../../../includes/verify-installation.md" >}}

## Optional: Monitor GPU utilization

{{< include "../../../includes/optional-setup-monitoring.md" >}}

## Optional: Enable TLS

{{< include "../../../includes/optional-setup-tls.md" >}}
