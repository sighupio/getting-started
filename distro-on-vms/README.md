# SIGHUP Distribution on VMs

This step-by-step tutorial helps you deploy a full SIGHUP Distribution (SD) cluster on a set of already existing VMs.

> [!TIP]
> ☁️ If you prefer trying SD in a cloud environment, check out the [SIGHUP Distribution on EKS][distro-on-eks] tutorial.

The goal of this tutorial is to introduce you to the main concepts of SD and how to work with its tooling.

## Prerequisites

This tutorial assumes some basic familiarity with Kubernetes.

To follow this tutorial, you need:

- **kubectl** - 1.36.x to interact with the cluster.
- VMs OS: RHEL 8, RHEL 9, Rocky Linux 8, Rocky Linux 9, Debian 12, Debian 13, Alma Linux 9, Ubuntu 20, or Ubuntu 24
- Valid FQDN for all the VMs, with a valid domain: for example, each VM should have a corresponding DNS entry like `worker1.example.com`, `worker2.example.com`, `master1.example.com`, etc.
- Fixed IP address for each VM.
- Two VMs for the load balancer Nodes (at least 1vCPU 1GB RAM each)
- An additional IP that will be used by keepalived to expose the two load balancers in HA, and a DNS record pointed to this IP for the control-plane address.
- Three VMs for the control-plane nodes (at least 2vCPU and 4GB RAM each)
- Three VMs for the worker nodes (at least 4vCPU and 8GB RAM each)
- `root` or passwordless sudo user SSH access to the VMs
- Route53 credentials with permission to modify a zone (optional)

> [!WARNING]
> Support for the ARM platform is still in beta status, the Load Balancers installed with the `haproxy` role are not currently supported for RHEL and RHEL derivatives running on ARM. Please use a different OS for the Load Balancers VMs (or disable them and create your own load balancer).

## Step 0 - Setup and initialize the environment

1. Open a terminal

