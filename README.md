# Prerequisites and Environment Setup

This repository contains two Kubernetes lab walkthroughs:

- [Deploy Argo CD with a certificate](Deploy_argocd_with_certificate/readme.md)
- [Deploy the monitoring stack with Traefik, cert-manager, Prometheus, and Grafana](K8S_Monitoring_Stack_with_Traefik_cert-manager,_Prometheus_&_Grafana/readme.md)

Use this checklist before following either walkthrough. The examples target a learning/lab environment: an Ubuntu Azure VM running Docker and a multi-node Kind cluster. A single VM and Kind cluster are not a highly available production Kubernetes platform.

## 1. Access and Accounts

Prepare the following before starting:

- An Azure subscription and permission to create or use a VM, public IP, network interface, and Network Security Group (NSG).
- SSH access and `sudo` privileges on an Ubuntu VM. Ubuntu 22.04 LTS or 24.04 LTS is recommended.
- A DNS domain you control, and access to edit its DNS records.
- A working email address for Let's Encrypt notifications.
- A Git client and access to this repository on the VM (or a way to transfer the files).

Recommended VM capacity for the full repository, including Prometheus and Grafana, is at least 4 vCPUs, 8 GiB RAM, and 40 GiB of free disk. Smaller environments may run out of memory or storage while installing charts or retaining metrics. Confirm Azure quota and VM availability in the target region.

## 2. Network and DNS

Use one Azure public IP for the VM and point the required DNS hostnames to it. Reserve a static public IP where possible so that DNS does not break after a VM lifecycle change.

Allow inbound **TCP 80 and 443** to the VM in the NSG and, if enabled, the Ubuntu firewall. The VM must also be able to make outbound DNS and HTTPS connections to download images/charts and reach Let's Encrypt. Do not expose Kubernetes API, Prometheus, Grafana, or Argo CD NodePorts directly to the internet; route public web traffic through Traefik.

Create DNS A records for the hostnames you intend to use, for example:

| Purpose | Example hostname | DNS target |
| --- | --- | --- |
| Argo CD | `argocd.example.com` | Azure VM public IP |
| Grafana | `grafana.example.com` | Azure VM public IP |

Add any application hostname required by your own deployment. DNS must resolve publicly to the VM before an HTTP-01 certificate can be issued. Verify from the VM or another machine:

```bash
dig +short argocd.example.com
dig +short grafana.example.com
```

Both commands should return the VM's public IP. If `dig` is not installed, use `getent hosts <hostname>`.

## 3. Install the Base Tools

Install supported/current versions of these tools on the Ubuntu VM:

- Docker Engine, with permission for your login user to run Docker commands.
- `kubectl` compatible with the Kubernetes version used by Kind.
- Kind.
- Helm 3.
- Azure CLI only if you will use the Azure inspection/configuration commands in the walkthroughs; it is not needed by Kubernetes itself.
- `curl`, `git`, and DNS utilities such as `dnsutils` (`dig`).

The installation method and versions may depend on your organization. After installation, check that each command is available:

```bash
docker --version
kubectl version --client
kind version
helm version
az version       # optional, only when using Azure CLI
git --version
curl --version
```

If Docker requires `sudo`, either use `sudo` consistently or configure Docker group access and reconnect to the VM. Do not continue until `docker ps` works for the account that will create the cluster.

## 4. Kind Cluster Requirements

The shared configuration expects a Kind cluster with one control-plane node and two workers. The control-plane container must map host ports to the Traefik NodePorts:

| VM host port | Kind node container port | Traefik NodePort |
| --- | --- | --- |
| TCP 80 | 30080 | 30080 |
| TCP 443 | 30443 | 30443 |

The repository's [Kind configuration](Deploy_argocd_with_certificate/kind-config.yaml) defines these mappings. They are set when Kind creates the node containers; they cannot be added to an existing cluster in place. If recreating a cluster, first confirm it contains no data or workloads you need. Back up important Kubernetes resources and persistent data before any rebuild.

Check for an existing cluster and local port conflicts:

```bash
kind get clusters
sudo ss -lntp | grep -E ':(80|443)\b' || true
```

After creating/selecting the cluster, verify it is healthy:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get storageclass
```

All three nodes should be `Ready`. The monitoring values request persistent volumes using the `standard` StorageClass. Confirm that a suitable StorageClass exists and that its provisioner works in your Kind setup before installing the monitoring chart. Do not assume an Azure managed-disk StorageClass is available inside Kind.

## 5. Shared Kubernetes Components

Both walkthroughs expect Traefik to be the public ingress controller, using NodePorts `30080` (HTTP) and `30443` (HTTPS), with the Kind host mappings above. The certificate workflow also depends on cert-manager and an HTTP-01 `ClusterIssuer` that selects Traefik. Install components using the versions and steps documented in the folder guides, and verify their readiness before proceeding:

```bash
kubectl get pods -A
kubectl get ingressclass
kubectl get clusterissuer
```

For the production issuer, verify the issuer is `Ready` before creating application certificates. Use the Let's Encrypt staging issuer while testing; repeated failed production requests can hit rate limits.

## 6. Workflow-Specific Requirements

**Argo CD:** The Argo CD walkthrough expects Argo CD to be installed in the `argocd` namespace before applying its ingress and certificate resources. Confirm the server is healthy:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

**Monitoring:** The monitoring walkthrough deploys `kube-prometheus-stack` in the `monitoring` namespace. Confirm sufficient free disk and working persistent-volume provisioning. Create a strong Grafana admin password and store it in a Kubernetes Secret as shown in that guide; do not commit credentials to Git.

## 7. Customize Repository Examples

Before applying manifests or requesting certificates, replace all sample values with your own:

- DNS hostnames in the certificate, ingress, and Grafana values files.
- The Let's Encrypt contact email.
- Any Azure resource group, VM, region, public IP, or subscription values used in commands.
- Secret values such as the Grafana administrator password.

Keep a hostname identical across DNS, the certificate `dnsNames`, the ingress `host`/TLS host, and (for Grafana) the Grafana `root_url` and `domain`. The current sample manifests contain repository-specific hostnames and email values; editing only the DNS record is not enough. Never commit real passwords, private keys, or cloud credentials.

## 8. Ready-to-Start Checklist

- [ ] Ubuntu Azure VM is reachable and has the required capacity and free disk.
- [ ] Docker, `kubectl`, Kind, and Helm run successfully for the deployment account.
- [ ] Azure NSG/firewall permits inbound TCP 80 and 443; outbound DNS/HTTPS works.
- [ ] DNS hostnames resolve to the VM's public IP.
- [ ] Kind has three `Ready` nodes and the required 80/443 port mappings.
- [ ] A usable `standard` StorageClass is available before deploying monitoring.
- [ ] Traefik is ready; cert-manager and the intended `ClusterIssuer` are ready for certificate workflows.
- [ ] Argo CD is installed in `argocd` before following its ingress/certificate steps.
- [ ] Sample domains, email addresses, cloud values, and credentials have been replaced.

When these checks pass, follow the relevant detailed guide linked at the top of this README.
