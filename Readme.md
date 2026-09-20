I incorporated the correction we discovered for Traefik Helm chart 41.6.0: the Service type must be configured under service.spec.type, not service.type.

I also use the current Argo CD Traefik pattern—TLS termination at Traefik, Argo CD running with server.insecure: "true", and h2c for the gRPC route. AArgo CD+1 Kind's extraPortMappings are used to get traffic from the remote Azure VM host into the Kind node. KKind+1 cert-manager's HTTP-01 solver uses an Ingress and ingressClassName to route ACME validation traffic through the selected ingress controller. Ccert-manager
Azure Ubuntu + Kind + Traefik + Argo CD + Let's Encrypt

This guide configures an externally accessible Argo CD installation on:

Azure Ubuntu VM
    |
    +-- Docker
    |
    +-- Kind Kubernetes Cluster
          |
          +-- 1 Control Plane
          +-- 2 Worker Nodes
          |
          +-- Traefik Ingress Controller
          |
          +-- Argo CD

The final result is:

https://argocd.yourdomain.com

The same architecture can later be used for application URLs such as:

https://app.yourdomain.com
https://api.yourdomain.com
https://grafana.yourdomain.com

1. Target Architecture

                              Internet
                                  |
                                  |
                         argocd.yourdomain.com
                                  |
                                  | DNS A Record
                                  v
                         +------------------+
                         | Azure Public IP  |
                         +--------+---------+
                                  |
                             TCP 80 / 443
                                  |
                         Azure Network NSG
                                  |
                                  v
                     +------------------------+
                     |    Ubuntu Azure VM     |
                     |                        |
                     |       Docker           |
                     |                        |
                     |   +----------------+   |
                     |   | Kind Cluster   |   |
                     |   |                |   |
                     |   | Control Plane  |   |
                     |   | Worker 1       |   |
                     |   | Worker 2       |   |
                     |   |                |   |
                     |   |   Traefik      |   |
                     |   |      |         |   |
                     |   |      v         |   |
                     |   | Argo CD        |   |
                     |   +----------------+   |
                     +------------------------+

Traffic flow:

Internet
   |
   | HTTPS :443
   v
Azure Public IP
   |
   v
Azure NSG
   |
   v
Ubuntu VM :443
   |
   v
Docker host port :443
   |
   v
Kind control-plane :30443
   |
   v
Kubernetes NodePort :30443
   |
   v
Traefik :443
   |
   v
argocd-server :80

Kind supports extraPortMappings specifically for forwarding ports from the host into Kind nodes, which is useful for NodePort-based ingress setups. KKind+1
2. Prerequisites

The Azure Ubuntu VM should have:

    Docker

    kubectl

    Kind

    Helm

    Azure CLI

    An existing Kind cluster

    Argo CD installed in namespace argocd

    A DNS domain that you control

    An Azure Public IP attached to the VM

Verify:

docker --version
kubectl version --client
kind version
helm version
az version

Verify Kind:

kind get clusters

Verify Kubernetes:

kubectl get nodes

Expected:

NAME                  STATUS   ROLES
kind-control-plane    Ready    control-plane
kind-worker           Ready    <none>
kind-worker2          Ready    <none>

Verify Argo CD:

kubectl get pods -n argocd

3. Configure Environment Variables

Replace yourdomain.com with your real domain.

export DOMAIN="yourdomain.com"
export ARGOCD_HOST="argocd.${DOMAIN}"

Verify:

echo "$DOMAIN"
echo "$ARGOCD_HOST"

Expected:

yourdomain.com
argocd.yourdomain.com

Also set your Let's Encrypt email:

export LE_EMAIL="your-email@example.com"

4. Important: Kind Port Mapping

The Kind cluster must expose:

Host :80  -> Kind Node :30080
Host :443 -> Kind Node :30443

The Kind configuration is:

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 80
        protocol: TCP
        listenAddress: "0.0.0.0"

      - containerPort: 30443
        hostPort: 443
        protocol: TCP
        listenAddress: "0.0.0.0"

  - role: worker

  - role: worker

Important

extraPortMappings are created when the Kind node containers are created.

If the existing Kind cluster was created without these mappings, the cleanest approach for a disposable development/lab cluster is to recreate it.

If the existing cluster contains important workloads, do not delete it. Use the alternative host-proxy approach described later in this README.
5. Back Up Existing Argo CD Configuration

