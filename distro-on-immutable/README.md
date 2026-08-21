# SIGHUP Distribution on Immutable machines

This tutorial shows how to deploy a full SIGHUP Distribution (SD) cluster with the `Immutable` kind.
The `Immutable` kind installs [Flatcar Container Linux][flatcar-site] on the machines over the network,
and then installs Kubernetes and SD on them. The machines can be bare-metal machines or virtual machines.

> [!WARNING]
> The Immutable Installer is in **alpha** status and is under active development. Its configuration and
> behavior can change between releases.

<!-- spacer -->

> [!TIP]
> 💻 If your machines already have an operating system, use the [SIGHUP Distribution on VMs][distro-on-vms]
> tutorial instead. ☁️ To try SD in a cloud environment, use the [SIGHUP Distribution on EKS][distro-on-eks]
> tutorial.

The goal of this tutorial is to show the main concepts of SD and how to work with its tooling.

## What makes this kind different

The `Immutable` kind starts from bare machines. There is no operating system to prepare and no SSH access
to set up. `furyctl` does this work for you:

1. `furyctl` generates one [iPXE][ipxe] script and two [Ignition][ignition] configurations for each machine.
   The MAC address of the machine is the key that selects the correct files.
2. `furyctl` starts an HTTP boot server on port `8080` and waits.
3. Each machine boots over the network, installs Flatcar to its disk, and reboots.
4. Each machine reports the `booted` status to `furyctl`.
5. After all the machines report `booted`, `furyctl` runs Ansible and builds the cluster.

Flatcar has a read-only `/usr` partition and an A/B update scheme. You do not patch packages on a node.
Each node gets its full configuration from Ignition at first boot. Automatic OS updates are off, because
the installer masks `update-engine` and sets `REBOOT_STRATEGY=off`. `furyctl` drives the A/B update during
a cluster upgrade.

For the full description of the flow, read the [Immutable Installation Guide][immutable-install].

## Prerequisites

This tutorial assumes some basic familiarity with Kubernetes.

### Tooling

- **furyctl** — 0.35.1 or later. It downloads and manages the other tools that the installation needs.
- **kubectl** — 1.35.x, to interact with the cluster.
- **openssl** — to create the TLS certificate for the ingresses.
- A machine that runs `furyctl`, often a bastion host. Every cluster machine must reach TCP port `8080`
  on this machine.

### Machines

You need 8 machines:

| Role          | Quantity | Minimum size    |
| ------------- | -------- | --------------- |
| Load balancer | 2        | 1 vCPU, 1 GB RAM  |
| Control plane | 3        | 2 vCPU, 4 GB RAM  |
| Worker        | 3        | 4 vCPU, 8 GB RAM  |

Each machine also needs:

- An empty install disk, for example `/dev/sda`. The installer writes over its content. If the disk
  already holds a Flatcar installation, the installer stops instead.
