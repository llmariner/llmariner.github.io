---
title: Set up a Playground
linkTitle: "Playground"
description: Set up the playground environment on the Amazon EC2 instance with GPUs.
weight: 10
---

You can easily set up a playground for LLMariner and learn it. In this page, we provision an EC2 instance, build a [Kind](https://kind.sigs.k8s.io/) cluster, and deploy LLMariner and other required components.

{{% alert title="Warn" color="secondary" %}}
Playground environments are for experimentation use only. For a production-ready installation, please refere to the other installation guide.
{{% /alert %}}

Once all the setup completes, you can interact with the LLM service by directly hitting the API endpoints or using [the OpenAI Python library](https://github.com/openai/openai-python).

## Step 1: Install Terraform and Ansible

We use Terraform and Ansible. Follow the links to install if you haven\'t.

-   [Terraform](https://developer.hashicorp.com/terraform/install)
-   [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
-   [kubernetes.core.k8s module for Ansible](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_module.html)

To install `kubernetes.core.k8s` module, run the following command:

``` bash
ansible-galaxy collection install kubernetes.core
```

## Step 2: Clone the LLMariner Repository

We use the Terraform configuration and Ansible playbook in the [LLMariner repository](https://github.com/llmariner/llmariner). Run the following commands to clone the repo and move to the directory where the Terraform configuration file is stored.

``` bash
git clone https://github.com/llmariner/llmariner.git
cd llmariner/provision/aws
```

## Step 3: Run Terraform

First create a `local.tfvars` file for your deployment. Here is an example.

``` terraform
project_name = "<instance-name> (default: "llmariner-demo")"
profile      = "<aws-profile>"

public_key_path  = "</path/to/public_key_path>"
private_key_path = "</path/to/private_key_path>"
ssh_ip_range     = "<ingress CIDR block for SSH (default: "0.0.0.0/0")>"
```

`profile` is an AWS profile that is used to create an EC2 instance. `public_key_path` and `private_key_path` specify an SSH key used to access the EC2 instance.

{{% alert title="Note" color="primary" %}}
See `variables.tf` for other customizable and default values.
{{% /alert %}}

Then, run the following Terraform commands to initialize and create an EC2 instance. This will approximately take 10 minutes.

``` bash
terraform init
terraform apply -var-file=local.tfvars
```

{{% alert title="Note" color="primary" %}}
If you want to run only the Ansible playbook, you can just run `ansible-playbook -i inventory.ini playbook.yml`.
{{% /alert %}}

Once the deployment completes, a Kind cluster is built in the EC2 instance and LLMariner is running in the cluster. It will take another about five minutes for LLMariner to load base models, but you can move to the next step meanwhile.

## Step 4: Set up SSH Connection

You can access the API endpoint and Grafana by establishing SSH port-forwarding.

``` bash
ansible all \
  -i inventory.ini \
  --ssh-extra-args="-L8080:localhost:80 -L8081:localhost:8081" \
  -a "kubectl port-forward -n monitoring service/grafana 8081:80"
```

With the above command, you can hit the API via `http://localhost:8080`. You can directly hit the endpoint via `curl` or other commands, or you can use the [OpenAI Python library](https://github.com/openai/openai-python).

You can also reach Grafana at `http://localhost:8081`. The login username is `admin`, and the password can be obtained with the following command:

``` bash
ansible all \
  -i inventory.ini \
  -a "kubectl get secrets -n monitoring grafana -o jsonpath='{.data.admin-password}'" | tail -1 | base64 --decode; echo
```

## Step 5: Obtain an API Key

To access LLM service, you need an API key. You can download the LLMariner CLI and use that to login the system, and obtain the API key.

{{< include "../../../includes/cli-install.md" >}}

``` bash
# Login. Please see below for the details.
llma auth login

# Create an API key.
llma auth api-keys create my-key
```

`llma auth login` will ask for the endpoint URL and the issuer URL. Please use the default values for them (`http://localhost:8080/v1` and `http://kong-proxy.kong/v1/dex`).

Then the command will open a web browser to login. Please use the following username and the password.

-   Username: `admin@example.com`
-   Password: `password`

The output of `llma auth api-keys create` contains the secret of the created API key. Please save the value in the environment variable to use that in the following step:

``` bash
export LLMARINER_TOKEN=<Secret obtained from llma auth api-keys create>
```

## Step 6: Interact with the LLM Service

There are mainly three ways to interact with the LLM service.

The first option is to use the CLI. Here are example commands:

``` bash
llma models list

llma chat completions create --model google-gemma-2b-it-q4_0 --role user --completion "What is k8s?"
```

``` bash
curl \
  --header "Authorization: Bearer ${LLMARINER_TOKEN}" \
  --header "Content-Type: application/json" \
  http://localhost:8080/v1/models | jq

curl \
  --request POST \
  --header "Authorization: Bearer ${LLMARINER_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{"model": "google-gemma-2b-it-q4_0", "messages": [{"role": "user", "content": "What is k8s?"}]}' \
  http://localhost:8080/v1/chat/completions
```

The second option is to run the `curl` command and hit the API endpoint. Here is an example command for listing all available models and hitting the chat endpoint.

The third option is to use Python. Here is an example Python code for hitting the chat endpoint.

``` python
from os import environ
from openai import OpenAI

client = OpenAI(
  base_url="http://localhost:8080/v1",
  api_key=environ["LLMARINER_TOKEN"]
)

completion = client.chat.completions.create(
  model="google-gemma-2b-it-q4_0",
  messages=[
    {"role": "user", "content": "What is k8s?"}
  ],
  stream=True
)
for response in completion:
  print(response.choices[0].delta.content, end="")
print("\n")
```

Please visit `tutorials`{.interpreted-text role="doc"} to further exercise LLMariner.

## Step 7: Clean up

Run the following command to destroy the EC2 instance.

``` bash
terraform destroy -var-file=local.tfvars
```
