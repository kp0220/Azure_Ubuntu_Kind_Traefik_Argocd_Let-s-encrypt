    This repository documents the monitoring architecture for a Kind Kubernetes cluster running on an Azure VM.

    The design keeps Traefik as the single public ingress point, uses cert-manager + Let's Encrypt for Grafana TLS, and deploys kube-prometheus-stack for Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter.
    Target Architecture

                             Internet
                                |
                                | 443 / HTTPS
                                v
                        Azure VM Public IP
                                |
                         TCP 80 / 443
                                |
                                v
                        Kind NodePort
                       30080 / 30443
                                |
                                v
                         +-------------+
                         |   Traefik   |
                         |   80 / 443  |
                         +------+------+
                                |
                 +--------------+--------------+
                 |              |              |
                 v              v              v
          argocd.yog...   app.yog...   grafana.yog...
                 |              |              |
              ArgoCD        Online-Shop       Grafana
                                             |
                                             v
                                        Prometheus
                                             |
                             +---------------+---------------+
                             |               |               |
                             v               v               v
                        kube-state-     node-exporter     kubelet
                         metrics

    Public endpoints
    Hostname	Component	Exposure
    argocd.yogifods.store	ArgoCD	Traefik
    app.yogifods.store	Online Shop	Traefik
    grafana.yogifods.store	Grafana	Traefik
    Prometheus	Prometheus UI	Internal only

    The important design principle is:

        Traefik is the only application-facing public ingress.

    Grafana and Prometheus should not be exposed using additional NodePorts or Azure public endpoints.
    1. Current Environment

    The current lab environment consists of:

    Azure VM
    └── Kind Kubernetes Cluster
        ├── Traefik
        ├── ArgoCD
        ├── Online-Shop
        ├── cert-manager
        └── monitoring
            ├── Prometheus
            ├── Grafana
            ├── Alertmanager
            ├── kube-state-metrics
            └── node-exporter

    The cluster is running inside a single Azure VM.

    This is suitable for:

        Kubernetes learning

        DevOps experimentation

        CI/CD testing

        Monitoring experiments

        GitOps practice

        Development and lab environments

    It should not be considered a highly available production Kubernetes architecture.
    2. Existing Traefik Architecture

    The existing Traefik service uses NodePorts:

    Traefik
        |
        +-- HTTP  -> 30080
        |
        +-- HTTPS -> 30443

    Conceptually:

    Internet
       |
       v
    Azure VM Public IP
       |
       +---- TCP 80
       |
       +---- TCP 443
       |
       v
    Kind
       |
       v
    Traefik NodePort
       |
       +---- 30080
       |
       +---- 30443

    If the existing endpoints:

    https://argocd.yogifods.store
    https://app.yogifods.store

    already work from the Internet, this architecture should be reused rather than redesigned.
    3. Azure Network Requirements

    The Azure VM's Network Security Group (NSG) must permit inbound:

    TCP 80
    TCP 443

    Traffic should ultimately reach the corresponding Kind NodePorts.

    Verify the current architecture before making changes.

    The desired path is:

    Internet
       |
       v
    Azure Public IP
       |
       v
    Azure NSG
       |
       v
    Azure VM
       |
       v
    Kind NodePort
       |
       v
    Traefik

    Do not create another public load balancer solely for Grafana.
    4. DNS

    Create the following DNS record:

    grafana.yogifods.store

    It should point to the same Azure public IP currently used by the other public services.

    Example:

    grafana.yogifods.store -> <AZURE_PUBLIC_IP>

    The resulting public endpoints are:

    argocd.yogifods.store
    app.yogifods.store
    grafana.yogifods.store

    Traefik determines the destination using the HTTP Host header and TLS SNI.
    5. Monitoring Namespace

    Create a dedicated namespace:

    kubectl create namespace monitoring

    Verify:

    kubectl get namespace monitoring

    Expected:

    NAME         STATUS
    monitoring   Active

    6. Install kube-prometheus-stack

    Add the Prometheus Community Helm repository:

    helm repo add prometheus-community \
      https://prometheus-community.github.io/helm-charts

    helm repo update

    Check the available chart:

    helm search repo prometheus-community/kube-prometheus-stack

    Create a working directory:

    mkdir -p ~/monitoring
    cd ~/monitoring

    The recommended approach is to keep the Helm configuration in a values file rather than maintaining a large command-line installation.
    7. Grafana Admin Secret

    Do not store the Grafana administrator password directly inside the Helm values file.

    Create a Kubernetes Secret:

    kubectl -n monitoring create secret generic grafana-admin \
      --from-literal=admin-user=admin \
      --from-literal=admin-password='CHANGE_THIS_TO_A_LONG_RANDOM_PASSWORD'

    Verify:

    kubectl -n monitoring get secret grafana-admin

    For a more production-oriented setup, consider using:

        Azure Key Vault

        External Secrets

        Another dedicated secrets-management solution

    rather than storing credentials directly in Git.
    8. kube-prometheus-stack Values

    Create:

    monitoring-values.yaml

    Recommended baseline configuration:

    grafana:
      enabled: true

      admin:
        existingSecret: grafana-admin
        userKey: admin-user
        passwordKey: admin-password

      persistence:
        enabled: true
        type: pvc
        size: 10Gi
        storageClassName: standard

      service:
        type: ClusterIP
        port: 80

      ingress:
        enabled: false

      grafana.ini:
        server:
          domain: grafana.yogifods.store
          root_url: https://grafana.yogifods.store
          protocol: http


    prometheus:
      enabled: true

      prometheusSpec:
        retention: 15d

        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: standard
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 20Gi

    The important architectural decisions are:

    Grafana       -> ClusterIP
    Prometheus    -> ClusterIP
    Traefik       -> Public ingress

    Grafana itself should not terminate public TLS.

    The TLS flow is:

    Browser
       |
     HTTPS
       |
       v
    Traefik
       |
     HTTP
       |
       v
    Grafana

    9. Install the Monitoring Stack

    Install the chart:

    helm upgrade --install monitoring \
      prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --create-namespace \
      -f monitoring-values.yaml

    Check the release:

    helm list -n monitoring

    Check the pods:

    kubectl get pods -n monitoring

    You should see components similar to:

    alertmanager-...
    grafana-...
    kube-state-metrics-...
    prometheus-...
    prometheus-node-exporter-...
    prometheus-operator-...

    10. Verify Services

    Run:

    kubectl get svc -n monitoring

    Expected services include:

    grafana
    monitoring-kube-prometheus-prometheus
    monitoring-kube-prometheus-alertmanager

    Grafana should remain:

    ClusterIP

    Verify:

    kubectl -n monitoring get svc grafana

    11. Grafana Persistence

    Grafana should use persistent storage.

    The values configuration enables:

    grafana:
      persistence:
        enabled: true
        type: pvc
        size: 10Gi

    This preserves data such as:

        Dashboards

        Users

        Preferences

        Grafana configuration

        Grafana database data

    If dashboards are completely managed through Git, ConfigMaps, or provisioning, Grafana persistence becomes less critical, but enabling it is still recommended for this lab.
    12. Prometheus Persistence

    Prometheus should not use ephemeral storage.

    The recommended configuration is:

    prometheus:
      prometheusSpec:
        retention: 15d

        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: standard
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 20Gi

    Before relying on this configuration, check the available StorageClasses:

    kubectl get storageclass

    Check monitoring PVCs:

    kubectl get pvc -n monitoring

    Because this is a Kind cluster running inside an Azure VM, do not assume that an Azure managed-disk StorageClass is automatically available.
    13. Verify Prometheus

    Prometheus should remain internal.

    Do not create:

    prometheus.yogifods.store

    unless there is a specific requirement to expose the Prometheus UI.

    For temporary local access:

    kubectl -n monitoring port-forward \
      svc/monitoring-kube-prometheus-prometheus \
      9090:9090

    Then open:

    http://localhost:9090

    This keeps Prometheus away from the public Internet.
    14. cert-manager

    The existing cert-manager installation should continue to manage TLS certificates.

    Check cert-manager:

    kubectl get pods -n cert-manager

    Check available ClusterIssuers:

    kubectl get clusterissuer

    For example:

    letsencrypt-prod

    The certificate lifecycle should be:

    Let's Encrypt
         |
         | Certificate
         v
    cert-manager
         |
         | Creates / renews Secret
         v
    grafana-tls
         |
         v
    Traefik
         |
         v
    Grafana

    This keeps certificate lifecycle management centralized.
    15. Grafana Certificate

    Create:

    grafana-certificate.yaml

    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: grafana-tls
      namespace: monitoring
    spec:
      secretName: grafana-tls
      issuerRef:
        name: letsencrypt-prod
        kind: ClusterIssuer
      dnsNames:
        - grafana.yogifods.store

    Apply:

    kubectl apply -f grafana-certificate.yaml

    Check:

    kubectl get certificate -n monitoring

    The certificate should eventually report:

    READY   True

    Check the generated Secret:

    kubectl get secret grafana-tls -n monitoring

    For troubleshooting:

    kubectl describe certificate grafana-tls -n monitoring

    16. Grafana Ingress

    Create:

    grafana-ingress.yaml

    Use the standard Kubernetes Ingress API:

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: grafana
      namespace: monitoring
      annotations:
        traefik.ingress.kubernetes.io/router.entrypoints: websecure
        traefik.ingress.kubernetes.io/router.tls: "true"
    spec:
      ingressClassName: traefik

      tls:
        - hosts:
            - grafana.yogifods.store
          secretName: grafana-tls

      rules:
        - host: grafana.yogifods.store
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: grafana
                    port:
                      number: 80

    Apply:

    kubectl apply -f grafana-ingress.yaml

    Verify:

    kubectl get ingress -n monitoring

    The resulting request flow is:

    https://grafana.yogifods.store
                 |
                 v
              Traefik
                 |
                 v
              Grafana

    17. HTTP to HTTPS Redirect

    HTTP should redirect to HTTPS.

    The Traefik configuration should contain an equivalent configuration:

    ports:
      web:
        port: 80
        http:
          redirections:
            entryPoint:
              to: websecure
              scheme: https
              permanent: true

    The expected behavior is:

    http://grafana.yogifods.store
                  |
                  v
    https://grafana.yogifods.store

    This should also apply consistently to the other public applications.
    18. Grafana External URL

    Grafana should know its externally visible URL:

    grafana:
      grafana.ini:
        server:
          domain: grafana.yogifods.store
          root_url: https://grafana.yogifods.store
          protocol: http

    The protocol between Traefik and Grafana remains HTTP:

    Internet
       |
     HTTPS
       |
       v
    Traefik
       |
     HTTP
       |
       v
    Grafana

    There is no need to make the Grafana pod terminate TLS.
    19. Monitoring Components

    kube-prometheus-stack provides a complete Kubernetes monitoring foundation.

    The stack includes components such as:

    kube-prometheus-stack
    │
    ├── Prometheus
    ├── Grafana
    ├── Alertmanager
    ├── Prometheus Operator
    ├── kube-state-metrics
    └── node-exporter

    This avoids manually assembling each monitoring component.
    20. What Will Be Monitored?

    The monitoring stack provides visibility into:
    Kubernetes

    Cluster
    ├── Nodes
    │   ├── CPU
    │   ├── Memory
    │   ├── Disk
    │   └── Network
    │
    ├── Pods
    ├── Deployments
    ├── StatefulSets
    ├── Namespaces
    └── Replica status

    Traefik

    Traefik
    ├── Requests/sec
    ├── HTTP 2xx
    ├── HTTP 4xx
    ├── HTTP 5xx
    ├── Response latency
    ├── Active connections
    └── Requests by host

    Applications

    Online-Shop
    ├── Pods
    ├── CPU
    ├── Memory
    ├── Availability
    └── Application metrics

    ArgoCD

    ArgoCD
    ├── Applications
    ├── Sync status
    ├── Health status
    └── Controller metrics

    21. Traefik Monitoring

    Traefik is the primary public entry point:

    Internet
       |
       v
    Traefik
       |
       +---- ArgoCD
       |
       +---- Online-Shop
       |
       +---- Grafana

    Therefore, Traefik metrics are particularly valuable.

    Useful dashboards should expose:

        Requests per second

        HTTP 2xx responses

        HTTP 4xx responses

        HTTP 5xx responses

        Response latency

        Active connections

        Requests by hostname

        Requests by route

    For example:

    app.yogifods.store

    Requests:      12,430
    2xx:           99.8%
    4xx:            0.1%
    5xx:            0.1%

    This makes it easier to distinguish application failures from ingress-level problems.
    22. Alertmanager

    Monitoring should not stop at dashboards.

    The desired architecture is:

    Prometheus
        |
        v
    Alertmanager
        |
        +---- Email
        |
        +---- Slack / Teams
        |
        +---- Other notification channels

    kube-prometheus-stack includes Alertmanager.

    Recommended alert categories include:
    Kubernetes

    PodCrashLooping
    PodNotReady
    DeploymentReplicasMismatch
    NodeNotReady
    NodeDiskPressure
    NodeMemoryPressure

    Infrastructure

    CPU > 80%
    Memory > 85%
    Disk > 80%
    Disk nearly full

    Application

    High 5xx rate
    High request latency
    Pod unavailable
    Replica count below desired

    Certificates

    Certificate monitoring is particularly useful because the public architecture depends on TLS.

    Monitor for:

    Certificate expiring soon
    Certificate renewal failure

    This helps detect problems before:

    argocd.yogifods.store   -> certificate failure
    app.yogifods.store      -> certificate failure
    grafana.yogifods.store  -> certificate failure

    23. Security Considerations

    Grafana is externally accessible, so it should be protected appropriately.

    Recommended baseline:

        HTTPS only

        TLS 1.2+

        Strong Grafana administrator password

        Anonymous Grafana access disabled

        Prometheus kept private

        Azure NSG configured appropriately

        No unnecessary NodePorts

        Optional VPN/private access

        Optional SSO/OIDC

        Rate limiting where appropriate

    The public path should remain:

    Internet
        |
        v
    Azure NSG
        |
        v
    Traefik
        |
        +-- TLS
        |
        +-- Security controls
        |
        v
    Grafana

    24. Namespace Layout

    The target namespace layout is:

    kube-system
    │
    ├── Kubernetes system components
    │
    ├── traefik
    │   └── Traefik
    │
    ├── cert-manager
    │   ├── cert-manager
    │   ├── cainjector
    │   └── webhook
    │
    ├── argocd
    │   └── ArgoCD
    │
    ├── online-shop
    │   └── Online Shop
    │
    └── monitoring
        ├── Grafana
        ├── Prometheus
        ├── Alertmanager
        ├── kube-state-metrics
        ├── node-exporter
        └── Prometheus Operator

    25. Final External Architecture

    The final architecture should look like:

                             Azure VM
                                |
                        Public IP / NSG
                                |
                         TCP 80 / 443
                                |
                                v
                             Traefik
                                |
            +-------------------+-------------------+
            |                   |                   |
            v                   v                   v
     argocd.yogifods.store  app.yogifods.store  grafana.yogifods.store
            |                   |                   |
          ArgoCD           Online-Shop            Grafana
                                                    |
                                                    v
                                               Prometheus

    The key rule is:

                     PUBLIC
                        |
                        v
                     Traefik
                        |
           +------------+------------+
           |            |            |
           v            v            v
         ArgoCD      App          Grafana
                                      |
                                      v
                                 Prometheus

    Prometheus remains internal.
    26. High Availability Considerations

    The current architecture is a lab environment rather than a highly available production environment.

    The primary limitation is:

    Azure VM
       |
       v
    Kind
       |
       +-- worker
       +-- worker2

    Both Kubernetes workers ultimately depend on the same Azure VM.

    If the VM fails:

    Traefik       ❌
    ArgoCD        ❌
    Online-Shop   ❌
    Prometheus    ❌
    Grafana       ❌

    Adding Kubernetes replicas does not eliminate the single-VM failure domain.

    For learning and lab purposes, this is reasonable.
    27. Production Architecture

    For a production environment, the Kubernetes layer should be separated from the single-VM Kind architecture.

    A production-oriented design would look more like:

    Azure
      |
      v
    AKS
      |
      +-- Ingress
      +-- cert-manager
      +-- ArgoCD
      +-- Applications
      +-- Monitoring
      +-- Azure-managed storage
      +-- Multiple nodes
      +-- Identity controls
      +-- Network controls
      +-- Backup

    Kind remains useful for:

        Local development

        Learning

        CI

        Testing

        Lab environments
    28. Recommended Implementation Order

    Avoid changing the entire environment at once.

    Follow this sequence:

    1. Verify DNS
           |
           v
    2. Verify Azure TCP 80/443
           |
           v
    3. Verify Traefik
           |
           v
    4. Verify cert-manager ClusterIssuer
           |
           v
    5. Install kube-prometheus-stack
           |
           v
    6. Verify Prometheus
           |
           v
    7. Verify Grafana
           |
           v
    8. Create Grafana Certificate
           |
           v
    9. Create Grafana Ingress
           |
           v
    10. Verify HTTPS
           |
           v
    11. Configure persistence
           |
           v
    12. Configure Grafana authentication
           |
           v
    13. Configure Alertmanager
           |
           v
    14. Add Traefik/application dashboards

    29. Verification Checklist

    Use the following checklist after deployment.
    Azure

    # Verify public IP configuration and NSG externally.

    Confirm:

    TCP 80  -> reachable
    TCP 443 -> reachable

    Kind

    kubectl get nodes

    Confirm nodes are:

    Ready

    Traefik

    kubectl get svc -A | grep traefik

    Confirm:

    80  -> 30080
    443 -> 30443

    cert-manager

    kubectl get pods -n cert-manager
    kubectl get clusterissuer

    Monitoring

    kubectl get pods -n monitoring
    kubectl get svc -n monitoring
    kubectl get pvc -n monitoring

    Grafana certificate

    kubectl get certificate -n monitoring
    kubectl get secret grafana-tls -n monitoring

    Expected:

    READY=True

    Grafana ingress

    kubectl get ingress -n monitoring

    Expected host:

    grafana.yogifods.store

    Grafana

    Open:

    https://grafana.yogifods.store

    Prometheus

    Use port forwarding:

    kubectl -n monitoring port-forward \
      svc/monitoring-kube-prometheus-prometheus \
      9090:9090

    Then access:

    http://localhost:9090

    30. Operational Commands
    View monitoring pods

    kubectl get pods -n monitoring

    View monitoring services

    kubectl get svc -n monitoring

    View PVCs

    kubectl get pvc -n monitoring

    View Grafana logs

    kubectl logs -n monitoring deploy/monitoring-grafana

    View Prometheus logs

    kubectl logs -n monitoring \
      statefulset/prometheus-monitoring-kube-prometheus-prometheus

    Describe Grafana

    kubectl describe pod -n monitoring \
      -l app.kubernetes.io/name=grafana

    Describe certificate

    kubectl describe certificate grafana-tls \
      -n monitoring

    Describe ingress

    kubectl describe ingress grafana \
      -n monitoring

    31. Design Principles

    The architecture follows these principles:
    Single public ingress

    Traefik remains the single public entry point.

    Internet
       |
       v
    Traefik

    Centralized certificate management

    cert-manager owns the certificate lifecycle.

    Let's Encrypt
          |
          v
    cert-manager
          |
          v
    Kubernetes Secret
          |
          v
    Traefik

    Internal monitoring services

    Grafana and Prometheus are Kubernetes services.

    Grafana    -> ClusterIP
    Prometheus -> ClusterIP

    Only Grafana is exposed through Traefik.
    TLS termination at the edge

    Internet
       |
     HTTPS
       |
       v
    Traefik
       |
     HTTP
       |
       v
    Grafana

    Persistent monitoring data

    Prometheus and Grafana use PVCs.
    Infrastructure observability

    The monitoring stack covers:

    Kubernetes
       +
    Nodes
       +
    Pods
       +
    Traefik
       +
    Applications
       +
    ArgoCD
       +
    Certificates

    32. Final Recommendation

    For the current Kind-on-Azure lab, keep the architecture simple:

    Azure VM
       |
       v
    Kind
       |
       v
    Traefik
       |
       +---- ArgoCD
       |
       +---- Online-Shop
       |
       +---- Grafana
                 |
                 v
             Prometheus

    Use:

    Traefik
        -> Single public ingress

    cert-manager
        -> TLS certificate lifecycle

    Let's Encrypt
        -> Public TLS certificates

    kube-prometheus-stack
        -> Prometheus
        -> Grafana
        -> Alertmanager
        -> kube-state-metrics
        -> node-exporter

    PVCs
        -> Persistent monitoring data

    Most importantly:

        Do not create another NodePort for Grafana. Keep Grafana as a ClusterIP service and expose it through the existing Traefik ingress.

    The desired public endpoint is therefore:

    https://grafana.yogifods.store

    with:

    Let's Encrypt
          |
          v
    cert-manager
          |
          v
    grafana-tls
          |
          v
    Traefik
          |
          v
    Grafana

    This keeps the architecture consistent with the existing ArgoCD and Online-Shop deployments while providing a scalable foundation for Kubernetes monitoring and alerting.

    This is structured as a repository-ready README, with the architecture, manifests, commands, verification steps, security considerations, and production/lab distinction included.