- A stable IP address. This tutorial uses DHCP with a reservation for each machine.
- Network boot first in the boot order.
- A way to start [iPXE][ipxe]. Read [Network boot](#network-boot) below.

### Network

- A **DHCP server** that you can modify. It must send **option 67** to the machines, so that they reach
  the `furyctl` boot server. The [existing DHCP with PXE flags][immutable-case-dhcp] case documents
  the rules to add. If you cannot modify your DHCP server, read the
  [Immutable Installation Guide][immutable-install]. Then select one of the other three boot cases.
- A **DNS server** that resolves the name of each machine to its IP address. The tutorial uses the
  `example.com` domain, for example `cp1.example.com`.
- One free IP address for the `keepalived` virtual IP. The tutorial uses `192.168.1.179`.
- A DNS record `control-plane.example.com` that points to the virtual IP.
- A wildcard DNS record `*.sighup.example.com` that points to the virtual IP.
- Access to the internet, to download the Flatcar images and the container images.

> [!IMPORTANT]
> `furyctl` does not run DHCP, TFTP, or DNS. It serves only the boot script, the Ignition configurations,
> and the Flatcar assets. The DHCP and DNS setup is yours.

### Network boot

Each machine must run [iPXE][ipxe], and then chain to the `furyctl` boot server at
`http://<furyctl-host>:8080/boot/${mac:hexhyp}`. How iPXE starts depends on the machine:

- **The network card runs iPXE already.** Many server network cards have an iPXE ROM. iPXE sends the DHCP
  user class `iPXE`, and your DHCP server answers with the URL of the boot server. This path is the
  shortest one. You do not need an `ipxe.efi` file, and you do not need a second HTTP server.
- **The firmware does not run iPXE.** The machine must load iPXE first. With UEFI HTTP Boot (UEFI 2.5+),
  DHCP gives the firmware the URL of an `ipxe.efi` file. You host this file on your own HTTP server, and
  you get it from the [iPXE download page][ipxe-download]. `furyctl` does not supply this file. Virtual
  machines usually need this path, because their UEFI firmware has no iPXE ROM.

> [!TIP]
> You can avoid a second HTTP server. The boot server of `furyctl` serves every file in
> `<outdir>/.furyctl/<cluster-name>/infrastructure/server`, so a copy of `ipxe.efi` in that folder is
> available at `http://<furyctl-host>:8080/ipxe.efi`. `furyctl` creates this folder on the first run and
> does not delete your file on the next runs. The folder is the working directory of `furyctl`, so keep
> your own copy of the file.

> [!NOTE]
> `furyctl` makes no assumption about the firmware. Its boot script loads the Flatcar kernel and initrd
> only. The two DHCP cases in the installer documentation describe the UEFI HTTP Boot path, because that
> path needs no TFTP server.

## Step 0 - Setup and initialize the environment

1. Open a terminal.

2. Clone the [getting started repository][getting-started] that holds the example code of this tutorial:

    ```bash
    git clone https://github.com/sighupio/getting-started/
    cd getting-started/distro-on-immutable
    ```

## Step 1 - Install furyctl

Install the `furyctl` binary with the instructions in [furyctl's documentation][furyctl-installation].

Always install the latest version. The latest versions work with previous versions of the distribution,
and they can include more bug fixes. This tutorial needs furyctl 0.35.1 or later: it is the first release
that accepts SD v1.35.1 for the `Immutable` kind. Run this command to see the version:

```bash
furyctl version
```

## Step 2 - Create the SSH key pair

`furyctl` writes the public key into the `core` user on every machine. Then it uses the private key to
connect to the machines. Create the key pair in the tutorial directory:

```bash
ssh-keygen -t ed25519 -f ./ssh-key -C "sd-getting-started" -N ""
```

> [!NOTE]
> This command creates the `ssh-key` and `ssh-key.pub` files. Both paths are already in the example
> `furyctl.yaml` file.

## Step 3 - Initialize the PKI

Kubernetes uses TLS to encrypt the traffic in the control plane, and between the control plane and its
clients. First, you must initialize the Certificate Authorities for Kubernetes and for the etcd database.
Then you must create the certificates for each component. `furyctl` does all of this with one command:

```bash
furyctl create pki
```

> [!TIP]
> To see the advanced options, run `furyctl create pki --help`.

<!-- spacer -->

> [!NOTE]
> For more information, read the
> [Kubernetes security documentation](https://kubernetes.io/docs/concepts/security/#control-plane-protection).

After the command completes, the `pki` folder has this content:

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

## Step 4 - Create the TLS certificate for the ingresses

SD uses the HTTPS protocol for its ingresses. This tutorial uses a self-signed certificate, because the
cluster is not reachable from the internet.

Run these commands to create the `ca.crt`, `tls.crt`, and `tls.key` files:

```bash
openssl genrsa -out ca-key.pem 2048
openssl req -x509 -new -days 365 -nodes -key ca-key.pem -out ca.crt -subj "/CN=kube-ca"
openssl genrsa -out tls.key 2048
openssl req -new -key tls.key -out csr.pem -subj "/CN=kube-ca" -config req-dns.cnf
openssl x509 -req -in csr.pem -CA ca.crt -CAkey ca-key.pem -CAcreateserial -out tls.crt -days 365 -extensions v3_req -extfile req-dns.cnf
```

The `req-dns.cnf` file is in the tutorial directory. It has this content:

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

Change the two `DNS` values to your own domain.

> [!TIP]
> If your load balancers are reachable from the internet, you can use cert-manager instead. SD includes
> cert-manager and integrates it with the distribution. The [SIGHUP Distribution on VMs][distro-on-vms]
> tutorial shows the `dns01` challenge with Route 53.

## Step 5 - Write the `furyctl.yaml` configuration file

`furyctl` reads the cluster configuration from a `furyctl.yaml` file. The tutorial directory holds a
complete example file. Use it as your starting point.

> [!TIP]
> To create a new file from scratch instead, run:
>
> ```bash
> furyctl create config --kind Immutable --version v1.35.1 --name getting-started
> ```

This step explains the main fields of the file.

### `.spec.infrastructure`

#### SSH access and boot server

```yaml
spec:
  infrastructure:
    ssh:
      username: core
      privateKeyPath: ./ssh-key
      publicKeyPath: ./ssh-key.pub
    ipxeServer:
      url: http://furyctl.example.com:8080
```

`ssh.username` selects the account that `furyctl` uses to reach the machines. This tutorial uses `core`,
the built-in Flatcar account, but you can name a different account. `furyctl` writes the public key into
this account on every machine.

> [!TIP]
> To add more accounts, use the `passwd.users` section of a node. It follows the Butane `passwd` schema.
> Read the [Flatcar configuration reference][flatcar-passwd].

`ipxeServer.url` is the address of the boot server that `furyctl` starts. Every machine downloads its
boot script and its Ignition configurations from this address. Use the name or the IP address of the
machine that runs `furyctl`.

> [!IMPORTANT]
> The machines must reach this URL during the boot. A firewall between the machines and the boot server
> makes the installation stop.

<!-- spacer -->

> [!TIP]
> By default, the boot server binds to the host and the port of this URL. If the host does not resolve to
> a local address, add `bindAddress: 0.0.0.0` below `ipxeServer`. You can also change the port with
> `bindPort`.

#### Machines

```yaml
spec:
  infrastructure:
    nodes:
      - hostname: cp1.example.com
        macAddress: "52:54:00:01:00:01"
        arch: x86-64
        storage:
          installDisk: /dev/sda
        network:
          ethernets:
            eth0:
              dhcp4: true
```

The `nodes` list holds every machine of the cluster: the load balancers, the control-plane machines,
and the workers. This list gives the hardware description of each machine. The role of a machine comes
later, from the `.spec.infrastructure.loadBalancers` and `.spec.kubernetes` sections.

Each entry has four important fields:

- `hostname` — the full name of the machine. Your DNS must resolve this name to the IP address below.
- `macAddress` — the MAC address of the boot interface. `furyctl` uses it to send the correct
  configuration to each machine.
- `storage.installDisk` — the disk that receives Flatcar. **The installer writes over the content of this
  disk.** If the disk already holds a Flatcar installation, the installer stops and reports the
  `installation-blocked` status instead.
- `network.ethernets` — the network configuration.

> [!IMPORTANT]
> The key below `ethernets` is the real name of the network interface, for example `eth0` or `enp1s0`.
> `furyctl` writes it into a `systemd-networkd` match rule. A wrong name leaves the machine without
> network.

<!-- spacer -->

> [!TIP]
> This tutorial uses DHCP, because the network already has a DHCP server. To give a machine a fixed
> address instead, replace `dhcp4: true` with:
>
> ```yaml
>             eth0:
>               addresses:
>                 - 192.168.1.181/24
>               gateway: 192.168.1.1
>               nameservers:
>                 addresses:
>                   - 192.168.1.1
> ```

<!-- spacer -->

> [!IMPORTANT]
> Set `arch` on every node. The field accepts `x86-64` and `arm64`, and one cluster can mix the two
> architectures.

#### Load balancers

```yaml
spec:
  infrastructure:
    loadBalancers:
      members:
        - hostname: lb1.example.com
        - hostname: lb2.example.com
      keepalived:
        enabled: true
        interface: eth0
        ip: 192.168.1.179
        virtualRouterId: "201"
        passphrase: "b16cf069"
      haproxy:
        configuration: "{file://./haproxy.cfg}"
```

The two load balancer machines run HAProxy in a container. `keepalived` moves a virtual IP address
between them, so one machine can fail without loss of service.

> [!IMPORTANT]
> The `interface` field must hold the real name of the interface that receives the virtual IP.

<!-- spacer -->

> [!NOTE]
> If two `keepalived` clusters share a network, give each one a different `virtualRouterId`.

The `haproxy.configuration` field **replaces** the default HAProxy configuration. The default one load
balances only the Kubernetes API server. The `haproxy.cfg` file in the tutorial directory load balances
the API server **and** the HAProxy Ingress Controller:

```text
frontend k8s-api-server
    mode tcp
    bind *:6443 alpn h2,http/1.1
    default_backend control-plane
    timeout client 50s

backend control-plane
    mode tcp
    option httpchk GET /healthz
    balance roundrobin
    timeout connect 5s
    timeout server 50s
    server cp1.example.com cp1.example.com:6443 check check-ssl ca-file /usr/local/etc/haproxy/kubernetes.crt
    ...

frontend ingress-http
    mode tcp
    bind *:80
    default_backend ingress-http
    timeout client 50s

backend ingress-http
    mode tcp
    balance roundrobin
    timeout connect 5s
    timeout server 50s
    server worker1.example.com worker1.example.com:30080 maxconn 256 check
    ...
```

The HAProxy Ingress Controller of SD listens on the node ports `30080` and `30443`. The load balancers
send the traffic of port `80` and port `443` to these node ports.

> [!NOTE]
> The installer always adds a `prometheus` frontend on port `8405`. Do not add it to your own file.

### `.spec.kubernetes`

```yaml
spec:
  kubernetes:
    pkiPath: ./pki
    networking:
      podCIDR: 172.16.128.0/17
      serviceCIDR: 172.16.0.0/17
    controlPlane:
      address: control-plane.example.com:6443
      members:
        - hostname: cp1.example.com
        - hostname: cp2.example.com
        - hostname: cp3.example.com
    nodeGroups:
      - name: worker
        nodes:
          - hostname: worker1.example.com
          - hostname: worker2.example.com
          - hostname: worker3.example.com
```

`pkiPath` points to the folder from step 3. `controlPlane.address` is the DNS record that points to the
virtual IP of the load balancers.

`podCIDR` and `serviceCIDR` are the networks of the Pods and of the Kubernetes services. These networks
must not overlap the IP addresses of the machines or other networks that the cluster must reach.

`controlPlane.members` and `nodeGroups[].nodes` select machines from the `.spec.infrastructure.nodes`
list. The `hostname` values must be the same.

> [!TIP]
> To run etcd on separate machines, add a `.spec.kubernetes.etcd.members` list. Without this list, etcd
> runs on the control-plane machines.

#### Custom registry for the Kubernetes core components

You can change the registry that supplies the images of the Kubernetes core components. This registry gives
`kubeadm` its images (`kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, and
`coredns`), and also the `pause` and `kubelet-csr-approver` images. The host is mandatory. The port is
optional.

```yaml
spec:
  kubernetes:
    advanced:
      registry: <registry-host>[:<registry-port>]
```

For a mirror of the official SIGHUP Distribution registry, add `/fury/on-premises` at the end. The
default value is `registry.sighup.io/fury/on-premises`.

### `.spec.distribution`

#### Networking core module

```yaml
spec:
  distribution:
    modules:
      networking:
        type: cilium
```

This section installs Cilium as the CNI (Container Network Interface) from the `module-networking`
core module.

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
            provider: secret
            secret:
              cert: "{file://./tls.crt}"
              key: "{file://./tls.key}"
              ca: "{file://./ca.crt}"
        certManager:
          clusterIssuer:
            name: letsencrypt-sighup
            email: example@sighup.io
            type: http01
```

This section installs one battery of HAProxy Ingress Controller from the `module-ingress` core module.
The certificate from step 4 becomes the default certificate of the controller.

`baseDomain` is the suffix of every ingress of the SD modules. For example, Grafana becomes
`grafana.sighup.example.com`.

> [!NOTE]
> The `certManager` section is mandatory, also when `provider` is `secret`. Other SD components need
> certificates too, not only the ingresses. In this configuration, Cilium uses cert-manager to issue the
> Hubble certificates.

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

This section configures the `module-logging` core module. Loki stores the logs. The Logging Operator
sends the logs to Loki with its Flows and Outputs. `retentionTime` keeps the logs of one day only, to
keep this test cluster small.

The `minio` section deploys a MinIO cluster with the S3 bucket that Loki uses to store the logs.
`storageSize` is the size of each MinIO disk. MinIO has 3 replicas with 2 disks each, so 6 disks in total.

#### Monitoring core module

```yaml
spec:
  distribution:
    modules:
      monitoring:
        type: prometheus
```

This section configures the `module-monitoring` core module with the full Prometheus stack.

#### Policy core module and Tracing core module

```yaml
spec:
  distribution:
    modules:
      policy:
        type: none
      tracing:
        type: none
```

To keep this tutorial short, it does not install a policy engine (Gatekeeper or Kyverno) and it does not
install a tracing solution (Tempo).

#### DR core module

```yaml
spec:
  distribution:
    modules:
      dr:
        type: on-premises
        velero: {}
```

This section installs Velero from the `module-dr` core module, for the backups of the cluster. Velero uses a
MinIO instance to store the backups.

#### Auth core module

```yaml
spec:
  distribution:
    modules:
      auth:
        provider:
          type: none
```

This section configures the authentication of the ingresses, and the OIDC authentication to the
Kubernetes API server. To keep this tutorial short, both are disabled.

#### Custom registry for the distribution phase

You can also change the registry that supplies the images of the SD core modules. The host is mandatory.
The port is optional.

```yaml
spec:
  distribution:
    common:
      registry: <registry-host>[:<registry-port>]
```

For a mirror of the official SIGHUP Distribution registry, add `/fury` at the end. The default value is
`registry.sighup.io/fury`.

> [!WARNING]
> If the plugins pull from the official SIGHUP Distribution registry, this value replaces the registry of
> the plugins too.

### `.spec.plugins`

```yaml
spec:
  plugins:
    kustomize:
      - name: local-storage
        folder: ./local-storage
```

This section installs additional plugins in the cluster. A plugin is a `helm` chart or a `kustomize`
project. This tutorial installs one Kustomize project.

The `local-storage` folder installs the `local-path-provisioner` and makes it the default storage class.
It gives the cluster a simple dynamic storage.

> [!WARNING]
> `local-path-provisioner` is not a production-grade storage solution. Use it only for a test cluster.

## Step 6 - Run the installation with `furyctl`

The configuration is complete. Now `furyctl` can create the cluster.

1. Start the installation:

    ```bash
    furyctl apply --outdir $PWD --post-apply-phases distribution
    ```

    > [!NOTE]
    > The `--outdir` flag selects the directory of the hidden `.furyctl` folder. This folder holds all the
    > files of the installation. Without the flag, `furyctl` uses the home directory of the user.
    >
    > The `--post-apply-phases distribution` flag repeats the distribution phase at the end of the run.
    > The first run of the phase finds no storage class, because the plugins phase installs it later.
    > The second run installs the components that need storage.

2. Wait for this message:

    ```text
    WARN Assets server started. You can boot your machines now
    INFO Press ENTER to skip waiting and continue, or CTRL+C to cancel and exit.
    ```

3. Power on all the machines. Every machine boots over the network, installs Flatcar, and reboots.

4. Watch the table of the machines. `furyctl` refreshes it after each report:

    ```text
    Nodes bootstrap status — 3/8 booted
      NODE                 STATUS   UPDATED
      cp1.example.com      booted   12:05:02
      cp2.example.com      pending  —
      cp3.example.com      pending  —
      lb1.example.com      booted   12:04:31
      lb2.example.com      booted   12:04:38
      worker1.example.com  pending  —
      worker2.example.com  pending  —
      worker3.example.com  pending  —
    ```

5. After all the machines report `booted`, `furyctl` stops the server and continues alone.

> [!NOTE]
> A machine with the `installation-blocked` status already has Flatcar on its install disk. The
> installer does not overwrite it. Erase the disk, then start the machine again.

> [!TIP]
> ⏱ The full process takes some minutes. To follow it in detail, run this command in a second terminal:
>
> ```bash
> tail -f .furyctl/furyctl.<timestamp>-<random-id>.log | jq -j '.msg'
> ```

The output is similar to this one:

```text
INFO Downloading distribution...
INFO Validating configuration file...
INFO Downloading dependencies...
INFO Tools ready (7 installed via mise)
INFO Installing ansible collections...
INFO Validating dependencies...
INFO Running preflight checks...
INFO Preflight checks completed successfully
INFO Running preupgrade phase...
INFO Preupgrade phase completed successfully
INFO Flatcar installation butane files generated successfully
INFO Downloading Flatcar boot artifacts and sysext packages...
INFO Assets download completed successfully
WARN Assets server started. You can boot your machines now
INFO Press ENTER to skip waiting and continue, or CTRL+C to cancel and exit.
INFO All 8 nodes reached 'booted' state. Stopping server and continuing...
INFO Server stopped
INFO Applying nodes configuration...
INFO Configuring SIGHUP Distribution Kubernetes cluster...
INFO Checking that the hosts are reachable...
INFO Applying cluster configuration...
INFO SIGHUP Distribution Kubernetes cluster configured successfully
INFO Configuring SIGHUP Distribution modules...
INFO Checking that the cluster is reachable...
INFO Checking for a default storage class...
WARN No storage classes found in the cluster. logging module (if enabled), tracing module (if enabled), dr module (if enabled) and prometheus-operated package installation will be skipped. Install a *default* StorageClass and re-run furyctl to install the missing components.
INFO Applying Distribution modules...
INFO SIGHUP Distribution configured successfully
INFO Applying plugins...
INFO Plugins installed successfully
INFO Executing extra phases: distribution...
INFO Configuring SIGHUP Distribution modules...
INFO Checking for a default storage class...
INFO Applying Distribution modules...
INFO SIGHUP Distribution configured successfully
INFO Saving furyctl configuration file in the cluster...
INFO Saving distribution configuration file in the cluster...
```

🚀 Success! The cluster is ready.

`furyctl` writes a `kubeconfig` file in the working directory. Use it with `kubectl`:

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes
```

All 6 Kubernetes nodes must reach the `Ready` state. The two load balancers are not Kubernetes nodes,
so they do not appear in this list.

For a summary of the cluster, run:

```bash
furyctl get cluster-info
```

The command shows the SD version and kind, the installer version, the available upgrade paths, the
Kubernetes version, the etcd topology, the installed modules, and the plugins. Add `--format json` or
`--format yaml` for a machine-readable output.

## Step 7 - Explore the distribution

### Forecastle

[Forecastle](https://github.com/stakater/Forecastle) is an open-source control panel. It shows all the
exposed applications that run on Kubernetes.

Open https://directory.sighup.example.com to see the other ingresses, grouped by namespace.

![Forecastle][forecastle-screenshot]

> [!NOTE]
> The browser shows a warning, because the certificate from step 4 is self-signed. Accept the warning
> to continue.

### Grafana

[Grafana](https://github.com/grafana/grafana) is an open-source platform for monitoring and
observability. With Grafana you can query, visualize, and understand your metrics, and you can create
alerts.

Open https://grafana.sighup.example.com, or click the Grafana icon in Forecastle.

#### Discover the logs

SD installs the Grafana Logs Drilldown app. It shows the logs of each service, and you do not write a
query.

1. Click **Drilldown** in the left menu, then click **Logs**.
2. Select a service, for example `loki-distributed`.
3. Read the log volume by level. Then use the **Labels**, **Fields**, and **Patterns** tabs to narrow the
   result.

This is the result:

![Grafana Logs][grafana-screenshot-logs]

#### Discover the metrics dashboards

SD gives you a set of dashboards for the state of the cluster and its workload. To open one of them:

1. Click the search icon in the left sidebar.
2. Write `pods` and press Enter.
3. Select the `Kubernetes/Pods` dashboard.

This is the result:

![Grafana][grafana-screenshot]

## Conclusions

Congratulations, you made it! 🥳🥳

We hope you enjoyed this tour of SIGHUP Distribution!

### Issues/Feedback

If this tutorial has an error, [open an issue in the getting-started repository][gs-issues].

If the installation fails, [open an issue in the SIGHUP Distribution repository][sd-issues]. That repository
has issue templates, and its maintainers triage the components that this tutorial installs.

### Where to go next?

More tutorials:

- [SIGHUP Distribution on VMs][distro-on-vms]
- [SIGHUP Distribution on EKS][distro-on-eks]
- [SIGHUP Distribution on Minikube][distro-on-minikube]

More about SD:

- [SD Documentation][docs]
- [SD Production-grade Installation][docs-prod-install]
- [Immutable Installation Guide][immutable-install]

<!-- Links -->
[distro-on-minikube]: https://github.com/sighupio/getting-started/tree/main/distro-on-minikube
[distro-on-eks]: https://github.com/sighupio/getting-started/tree/main/distro-on-eks
[distro-on-vms]: https://github.com/sighupio/getting-started/tree/main/distro-on-vms
[getting-started]: https://github.com/sighupio/getting-started
[gs-issues]: https://github.com/sighupio/getting-started/issues/new
[sd-issues]: https://github.com/sighupio/distribution/issues/new/choose
[docs]: https://docs.sighup.io
[docs-prod-install]: https://docs.sighup.io/docs/installation
[furyctl-installation]: https://github.com/sighupio/furyctl#installation
[immutable-install]: https://github.com/sighupio/installer-immutable/blob/main/docs/IMMUTABLE_INSTALL.md
[immutable-case-dhcp]: https://github.com/sighupio/installer-immutable/blob/main/docs/install-case-dhcp-pxe.md
[flatcar-site]: https://www.flatcar.org/
[flatcar-passwd]: https://coreos.github.io/butane/config-flatcar-v1_1/
[ignition]: https://coreos.github.io/ignition/
[ipxe]: https://ipxe.org/
[ipxe-download]: https://ipxe.org/download

<!-- Images -->
[grafana-screenshot]: https://github.com/sighupio/getting-started/blob/media/grafana.png?raw=true
[grafana-screenshot-logs]: https://github.com/sighupio/getting-started/blob/media/grafana-logs.png?raw=true
[forecastle-screenshot]: https://github.com/sighupio/getting-started/blob/media/forecastle_immutable.png?raw=true
