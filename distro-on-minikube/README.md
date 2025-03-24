# SIGHUP Distribution on minikube

This step-by-step tutorial helps you deploy a subset of the **SIGHUP Distribution** on a local minikube cluster.

This tutorial covers the following steps:

1. Deploy a local minikube cluster.
2. Download the latest `furyctl` CLI.
3. Install SKD distribution using `furyctl` CLI.
4. Explore some features of the distribution.
5. Teardown the environment.

> ☁️ If you prefer trying SKD in a cloud environment, check out the [SIGHUP Distribution on EKS][distro-on-eks] tutorial.

The goal of this tutorial is to introduce you to the main concepts of SKD and how to work with its tooling.

## Prerequisites

This tutorial assumes some basic familiarity with Kubernetes.

To follow this tutorial, you need:

- **minikube** - follow the [installation guide](https://minikube.sigs.k8s.io/docs/start/). You can run the minikube cluster with one of the drivers listed.
- **kubectl** - to interact with the cluster.

### Setup and initialize the environment

1. Open a terminal

2. Clone the [getting started repository](https://github.com/sighupio/getting-started) containing all the example code used in this tutorial:

```bash
git clone https://github.com/sighupio/getting-started/
cd getting-started/distro-on-minikube
```

## Step 1 - Start the minikube cluster

1. Start minikube cluster:

    ```bash
    export REPO_DIR=$PWD
    export KUBECONFIG=$REPO_DIR/kubeconfig
    minikube start --kubernetes-version v1.31.5 --memory=16384m --cpus=6
    ```

    > ⚠️ This command will spin up by default a single-node Kubernetes v1.31.5 cluster, using the default driver, with 6 CPUs, 16GB RAM and 20 GB Disk.

2. Test the connection to the minikube cluster:

    ```bash
    kubectl get nodes
    ```

    Output:

    ```bash
    NAME       STATUS   ROLES           AGE   VERSION
    minikube   Ready    control-plane   9s    v1.31.5
    ```

## Step 3 - Install furyctl

Install `furyctl` binary following the instructions in [furyctl's documentation][furyctl-installation].

## Step 3 - Installation

In this directory, an example `furyctl.yaml` file is present.

`furyctl` will use the provider `KFDDistribution` that will install only the Distribution on top of an existing cluster.

```yaml
apiVersion: kfd.sighup.io/v1alpha2
kind: KFDDistribution
metadata:
  name: fury-local
spec:
  distributionVersion: v1.31.0
  distribution:
    kubeconfig: "{env://KUBECONFIG}"
    modules:
      networking:
        type: none
      ingress:
        baseDomain: demo.example.internal
        nginx:
          type: single
          tls:
            provider: certManager
        certManager:
          clusterIssuer:
            name: letsencrypt-fury
            email: example@sighup.io
            type: http01
      logging:
        type: loki
        loki:
          tsdbStartDate: "2024-12-03"
      monitoring:
        type: prometheus
      policy:
        type: none
      dr:
        type: none
        velero: {}
      auth:
        provider:
          type: none
    customPatches:
      patchesStrategicMerge:
        - |
          $patch: delete
          apiVersion: logging-extensions.banzaicloud.io/v1alpha1
          kind: HostTailer
          metadata:
            name: systemd-common
            namespace: logging
        - |
          $patch: delete
          apiVersion: logging-extensions.banzaicloud.io/v1alpha1
          kind: HostTailer
          metadata:
            name: systemd-etcd
            namespace: logging
        - |
          $patch: delete
          apiVersion: apps/v1
          kind: DaemonSet
          metadata:
            name: x509-certificate-exporter-control-plane
            namespace: monitoring
```

In this example, we are installing the distribution with the following options:

- No CNI installation, minikube comes with a CNI by default
- A single battery of nginx
- Loki as storage for the logs
- No gatekeeper installation
- No velero and DR installation
- No Auth on the ingresses
- Disabled some logging extensions due to minikube incompatibilities
- Disabled master certificate-exporter, due to minikube incompatibilities

> ℹ️ Usually, when using the dual ingress controller, the `internal.<ingress domain>` base domain is specified. For this occurence, since there is
> just a single NGINX Ingress (for the purposes of this guide), only the ingress base domain is configured. Both, single or dual ingress configuration, are valid.
> Feel free to edit the furyctl.yaml file according to your needs. For more information see
> [Ingress NGINX Dual](https://docs.sighup.io/docs/components/modules/ingress/dual-nginx) and
> [Ingress NGINX Single](https://docs.sighup.io/docs/components/modules/ingress/nginx) documentation pages.

Execute the installation with furyctl:

```bash
furyctl apply --outdir $PWD
```

> ⏱ The process will take some minutes to complete, you can follow the progress in detail by running the following command:
>
> ```bash
> tail -f .furyctl/furyctl.<timestamp>-<random-id>.log | jq
> ```
>
> `--outdir` flag is used to define in which directory to create the hidden `.furyctl` folder that contains all the required files to install the cluster.
> If not provided, a `.furyctl` folder will be created in the user home.

The output should be similar to the following:

```bash
INFO Downloading distribution...
INFO Validating configuration file...
INFO Downloading dependencies...
INFO Validating dependencies...
INFO Running preflight checks
INFO Checking that the cluster is reachable...
INFO Cannot find state in cluster, skipping...
INFO Running preupgrade phase...
INFO Preupgrade phase completed successfully
INFO Installing Kubernetes Fury Distribution...
INFO Checking that the cluster is reachable...
INFO Checking storage classes...
INFO Checking if all nodes are ready...
INFO Applying manifests...
INFO Kubernetes Fury Distribution installed successfully
INFO Applying plugins...
INFO Skipping plugins phase as spec.plugins is not defined
INFO Saving furyctl configuration file in the cluster...
INFO Saving distribution configuration file in the cluster...
```

🚀 The (subset of the) distribution is finally deployed! In this section you will explore some of its features.

## Step 4 - Explore the distribution

### Setup local DNS

In Step 3, alongside the distribution, you have deployed Kubernetes ingresses to expose underlying services at the following HTTP routes:

- `directory.demo.example.internal`
- `grafana.demo.example.internal`
- `prometheus.demo.example.internal`

To access the ingresses more easily via the browser, configure your local DNS to resolve the ingresses to the external minikube IP:

1. Get the address of the cluster IP:

    ```bash
    minikube ip
    <SOME_IP>
    ```

2. Add the following line to your local `/etc/hosts`:

    ```bash
    <SOME_IP> directory.demo.example.internal grafana.demo.example.internal prometheus.demo.example.internal
    ```

Now, you can reach the ingresses directly from your browser.

> ℹ️ Note
>
>If you are running minikube on macOS or Windows using Docker Desktop, you will need to port-forward the NGINX Ingress ports to localhost to enable access to your exposed applications.
>For example, you can run:
>
> ```bash
>  kubectl port-forward service/ingress-nginx -n ingress-nginx 31080:80 31443:443
>  # Output:
>  Forwarding from 127.0.0.1:31080 -> 8080
>  Forwarding from 127.0.0.1:31443 -> 8443
>```
>
>This command will forward both HTTP and HTTPS ports of the NGINX Ingress Controller to your localhost on ports 31080 and 31443 respectively.
>Leave that terminal window open while you need access to your applications. Inside your /etc/hosts file you can put 127.0.0.1 in place of minikube's IP.

### Forecastle

[Forecastle](https://github.com/stakater/Forecastle) is an open-source control panel where you can access all exposed applications running on Kubernetes.

Navigate to https://directory.demo.example.internal:31443 to see all the other ingresses deployed, grouped by namespace.

![Forecastle][forecastle-screenshot]

### Grafana

[Grafana](https://github.com/grafana/grafana) is an open-source platform for monitoring and observability. Grafana allows you to query, visualize, alert, and understand your metrics.

Navigate to https://grafana.demo.example.internal:31443 or click the Grafana icon from Forecastle (remember to append the port 31443 to the url).

#### Discover the logs

Navigate to grafana, and:

1. Click on explore
2. Select Loki datasource
3. Run your query!

This is what you should see:

![Grafana Logs][grafana-screenshot-logs]

#### Discover dashboards

Fury provides some pre-configured dashboards to visualize the state of the cluster. Examine an example dashboard:

1. Click on the search icon on the left sidebar.
2. Write `pods` and click enter.
3. Select the `Kubernetes/Pods` dashboard.

![Grafana][grafana-screenshot]

Make sure to select a namespace that has pods running and then select one of those pods.

Take a look around and test the other dashboards available.

## Step 6 - Tear down

1. Delete the minikube cluster:

    ```bash
    minikube delete
    ```

## Conclusions

Congratulations, you made it! 🥳🥳

We hope you enjoyed this tour of Fury!

### Issues/Feedback

In case you ran into any problems feel free to [open an issue in GitHub](https://github.com/sighupio/fury-getting-started/issues/new).

### Where to go next?

More tutorials:

- [SIGHUP Distribution on EKS][distro-on-eks]
- [SIGHUP Distribution on VMs][distro-on-vms]

More about Fury:

- [Documentation][docs]

<!-- Links -->
[distro-on-eks]: https://github.com/sighupio/getting-started/tree/main/distro-on-eks
[distro-on-vms]: https://github.com/sighupio/getting-started/tree/main/distro-on-vms
[furyctl-installation]: https://github.com/sighupio/furyctl#installation
[docs]: https://docs.sighup.io

<!-- Images -->
[grafana-screenshot]: https://github.com/sighupio/fury-getting-started/blob/media/grafana.png?raw=true
[grafana-screenshot-logs]: https://github.com/sighupio/fury-getting-started/blob/media/grafana-logs.png?raw=true
[forecastle-screenshot]: https://github.com/sighupio/fury-getting-started/blob/media/forecastle_minikube.png?raw=true
