# Software Infrastructure

By default, this infrastructure requires internet access for installation. However, all components also support deployment in an air-gapped environment. We have experience working in air-gapped setups and can assist you with the configuration.

## Components provisioned with Ansible

All our components are managed as code. Our goal is to manage the entire infrastructure using code, eliminating manual steps (except for OS installation, which may depend on your hardware provider).

The initial cluster bootstrap components are provisioned with Ansible.

### OS - Ubuntu

For the base operating system, we use Ubuntu. Most components are compatible with any modern Linux distribution, but the Nvidia GPU operator and drivers support only a limited set of distributions.

At the time of writing, Nvidia officially supports **Ubuntu 22.04** as the most up-to-date version.

### Linux Setup

Each Linux machine is provisioned manually or automatically, depending on your hardware provider, with Ubuntu 22.04 Server Edition.

We do not recommend using the minimal installation variant, as it lacks commonly used packages such as curl and vim.

Do not use desktop editions for Linux.

### SSH

We use SSH to access servers and provision software packages via Ansible. While other tools like Salt or Puppet can be used, we prefer Ansible for its minimal footprint, requiring only SSH.

For increased isolation, SSH listens only on an internal out-of-band management (OOBM) network. We use a VPN to connect from the outside to the OOBM network.

### K3s

For the Kubernetes distribution we use K3s. K3s is based on Rancher, which is a robust enterprise-grade Kubernetes distribution by SUSE. K3s is a more lightweight version of Rancher, originally designed for edge computing. However it also works very well in large server environments and is production ready and robust.

As K3s delivers everything we need for a fully robust scalable cluster that can run LLMs and the Unique platform, we can reduce the complexity of the infrastructure and the operational overhead, alongside the resources needed to run Kubernetes.

### Cilium CNI

We use Cilium as our CNI (Container Network Interface). Cilium is a modern, fast, and secure L3/L4 network and security solution. A key advantage of Cilium is its all-in-one approach, eliminating the need for additional tools like MetalLB, an Istio service mesh or similar.

### Certmanager

All public-facing services are secured using Let's Encrypt certificates, managed via CertManager, which also handles certificate renewal.

CertManager can also integrate with other certificate authorities, such as internal CAs in an enterprise environment.

### ArgoCD

We use ArgoCD for Kubernetes resource management. ArgoCD is a declarative, GitOps-based continuous delivery tool.

With ArgoCD, we deploy most components automatically, managing configuration via Git. It also provides a quick status check on cluster health and updates.

## Ansible

Our example ansible setup consists of a single playbook with multiple tags and roles.

### How to run the Ansible playbook

1. Create a `ansible/values.yaml` file with the appropriate values for your cluster. This is the central configuration file.

2. Create a `ansible/inventories/on_premise/hosts.yaml` file. Define all the hosts with their respective IP address and assign them to the control plane or worker group.

3. Adjust the `ansible/run.sh` script to use your 1Password account. If you don't use 1Password to provision your secrets, you can also run Ansible directly from the command line without the script.

4. Run the playbook from the `ansible` directory with:

```bash
./run.sh
```

You can provide a tag to the `run.sh` script to run a specific set of tasks:

Example:

```bash
./run.sh charts
```

This will only run the tasks tagged with `charts`.

Our example playbook has the following tags:

- `cluster`: Upgrade the apt-packages, setup and harden SSH, provision management user.
- `kubernetes`: Provision the Kubernetes cluster with k3s.
- `network`: Provision the network resources like Cilium and *optionally* Tailscale VPN.
- `charts`: Provision the Helm charts. You can define a list of roles to install in the `values.yaml` file.
- `apps`: Setup ArgoCD applications so they can be deployed on the cluster via ArgoCD.


## Secrets Provisioning

Often you have to deliver existing secrets to a cluster. Examples are API keys of third party services or oAuth credentials for SSO providers.

### 1Password (used at Unique)

At Unique, we store cluster-related secrets in a 1Password vault. During deployment, the 1Password CLI fetches secrets, sets them as environment variables, and Ansible provisions them.