Before recreating Kind:

mkdir -p ~/kind-backup

Save Argo CD resources:

kubectl get applications.argoproj.io -A -o yaml \
  > ~/kind-backup/argocd-applications.yaml

Save Argo CD resources:

kubectl get all -n argocd -o yaml \
  > ~/kind-backup/argocd-resources.yaml

Save all cluster resources if required:

kubectl get all -A -o yaml \
  > ~/kind-backup/cluster-resources.yaml

If your applications are already managed from Git through Argo CD, rebuilding the Kind cluster is much easier because Argo CD can recreate the workloads from Git.
6. Create Kind Configuration

Create a working directory:

mkdir -p ~/kind
cd ~/kind

Create:

nano kind-config.yaml

Use:

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 80
        protocol: TCP
        listenAddress: "0.0.0.0"

      - containerPort: 30443
        hostPort: 443
        protocol: TCP
        listenAddress: "0.0.0.0"

  - role: worker

  - role: worker

7. Recreate Kind

Only perform this step if the current cluster is disposable.

kind delete cluster

Create the cluster:

kind create cluster --config ~/kind/kind-config.yaml

Verify:

kubectl get nodes

Expected:

NAME                  STATUS   ROLES
kind-control-plane    Ready    control-plane
kind-worker           Ready    <none>
kind-worker2          Ready    <none>

Verify Docker port mapping:

docker port kind-control-plane

Expected to contain:

30080/tcp -> 0.0.0.0:80
30443/tcp -> 0.0.0.0:443

8. Install Argo CD

If Argo CD has not yet been installed:

kubectl create namespace argocd

Install using Helm according to your chosen Argo CD chart/version.

Verify:

kubectl get pods -n argocd

Verify Service:

kubectl get svc -n argocd

Expected:

argocd-server   ClusterIP

The Argo CD Service can remain ClusterIP. It does not need to be exposed directly because Traefik will provide the external entry point.
9. Install Traefik

Add the Helm repository:

helm repo add traefik https://traefik.github.io/charts

Update:

helm repo update

Create namespace:

kubectl create namespace traefik

10. Traefik Configuration

This guide uses Traefik Helm chart:

41.6.0

Create:

nano ~/kind/traefik-values.yaml

Use:

deployment:
  kind: Deployment

service:
  enabled: true

  spec:
    type: NodePort

ports:
  web:
    port: 8000
    exposedPort: 80
    nodePort: 30080
    protocol: TCP

    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    port: 8443
    exposedPort: 443
    nodePort: 30443
    protocol: TCP

    http:
      tls:
        enabled: true

Important Traefik chart detail

For Traefik Helm chart 41.6.0, the Service type is configured under:

service:
  spec:
    type: NodePort

Not:

service:
  type: NodePort

The latter will not override the chart's default Service type.
11. Install Traefik

helm upgrade --install traefik traefik/traefik \
  --namespace traefik \
  --version 41.6.0 \
  --values ~/kind/traefik-values.yaml

Wait for deployment:

kubectl rollout status deployment/traefik \
  -n traefik

Check:

kubectl get pods -n traefik

Expected:

NAME                       READY   STATUS
traefik-xxxxxxxxxx-xxxxx   1/1     Running

12. Verify Traefik Service

Run:

kubectl get svc traefik -n traefik

Expected:

NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
traefik   NodePort   10.x.x.x       <none>        80:30080/TCP,443:30443/TCP

Verify directly:

kubectl get svc traefik \
  -n traefik \
  -o jsonpath='{.spec.type}{"\n"}'

Expected:

NodePort

Verify ports:

kubectl get svc traefik \
  -n traefik \
  -o jsonpath='{range .spec.ports[*]}{.name}{" => "}{.port}{":"}{.nodePort}{"\n"}{end}'

Expected:

web => 80:30080
websecure => 443:30443

13. Verify Traefik Endpoints

kubectl get endpoints traefik -n traefik

Verify pods:

kubectl get pods -n traefik -o wide

The Service should have an endpoint pointing to the Traefik pod.
14. Test Traefik NodePort

From the Azure VM:

curl -v http://127.0.0.1:30080

A Traefik response such as:

404 Not Found

is acceptable.

At this stage there is no route configured yet.

Test HTTPS:

curl -vk https://127.0.0.1:30443

Again, a Traefik response is enough.
15. Test Kind Host Port Mapping

Test HTTP:

curl -v http://127.0.0.1

Test HTTPS:

curl -vk https://127.0.0.1

The expected flow is:

Ubuntu :80
    |
    v
Docker
    |
    v
Kind :30080
    |
    v
Kubernetes NodePort
    |
    v
Traefik

For HTTPS:

Ubuntu :443
    |
    v
Docker
    |
    v
Kind :30443
    |
    v
Kubernetes NodePort
    |
    v
Traefik

16. Azure Public IP

Get the VM public IP:

az vm list \
  --show-details \
  --query "[].{VM:name,RG:resourceGroup,PublicIP:publicIps}" \
  -o table

Example:

VM              RG              PublicIP
--------------- --------------- -------------
docker-images    my-resource    20.123.45.67

Set it:

export PUBLIC_IP="20.123.45.67"

Verify:

echo "$PUBLIC_IP"

17. Azure Network Security Group

Only ports 80 and 443 need to be exposed publicly for this architecture.

Do NOT expose:

30080
30443

to the Internet.

Those ports are internal to the Azure VM / Kind networking path.
18. Find the VM Network Interface

export RESOURCE_GROUP="<RESOURCE_GROUP>"
export VM_NAME="<VM_NAME>"

Find NIC:

export NIC_ID=$(az vm show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$VM_NAME" \
  --query "networkProfile.networkInterfaces[0].id" \
  -o tsv)

Verify:

echo "$NIC_ID"

Find NSG:

export NSG_ID=$(az network nic show \
  --ids "$NIC_ID" \
  --query "networkSecurityGroup.id" \
  -o tsv)

Verify:

echo "$NSG_ID"

Extract NSG name:

export NSG_NAME=$(basename "$NSG_ID")

19. Allow HTTP 80

For initial setup and Let's Encrypt HTTP-01 validation:

az network nsg rule create \
  --resource-group "$RESOURCE_GROUP" \
  --nsg-name "$NSG_NAME" \
  --name Allow-HTTP \
  --priority 300 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --source-port-ranges "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges 80

20. Allow HTTPS 443

az network nsg rule create \
  --resource-group "$RESOURCE_GROUP" \
  --nsg-name "$NSG_NAME" \
  --name Allow-HTTPS \
  --priority 310 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --source-port-ranges "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges 443

Verify:

az network nsg rule list \
  --resource-group "$RESOURCE_GROUP" \
  --nsg-name "$NSG_NAME" \
  -o table

21. Test Azure Public IP

HTTP:

curl -v http://$PUBLIC_IP

HTTPS:

curl -vk https://$PUBLIC_IP

At this point a Traefik 404 is acceptable.

The important thing is that the request reaches Traefik.
22. Configure DNS

Create an A record with your DNS provider:

Type:  A
Name:  argocd
Value: <AZURE_PUBLIC_IP>
TTL:   300

For example:

argocd.yourdomain.com -> 20.123.45.67

Verify from the Azure VM:

dig +short argocd.yourdomain.com

Expected:

20.123.45.67

You can also use:

nslookup argocd.yourdomain.com

Do not continue to Let's Encrypt until DNS resolves correctly.
23. Test DNS Through Traefik

Run:

curl -v http://argocd.yourdomain.com

At this point you should reach Traefik.

You may receive:

404 page not found

This is expected because the Argo CD route hasn't been created yet.
24. Configure Argo CD for Traefik TLS Termination

Argo CD normally serves HTTPS/gRPC itself.

For this architecture:

Internet
    |
 HTTPS
    |
 Traefik
    |
 HTTP/H2C
    |
 Argo CD

Argo CD TLS is therefore disabled internally.

Argo CD's documentation specifically documents this pattern for Traefik: set server.insecure: "true" and let Traefik terminate TLS. AArgo CD+1

Run:

kubectl -n argocd patch configmap argocd-cmd-params-cm \
  --type merge \
  -p '{"data":{"server.insecure":"true"}}'

Restart:

kubectl -n argocd rollout restart deployment argocd-server

Wait:

kubectl -n argocd rollout status deployment argocd-server

Verify:

kubectl get pods -n argocd

25. Install cert-manager

Install cert-manager:

kubectl apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.21.2/cert-manager.yaml

Verify:

kubectl get pods -n cert-manager

Expected components:

cert-manager
cert-manager-cainjector
cert-manager-webhook

Wait:

kubectl wait \
  --for=condition=Available \
  deployment/cert-manager \
  -n cert-manager \
  --timeout=180s

26. Create Let's Encrypt Staging Issuer

Use staging first to avoid unnecessary production ACME requests while troubleshooting.

Create:

nano ~/kind/letsencrypt-staging.yaml

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: YOUR_EMAIL@example.com

    server: https://acme-staging-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-staging-account-key

    solvers:
      - http01:
          ingress:
            ingressClassName: traefik

Replace:

YOUR_EMAIL@example.com

with your email.

Apply:

kubectl apply -f ~/kind/letsencrypt-staging.yaml

Check:

kubectl get clusterissuer

Expected:

NAME                  READY
letsencrypt-staging   True

cert-manager recommends using ingressClassName to select the Ingress controller used for HTTP-01 challenges. Ccert-manager
27. Create Argo CD TLS Certificate

Create:

nano ~/kind/argocd-certificate.yaml

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-tls
  namespace: argocd
spec:
  secretName: argocd-tls

  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer

  dnsNames:
    - argocd.yourdomain.com

Replace the hostname if required.

Apply:

kubectl apply -f ~/kind/argocd-certificate.yaml

Check:

kubectl get certificate -n argocd

28. Create Argo CD Traefik IngressRoute

Create:

nano ~/kind/argocd-ingressroute.yaml

Use:

apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure

  routes:
    - kind: Rule
      match: Host(`argocd.yourdomain.com`)
      priority: 10
      services:
        - name: argocd-server
          port: 80

    - kind: Rule
      match: Host(`argocd.yourdomain.com`) && Header(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argocd-server
          port: 80
          scheme: h2c

  tls:
    secretName: argocd-tls

Apply:

kubectl apply -f ~/kind/argocd-ingressroute.yaml

Verify:

kubectl get ingressroute -n argocd

The current Argo CD documentation shows the Traefik pattern with an HTTP route plus an h2c gRPC route and TLS termination at Traefik. AArgo CD+1
29. Monitor the Certificate

Run:

kubectl get certificate \
  -n argocd \
  -w

Eventually:

NAME          READY
argocd-tls    True

Check all ACME resources:

kubectl get certificate,certificaterequest,order,challenge \
  -n argocd

If the certificate fails:

kubectl describe certificate argocd-tls \
  -n argocd

Check challenges:

kubectl get challenge -n argocd

Then:

kubectl describe challenge -n argocd

30. Test HTTPS Using Staging

Once the certificate is ready:

curl -vk https://argocd.yourdomain.com

A browser will likely show a certificate warning because the Let's Encrypt staging CA is not trusted by normal browsers.

That is expected.

The important result is that:

DNS
 |
 v
Azure
 |
 v
Kind
 |
 v
Traefik
 |
 v
cert-manager
 |
 v
Let's Encrypt
 |
 v
Argo CD

works end-to-end.
31. Switch to Let's Encrypt Production

Once staging works, create the production ClusterIssuer.

nano ~/kind/letsencrypt-production.yaml

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: YOUR_EMAIL@example.com

    server: https://acme-v02.api.letsencrypt.org/directory

    privateKeySecretRef:
      name: letsencrypt-prod-account-key

    solvers:
      - http01:
          ingress:
            ingressClassName: traefik

Apply:

kubectl apply -f ~/kind/letsencrypt-production.yaml

Verify:

kubectl get clusterissuer

Expected:

NAME                  READY
letsencrypt-prod      True

32. Change Argo CD Certificate to Production

Edit:

nano ~/kind/argocd-certificate.yaml

Change:

issuerRef:
  name: letsencrypt-staging

to:

issuerRef:
  name: letsencrypt-prod

Apply:

kubectl apply -f ~/kind/argocd-certificate.yaml

Monitor:

kubectl get certificate -n argocd -w

Expected:

NAME          READY
argocd-tls    True

33. Verify Production Certificate

Run:

openssl s_client \
  -connect argocd.yourdomain.com:443 \
  -servername argocd.yourdomain.com \
  </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

You should see a Let's Encrypt issuer.

Then:

curl -I https://argocd.yourdomain.com

34. Open Argo CD

Open:

https://argocd.yourdomain.com

Get the initial admin password:

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

echo

Login:

Username: admin
Password: <password from above>

35. Test Argo CD CLI

Login:

argocd login argocd.yourdomain.com

If gRPC has an issue:

argocd login argocd.yourdomain.com --grpc-web

The reason the gRPC route is explicitly configured is that Argo CD's API server serves both the browser's HTTP/HTTPS traffic and gRPC traffic used by the CLI. GGitHub
36. Final Architecture

The completed system looks like:

                           Internet
                               |
                               |
                    argocd.yourdomain.com
                               |
                               v
                       DNS A Record
                               |
                               v
                       Azure Public IP
                               |
                         TCP 80 / 443
                               |
                               v
                       Azure NSG
                               |
                               v
                     +----------------+
                     |  Ubuntu VM     |
                     |                |
                     | Docker / Kind  |
                     +-------+--------+
                             |
                             |
                      Host :80/:443
                             |
                             v
                    Kind Port Mapping
                             |
                    +--------+--------+
                    |                 |
                  :30080           :30443
                    |                 |
                    +--------+--------+
                             |
                             v
                      Traefik NodePort
                             |
                             v
                         Traefik
                             |
                +------------+------------+
                |                         |
                v                         v
          HTTP/UI Route              gRPC Route
                |                         |
                +------------+------------+
                             |
                             v
                       Argo CD Server
                             |
                             v
                       Kubernetes

37. Application Architecture

Once Argo CD is accessible, applications should normally remain internal:

Internet
    |
    v
Traefik
    |
    +---- argocd.yourdomain.com
    |          |
    |          v
    |      Argo CD
    |
    +---- app.yourdomain.com
    |          |
    |          v
    |      frontend ClusterIP
    |
    +---- api.yourdomain.com
               |
               v
           backend ClusterIP

Applications should generally use:

spec:
  type: ClusterIP

rather than creating a public LoadBalancer for every application.
38. Example Application Service

apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-app
spec:
  type: ClusterIP

  selector:
    app: my-app

  ports:
    - port: 80
      targetPort: 8080

The application does not need a public IP.
39. Example Application IngressRoute

apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: my-app
  namespace: my-app
spec:
  entryPoints:
    - websecure

  routes:
    - match: Host(`app.yourdomain.com`)
      kind: Rule
      services:
        - name: my-app
          port: 80

  tls:
    secretName: my-app-tls

40. Example Application Certificate

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-app-tls
  namespace: my-app
spec:
  secretName: my-app-tls

  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

  dnsNames:
    - app.yourdomain.com

Then:

https://app.yourdomain.com

routes through the same Traefik instance.
41. Recommended DNS Strategy

For a small number of applications:

argocd.yourdomain.com  -> Azure Public IP
app.yourdomain.com     -> Azure Public IP
api.yourdomain.com     -> Azure Public IP

For a larger environment, consider:

*.yourdomain.com -> Azure Public IP

Then:

argocd.yourdomain.com
app.yourdomain.com
api.yourdomain.com
grafana.yourdomain.com

all resolve to the same Azure Public IP.

For wildcard TLS certificates, use cert-manager's DNS-01 challenge rather than HTTP-01.
42. Security Recommendations

Do not expose:

30080
30443

through the Azure NSG.

Only expose:

80
443

Publicly.

For a personal development environment, consider restricting SSH:

TCP 22
Source: YOUR_PUBLIC_IP/32

rather than allowing SSH from the Internet.

For Argo CD, consider restricting access further using:

    Azure NSG source restrictions

    VPN

    Azure Bastion/private networking

    SSO

    Argo CD RBAC

    additional authentication controls

Do not use the initial admin password as a permanent credential.
43. Troubleshooting
Traefik Service is LoadBalancer

Check:

helm get values traefik -n traefik

For chart 41.6.0, the configuration must contain:

service:
  spec:
    type: NodePort

Check rendered manifest:

helm get manifest traefik -n traefik \
  | grep -A20 "kind: Service"

It must contain:

spec:
  type: NodePort

Traefik Pod is not running

kubectl get pods -n traefik

Then:

kubectl describe pod \
  -n traefik \
  <POD_NAME>

Check logs:

kubectl logs \
  -n traefik \
  deployment/traefik

NodePort doesn't respond

Check:

kubectl get svc traefik -n traefik

Then:

kubectl get endpoints traefik -n traefik

Test:

curl -v http://127.0.0.1:30080

Kind host port doesn't respond

Check:

docker port kind-control-plane

Expected:

30080/tcp -> 0.0.0.0:80
30443/tcp -> 0.0.0.0:443

Azure public IP doesn't respond

Check NSG:

az network nsg rule list \
  --resource-group "$RESOURCE_GROUP" \
  --nsg-name "$NSG_NAME" \
  -o table

Check Linux:

sudo ss -lntp

Check Docker:

docker ps

DNS doesn't resolve

dig +short argocd.yourdomain.com

It must return the Azure public IP.
Certificate is stuck

kubectl get certificate -n argocd

kubectl get certificaterequest -n argocd

kubectl get order -n argocd

kubectl get challenge -n argocd

Then:

kubectl describe challenge -n argocd

For HTTP-01, the ACME validation request must be able to reach the HTTP endpoint through the ingress controller. Ccert-manager
Argo CD UI works but CLI doesn't

Check the gRPC route:

- kind: Rule
  match: Host(`argocd.yourdomain.com`) && Header(`Content-Type`, `application/grpc`)
  priority: 11
  services:
    - name: argocd-server
      port: 80
      scheme: h2c

Try:

argocd login argocd.yourdomain.com --grpc-web

44. Useful Verification Commands
Kubernetes

kubectl get nodes
kubectl get pods -A
kubectl get svc -A

Traefik

kubectl get pods -n traefik
kubectl get svc -n traefik
kubectl get ingressroute -A
kubectl logs -n traefik deployment/traefik

Argo CD

kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl get ingressroute -n argocd

Certificates

kubectl get certificate -A
kubectl get clusterissuer
kubectl get certificaterequest -A
kubectl get order -A
kubectl get challenge -A

Kind

kind get clusters
docker ps
docker port kind-control-plane

DNS

dig +short argocd.yourdomain.com

Network

curl -v http://127.0.0.1:30080
curl -vk https://127.0.0.1:30443
curl -v http://127.0.0.1
curl -vk https://127.0.0.1
curl -v http://argocd.yourdomain.com
curl -vk https://argocd.yourdomain.com

45. Final Checklist

Before considering the setup complete:

[ ] Azure VM has Public IP
[ ] Azure NSG allows TCP 80
[ ] Azure NSG allows TCP 443
[ ] Kind control plane maps host 80 -> 30080
[ ] Kind control plane maps host 443 -> 30443
[ ] Traefik is installed
[ ] Traefik Service is NodePort
[ ] Traefik HTTP NodePort is 30080
[ ] Traefik HTTPS NodePort is 30443
[ ] Traefik Pod is Running
[ ] DNS points to Azure Public IP
[ ] argocd-server exists
[ ] Argo CD server.insecure=true
[ ] cert-manager is Running
[ ] Let's Encrypt staging issuer is Ready
[ ] Argo CD staging certificate is Ready
[ ] Argo CD Traefik IngressRoute exists
[ ] HTTPS works with staging certificate
[ ] Let's Encrypt production issuer is Ready
[ ] Production certificate is Ready
[ ] https://argocd.yourdomain.com works
[ ] Argo CD CLI login works

46. Recommended Next Architecture

Once this is working, use GitOps to manage the rest of the platform:

                         Git Repository
                               |
                               v
                            Argo CD
                               |
              +----------------+----------------+
              |                |                |
              v                v                v
          Frontend          Backend            API
              |                |                |
              +----------------+----------------+
                               |
                               v
                            Traefik
                               |
              +----------------+----------------+
              |                |                |
              v                v                v
       app.yourdomain.com api.yourdomain.com other...

Keep the application Services as:

ClusterIP

and expose applications through Traefik.

This gives you one public entry point:

Azure Public IP
       |
      80/443
       |
    Traefik
       |
       +-- Argo CD
       +-- Frontend
       +-- Backend
       +-- API
       +-- Grafana
       +-- Other applications

This is the recommended pattern for the current Kind-based lab architecture.

One correction I'd make compared with the earlier walkthrough: don't use service.type: NodePort with Traefik chart 41.6.0. The README above uses the verified service.spec.type: NodePort structure from the chart you actually have installed.