2. Clone the [getting started repository](https://github.com/sighupio/getting-started) containing all the example code used in this tutorial:

    ```bash
    git clone https://github.com/sighupio/getting-started/
    cd getting-started/distro-on-vms
    ```

## Step 1 - Install furyctl

Install `furyctl` binary following the instructions in [furyctl's documentation][furyctl-installation].

We recommend always installing the latest version available. Latest versions are compatible with previous versions of the distribution and may include additional bug-fixes to the distribution. This guide assumes that the furyctl version is at least 0.36.0. You can check with the following command:

```bash
furyctl version
```

## Step 2 - Initialize the PKI

Kubernetes expects you to configure and use TLS to provide data encryption in transit within the control plane, and between the control plane and its clients. To initialize the Public Key Infrastructure needed to create the TLS certificates for the several components of the cluster, first, we need to initialize the Certificate Authorities for Kubernetes and for the etcd database and then create the certificates for each component. Luckily for us, `furyctl` can create for you all of this by running the following command:

```bash
furyctl create pki
```

> [!TIP]
> See the command's help for advanced options: `furyctl create pki --help`.

<!-- spacer -->

> [!NOTE]
> Learn more on [Kubernetes security documentation](https://kubernetes.io/docs/concepts/security/#control-plane-protection).

After the initialization of the PKI, you should have a `pki` folder with the following contents:

```text
pki
├── etcd
│   ├── ca.crt
│   └── ca.key
└── master
    ├── ca.crt
    ├── ca.key
    ├── front-proxy-ca.crt
    ├── front-proxy-ca.key
    ├── sa.key
    └── sa.pub
```

## Step 3 - Decide the strategy for the Ingress TLS certificates

We use the HTTPS protocol to expose the SD ingresses securely. HTTPS relies on certificates that need to be present, there are two approaches to achieve this:

1) Provide the certificates and keys (like a self-signed certificate)
2) Use cert-manager to generate the certificates

### Self-signed certificate

If you are using the first approach, and you don't have the needed `tls.key`, `tls.crt`, and `ca.crt` files you can generate a self-signed set using `openssl` or similar tools.

To generate these files using `openssl`, you can run the following commands:

```bash
openssl genrsa -out ca-key.pem 2048
openssl req -x509 -new -days 365 -nodes -key ca-key.pem -out ca.crt -subj "/CN=kube-ca"
openssl genrsa -out tls.key 2048
openssl req -new -key tls.key -out csr.pem -subj "/CN=kube-ca" -config req-dns.cnf
openssl x509 -req -in csr.pem -CA ca.crt -CAkey ca-key.pem -CAcreateserial -out tls.crt -days 365 -extensions v3_req -extfile req-dns.cnf
```

The file `req-dns.cnf` is already present the tutorial directory, with the following content:

```text
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = sighup.example.com
DNS.2 = *.sighup.example.com
```

Change the values according to your environment

### cert-manager

With cert-manager you can get valid certificates automatically created for you. You can use the `http01` challenge to get certificates from Let's Encrypt if your load-balancer is reachable from the Internet, otherwise, we suggest using the `dns01` solvers that use an authoritative DNS zone to emit certificates.

SD includes cert-manager in its core packages, and it is fully integrated with the distribution. We will use cert-manager with the `dns01` challenge approach in this tutorial.

## Step 4 - Write the `furyctl.yaml` configuration file

The next step is to write the cluster configuration file used by `furyctl`, in the tutorial directory you can find a pre-compiled file that you can use as a starting point.

We will explain in this step, what the main fields are for.

### `.spec.kubernetes`

#### PKI and access

```yaml
---
spec:
  kubernetes:
    pkiFolder: ./pki
    ssh:
      username: root
      keyPath: ./ssh-key
```

This first piece of configuration defines where to find the PKI files (created in step 1), and the SSH connection details for the `root` user.
`keyPath` can be a relative or an absolute path to the SSH private key.

#### Common DNS zone and networking

```yaml
---
spec:
  kubernetes:
    dnsZone: example.com
    controlPlaneAddress: control-plane.example.com:6443
    podCidr: 172.16.128.0/17
    svcCidr: 172.16.0.0/17
```

Next we need to define the DNS zone used by all the nodes, and the control-plane address. Also, we need to define the network CIDR for the Pods and the network CIDR for the Kubernetes services used by the cluster. These CIDRs must not collide with the IPs of the nodes or other external networks that should be reachable from within the cluster.

#### Load Balancers configuration

```yaml
spec:
  kubernetes:
    loadBalancers:
      enabled: true
      hosts:
        - name: haproxy1
          ip: 192.168.1.177
        - name: haproxy2
          ip: 192.168.1.178
      keepalived:
        enabled: true
        interface: enp0s8
        ip: 192.168.1.179/32
        virtualRouterId: "201"
        passphrase: "b16cf069"
      stats:
        username: admin
        password: password
      additionalConfig: "{file://./haproxy-additional.cfg}"
```

Next we need to define the load-balancer nodes, each node will have a name and an IP address; additionally, we are also enabling keepalived on an additional floating (virtual) IP address, in this example `192.168.1.179`.

> [!IMPORTANT]
> Check which is the main interface that will be used for the keepalived IP, in this example the interface name is `enp0s8`.

We can give to the HAProxy statistics ("stats") page a username and a password, and we can also add additional custom configuration to the HAProxy running the load balancers. For example, we are using the additional configuration to load balance the ingress battery using the same load balancers as the control plane address.

#### Kubernetes Control-plane and Worker nodes

```yaml
spec:
  kubernetes:
    masters:
      hosts:
        - name: master1
          ip: 192.168.1.181
        - name: master2
          ip: 192.168.1.182
        - name: master3
          ip: 192.168.1.183
    nodes:
      - name: worker
        hosts:
          - name: worker1
            ip: 192.168.1.184
          - name: worker2
            ip: 192.168.1.185
          - name: worker3
            ip: 192.168.1.186
        taints: []
```

Next we define the control-plane nodes and the worker nodes. The FQDN that will be used for each node will be the concatenation of the name and the `.spec.kubernetes.dnsZone` field.

For example, `master1` will become `master1.example.com`.

#### Custom registry for Kubernetes core components images

We can override the URL of the registry where to pull images from for the Kubernetes core components (kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy, coredns). The host is mandatory, while the port is optional. For mirrors of the official SIGHUP Distribution registry, append `/fury/on-premises` at the end (the default value is `registry.sighup.io/fury/on-premises`).

```yaml
spec:
  kubernetes:
    advanced:
      registry: <registry-host>[:<registry-port>]
```

### `.spec.distribution`

#### Networking core module

```yaml
spec:
  distribution:
    modules:
      networking:
        type: calico
```

In this piece of configuration, we are choosing to install Calico as CNI (Container Network Interface) in our cluster from the `networking` core module.

#### Ingress core module

```yaml
spec:
  distribution:
    modules:
      ingress:
        baseDomain: sighup.example.com
        nginx:
          type: none
        haproxy:
          type: single
          tls:
            provider: certManager
        certManager:
          clusterIssuer:
            name: letsencrypt-sighup
            email: example@sighup.io
            solvers:
              - dns01:
                  route53:
                    region: eu-west-1
                    accessKeyID: AKIAEXAMPLE
                    secretAccessKeySecretRef:
                      name: letsencrypt-production-route53-key
                      key: secret-access-key
```

In this section, on the configuration of the `module-ingress` core module, we are selecting to install a single HAProxy Kubernetes Ingress Controller and configuring cert-manager as the provider to emit TLS certificates for our ingresses.
`baseDomain` is the suffix hostname used on all the ingresses that will be created for the SD modules, for example, Grafana will become `grafana.<baseDomain>`.

To correctly configure the cert-manager clusterIssuer, we need to put a valid configuration for the `dns01` challenge solver. The secret `letsencrypt-production-route53-key` will be created using furyctl's plugins feature in the next steps.

> If instead you want to use a self-signed certificate (or a valid one from a file), you need to configure the ingress module like the following:
>
> ```yaml
>spec:
>  distribution:
>    modules:
>      ingress:
>        baseDomain: sighup.example.com
>        nginx:
>          type: none
>        haproxy:
>          type: single
>          tls:
>            provider: secret
>            secret:
>              cert: "{file://./tls.crt}"
>              key: "{file://./tls.key}"
>              ca: "{file://./ca.crt}" # this field can be empty if `cert` has the full chain and the CA is trusted (like Let's Encrypt)
>        certManager:
>          clusterIssuer:
>            name: letsencrypt-sighup
>            email: example@sighup.io
>            type: http01
> ```

#### Logging core module

```yaml
spec:
  distribution:
    modules:
      logging:
        type: loki
        loki:
          retentionTime: "1d"
        minio:
          storageSize: "20Gi"
```

This section configures the `module-logging` module. In this example we are installing Loki as log storage, and configuring the Logging operator with all the Flows and Outputs to send logs to the Loki stack.

The minio configuration section is to deploy a MinIO cluster with the S3 bucket used by Loki to store logs, the storageSize selected defines the size for each MinIO disk, in total 6 disks split in 2 per 3 MinIO replicas.

#### Monitoring core module

```yaml
spec:
  distribution:
    modules:
      monitoring:
        type: prometheus
```

This section configures the `monitoring` module, which installs a complete monitoring stack with Prometheus based on kube-prometheus and other monitoring tools.

#### Policy (OPA) core module and Tracing core module

```yaml
spec:
  distribution:
    modules:
      policy:
        type: none
      tracing:
        type: none
```

For simplicity, we are not installing a policy system (Gatekeeper or Kyverno) and a tracing solution (Tempo) in the cluster as part of this getting started guide.

#### DR core module

```yaml
spec:
  distribution:
    modules:
      dr:
        type: on-premises
        velero: {}
```

We are also configuring Velero for the cluster backups from the `module-dr` module. Velero will be deployed with a minio instance used to store all the backups.

#### Auth core module

```yaml
spec:
  distribution:
    modules:
      auth:
        provider:
          type: none
```

This section can be used to configure the authentication for the ingresses and the authentication via OIDC to the Kubernetes API Server, for simplicity we are disabling the authentication on the ingresses and not configuring the OIDC authentication for the API server.

#### Utilities core module

```yaml
spec:
  distribution:
    modules:
      utilities:
        headlamp:
          type: token-auth
          clusterRole: view
```

This section installs [Headlamp][headlamp] from the `module-utilities` core module: a web UI for Kubernetes, extended with plugins for the SD modules (alerts, certificates, logs, backups, and policies). Headlamp is exposed at `dashboard.<baseDomain>`, in this example `dashboard.sighup.example.com`.

With `type: token-auth`, users log in with a Kubernetes token and get the permissions of that token. The optional `clusterRole` field binds the `view` ClusterRole to the `headlamp` ServiceAccount, so that you can use its token to log in with read-only access to the cluster.

> [!NOTE]
> You can also set `type: sso` to put Headlamp behind Pomerium: users log in with SSO and the group in `oidcGroup` gets access to the cluster. This type requires `auth.provider.type: sso`. Set `type: none` to disable Headlamp.

#### Custom registry for distribution phase

Just like we can override the registry for the Kubernetes components, we can also override the URL of the registry where to pull images from for the SD core modules. The host is mandatory, while the port is optional. For mirrors of the official SIGHUP Distribution registry, append `/fury` at the end (the default value is `registry.sighup.io/fury`).

```yaml
spec:
  distribution:
    common:
      registry: <registry-host>[:<registry-port>]
```

> [!WARNING]
> If plugins are pulled from the official SIGHUP Distribution registry, the registry will be replaced for the plugins too.

### `.spec.distribution.customResources`

```yaml
spec:
  distribution:
    customResources:
      - ./local-storage/
```

This section adds custom resources to the distribution phase, furyctl applies them together with the SD core modules. In this example, the `local-storage` folder installs the `local-path-provisioner` and makes it the default storage class.

### `.spec.plugins`

```yaml
spec:
  plugins:
    kustomize:
      - name: cert-manager-secret
        folder: ./cert-manager-secret/
```

This section configures additional plugins to be installed in the cluster. There can be two types of plugin, `helm` and `kustomize`, in this example we are installing one Kustomize project.

The `cert-manager-secret` folder installs the secret used by cert-manager to interact with the route53 zone for the dns01 challenge. Change the example values in the `./cert-manager-secret` folder with the correct credentials to interact with your route53 zone.

## Step 5 - Launch the installation with `furyctl`

Now that everything is configured you can proceed with applying the configuration using the `furyctl` CLI.

Simply execute:

```bash
furyctl apply --outdir $PWD
```

> [!TIP]
> ⏱ The process will take some minutes to complete, you can follow the progress in detail by running the following command:
>
> ```bash
> tail -f .furyctl/furyctl.<timestamp>-<random-id>.log | jq -j '.msg'
> ```
>
> `--outdir` flag is used to define in which directory to create the hidden `.furyctl` folder that contains all the required files to install the cluster.
> If not provided, a `.furyctl` folder will be created in the user's home directory.

The output should be similar to the following:

```bash
INFO Downloading distribution...                  
INFO Validating configuration file...             
INFO Downloading dependencies...                  
INFO Tools ready (7 installed via mise)           
INFO Installing ansible collections...            
INFO Validating dependencies...                   
INFO Running preflight checks...                  
INFO Preflight checks completed successfully      
INFO Configuring SIGHUP Distribution cluster...   
INFO Checking that the hosts are reachable...     
INFO Applying cluster configuration...            
INFO Kubernetes cluster configured successfully   
INFO Installing SIGHUP Distribution...            
INFO Checking that the cluster is reachable...    
INFO Checking for a default storage class...      
WARN No StorageClass found in the cluster. furyctl skips these packages: dr, logging, tracing, prometheus-operated. furyctl checks again later in this apply and, if a default StorageClass exists by then, applies the distribution phase again to install them. If not, install a default StorageClass and run furyctl again. 
INFO Applying Distribution modules...             
INFO SIGHUP Distribution installed successfully   
INFO A default StorageClass is now available ("local-path"). Applying the distribution phase again to install the packages that were skipped: dr, logging, tracing, prometheus-operated 
INFO Installing SIGHUP Distribution...            
INFO Checking that the cluster is reachable...    
INFO Applying Distribution modules...             
INFO SIGHUP Distribution installed successfully   
INFO Applying plugins...                          
INFO Plugins installed successfully               
INFO Saving furyctl configuration file in the cluster... 
INFO Saving distribution configuration file in the cluster... 
```

🚀 Success! The cluster is ready.

> [!TIP]
> To speed up the following executions, you can use `--skip-deps-download`. This works only if the `.furyctl` folder has been already initialized.

To interact with the cluster a `kubeconfig` has been created on the folder, make it usable with `kubectl` with:

```bash
export KUBECONFIG=$PWD/kubeconfig
```

To see a summary of the cluster, you can run `furyctl get cluster-info`. It shows the SD version and kind, the installer version, the available upgrade paths, the Kubernetes version, and the installed modules and plugins. Add `--format json` or `--format yaml` for a machine-readable output.

```bash
furyctl get cluster-info
```

## Step 6 - Explore the distribution

### Forecastle

[Forecastle](https://github.com/stakater/Forecastle) is an open-source control panel where you can access all exposed applications running on Kubernetes.

Navigate to https://directory.sighup.example.com to see all the other ingresses deployed, grouped by namespace.

![Forecastle][forecastle-screenshot]

### Grafana

[Grafana](https://github.com/grafana/grafana) is an open-source platform for monitoring and observability. Grafana allows you to query, visualize, alert, and understand your metrics.

Navigate to https://grafana.sighup.example.com or click the Grafana icon from Forecastle.

#### Discover the logs

Navigate to Grafana, and:

1. Click on explore
2. Select Loki datasource
3. Run your query!

This is what you should see:

![Grafana Logs][grafana-screenshot-logs]

#### Discover the metrics dashboards

SD provides some pre-configured dashboards to visualize the state of the cluster and its workload. Examine an example dashboard:

1. Click on the search icon on the left sidebar.
2. Write `pods` and press Enter.
3. Select the `Kubernetes/Pods` dashboard.

This is what you should see:

![Grafana][grafana-screenshot]

### Headlamp

[Headlamp][headlamp] is a web UI for Kubernetes. The SD plugins show the state of the cluster and of the SD modules in one place.

1. Create a token for the `headlamp` ServiceAccount:

    ```bash
    kubectl -n headlamp create token headlamp --duration 8h
    ```

2. Navigate to https://dashboard.sighup.example.com or click the Headlamp icon from Forecastle.
3. Paste the token and log in.

![Headlamp][headlamp-screenshot]

## Conclusions

Congratulations, you made it! 🥳🥳

We hope you enjoyed this tour of SIGHUP Distribution!

### Issues/Feedback

In case you ran into any problems with this tutorial, feel free to [open an issue in the getting-started repository](https://github.com/sighupio/getting-started/issues/new).

If the installation itself failed, please [open an issue in the SIGHUP Distribution repository](https://github.com/sighupio/distribution/issues/new/choose) instead: it has issue templates, and its maintainers triage the components installed here.

### Where to go next?

More tutorials:

- [SIGHUP Distribution on EKS][distro-on-eks]
- [SIGHUP Distribution on Minikube][distro-on-minikube]

More about SD:

- [SD Documentation][docs]
- [SD Production-grade Installation][docs-prod-install]

<!-- Links -->
[distro-on-minikube]: https://github.com/sighupio/getting-started/tree/main/distro-on-minikube
[distro-on-eks]: https://github.com/sighupio/getting-started/tree/main/distro-on-eks
[docs]: https://docs.sighup.io
[docs-prod-install]: https://docs.sighup.io/docs/installation
[furyctl-installation]: https://github.com/sighupio/furyctl#installation
[headlamp]: https://github.com/kubernetes-sigs/headlamp

<!-- Images -->
[grafana-screenshot]: https://github.com/sighupio/getting-started/blob/media/grafana.png?raw=true
[grafana-screenshot-logs]: https://github.com/sighupio/getting-started/blob/media/grafana-logs.png?raw=true
[forecastle-screenshot]: https://github.com/sighupio/getting-started/blob/media/forecastle_minikube.png?raw=true
[headlamp-screenshot]: https://github.com/sighupio/getting-started/blob/media/headlamp.png?raw=true