There is also the option to deploy a [1Password Operator](https://developer.1password.com/docs/k8s/k8s-integrations) in the cluster and pull secrets at runtime from the 1Password vault.

Ref: [External Secrets](software-infrastructure.md#external-secrets)

### Other Solutions

- HashiCorp Vault (e.g. managed by https://github.com/bank-vaults/bank-vaults)
- Sealed Secrets

## Components provisioned with ArgoCD

After the initial cluster bootstrap with Ansible, we use ArgoCD to deploy the other components.

### Kong Ingress Controller

We use Kong as our Ingress Controller. Kong is a powerful, scalable, and flexible API gateway and ingress controller. We use it to terminate TLS connections, CORS requests and use it as an authentication layer to validate JWT tokens and API keys in front of our backend services.

Using the Ingress Controller allows us to deploy Kong without a database. It is configured purely
via custom resources in Kubernetes.

### Prometheus, Grafana, Alertmanager

We use the widely known Prometheus, Grafana and Alertmanager stack as our monitoring and alerting solution. All components in this setup are configured with metrics export to Prometheus, sensible production-ready alerts and a dashboard for visualizing the metrics.

### Loki

Loki is a logging solution that is tightly integrated with Grafana. We use it in a simple SingleBinary deployment mode for our cluster. Typically this is enough for most use cases and scales on bare-metal servers well until a certain log volume is exceeded.

We recommend keeping Loki as simple as possible until it becomes too slow to handle the log volume. After this point, it supports more modularized deployment modes that can scale very efficiently to any kind of log volume.

### Reloader

Reloader watches ConfigMaps and Secrets and triggers a restart of the affected PODs automatically. This is useful to ensure all components are running the latest version of their configuration.

### External Secrets

With External Secrets we prepare secrets and environment variables for our application layer. For example, we can use it to create a `DATABASE_URL` needed in our backend services to connect to our PostgreSQL database and pull the necessary credentials from the secret our Postgres operator created.

External Secrets can also be used to load secrets from HashiCorp Vault or a cluster external secret store.

### Rook Ceph

Ceph is a robust, well-known storage solution for Kubernetes. We selected it over more simple storage solutions like Longhorn, because it combines support for Block, File and Object storage (AWS S3 compatible) in a single solution.

Also in cases of hardware failures, Ceph is easy to administer to restore a healthy storage cluster.

The Rook Ceph operator is a userfriendly way to deploy Ceph in a Kubernetes cluster.

### Stackgres

Stackgres is a PostgreSQL operator for Kubernetes. We use it instead of a simple PostgreSQL chart, to benefit from automatic backups, monitoring and log handling. It also supports scripts to automatically boostrap databases for our backend services as code.

At the infrastructure layer, we only install the operator. Database instances are deployed at the application layer.

### Redis

Redis is deployed as a simple Helm chart for our solution. At the infrastructure layer, we only prepare Grafana dashboards and alerts for Redis, but don't deploy an actual Redis instance.

### RabbitMQ

The official RabbitMQ operator helps with managing and scaling RabbitMQ clusters. At the infrastructure layer, we only install the operator and prepare Grafana dashboards and alerts for it. The RabbitMQ instances are deployed at the application layer.

### Harbor

Harbor is a OCI conform container registry. We use it to sync all Unique container images for local installation. With each new Unique release, we ship a set of new container images that you can sync to your local registry.

Harbor can also be used as a mirror for public container images and helm charts, allowing a fully air-gapped deployment of the cluster.

### Nvidia GPU & NIM Operators

- Nvidia GPU Operator: Manages CUDA driver installation and GPU resource sharing (MIG, time-slicing).
- Nvidia NIM Operator: Caches and runs Nvidia NIMs, optimized containers for running LLMs like Llama and Qwen.

**Note: Running NIMs requires an Nvidia Enterprise AI license.**

### LiteLLM

LiteLLM is a LLM proxy that supports all kind of LLM providers and hosting options and offers a unified openAI compatible API to our backend services.