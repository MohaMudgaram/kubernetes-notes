# Installing Kubernetes

> Reference Book:
> 
> Managing Kubernetes
> 
> By Brendan Burns and Craig Tracey

---

## kubeadm

Among the wide array of Kubernetes installation solutions is the community-supported kubeadm utility. **This application provides all of the functionality needed to install Kubernetes**.

In fact, in the simplest of cases, **a user can have a Kubernetes installation operational in a matter of minutes—with just a single command**.

This simplicity makes it **a very compelling tool for developers and for those with production-grade needs**.

Because the code for kubeadm lives in-tree and is released in conjunction with a Kubernetes release, it borrows common primitives and is thoroughly tested for a large number of use cases.

> Because of the simplicity and great utility provided by kubeadm, many other installation tools actually leverage kubeadm behind the scenes. And the number of projects following this trend increases regularly. So, regardless of whether you ultimately choose kubeadm as your preferred installation tool, understanding how it works will likely help you further understand the tool you have chosen.

A production-grade deployment of Kubernetes ensures that data is secured, both during transport and at rest, that the Kubernetes components are well matched with their dependencies, that integrations with the environment are well defined, and that the configuration of all the cluster components work well together.

Ideally, too, these clusters are easily upgraded and the resulting configuration is continually reflective of these best practices. kubeadm can help you achieve all of this.

### Requirements

kubeadm, just like all of the Kubernetes binaries, is statically linked. As such, there are no dependencies on any shared libraries, and **kubeadm may be installed on just about any x86_64, ARM, or PowerPC Linux distribution**.

Fortunately, there is also not much that we need from a host application perspective, either. Most fundamentally,** we require a container runtime and the Kubernetes kubelet, but there are also a few necessary standard Linux utilities**.

**When it comes to installing a container runtime, you should ensure that it adheres to the Container Runtime Interface (CRI)**.

This open standard defines the interface that the kubelet uses to speak to the runtime available on the host. At the time of this writing, **some of the most popular CRI-compliant runtimes are Docker, rkt, and CRI-O**. For each of these, developers should consult the installation instructions provided by the respective projects.

> When choosing a container runtime, be sure to reference the Kubernetes release notes. Each release will clearly indicate which container runtimes have been tested. This way you know which runtimes and versions are known to be both compatible and performant.

### kubelet

As you may recall, **the kubelet is the on-host process responsible for interfacing with the container runtime**.

In the most common cases, **this work typically amounts to reporting node status to the API server and managing the full lifecycle of Pods that have been scheduled to the host on which it resides**.

Installation of the kubelet is usually as simple as downloading and installing the appropriate package for the target distribution. In all cases, you should be sure to install the kubelet with a version that matches the Kubernetes version you intend to run.

**The kubelet is the single Kubernetes process that is managed by the host service manager. In almost all cases, this is likely to be `systemd`**.

If you are installing the kubelet with the system packages built and provided by the community (currently `deb` and `rpm`), the kubelet will be managed by *systemd*.

As with any process managed in this way, a unit file defines which user the service runs as, what the command-line options are, how the service dependency chain is defined, and what the restart policy should be:

```
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=http://kubernetes.io/docs/

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
```

> Even if you are not installing the kubelet with the community-provided packages, examining the provided unit files is often be helpful for understanding common best practices for running the kubelet daemon. These unit files change often, so be sure to reference the versions that match your deployment target.

The behavior of the kubelet can be manipulated by adding additional unit files to the */etc/systemd/system/kubelet.service.d/* path. These unit files are read lexically (so name them appropriately) and allow you to override how the package configures the kubelet. This may be required if your environment calls for specific needs (i.e., container registry proxies).

For example, when deploying Kubernetes to a supported cloud provider, you need to set the `--cloud-provider` parameter on the kubelet:

```
$ cat /etc/systemd/system/kubelet.service.d/09-extra-args.conf
[Service]
Environment="KUBELET_EXTRA_ARGS= --cloud-provider=aws"
```

With this additional file in place, we simply perform a daemon reload and then restart the service:

```
$ sudo systemctl daemon-reload
$ sudo systemctl restart kubelet
```

By and large, the default configurations provided by the community are typically more than adequate and usually do not require modification. With this technique, however, we can utilize the community defaults while simultaneously maintaining our ability to override, when applicable.

The kubelet and container runtime are necessary on all hosts in the cluster.

## Installing the Control Plane

Within Kubernetes **the componentry that directs the actions of the worker nodes is termed the control plane**.

**These components consist of the API server, the controller manager, and the scheduler**. Each of these daemons directs some portion of how the cluster ultimately operates.

In addition to the Kubernetes components, **we require a place to store our cluster state. That data store is etcd**.

Fortunately, **kubeadm is capable of installing all of these daemons on a host (or hosts) that we, as administrators, have delegated as a control plane node**. **kubeadm does so by creating a static manifest for each of the daemons** that we require.

> With static manifests, we can write Pod specifications directly to disk, and the kubelet, upon start, immediately attempts to launch the containers specified therein. In fact, the kubelet also monitors these files for changes and attempts to reconcile any specified changes. Note, however, that since these Pods are not managed by the control plane, they cannot be manipulated with the kubectl command-line interface.

In addition to the daemons, **we need to secure the components with Transport Layer Security (TLS), create a user that can interact with the API, and provide the capability for worker nodes to join the cluster**. **Kubeadm does all of this**.

In the simplest of scenarios, **we can install the control plane components on a node that has already been prepared with a running kubelet and functional container runtime**, like so:

```
$ kubeadm init
```

After detailed descriptions of the steps kubeadm has taken on behalf of the user, the end of the output might look something like this:

```
...
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 878b76.ddab3219269370b2 10.1.2.15:6443 \
      --discovery-token-ca-cert-hash \
      sha256:312ce807a9e98d544f5a53b36ae3bb95cdcbe50cf8d1294d22ab5521ddb54d68
```

### kubeadm Configuration

Although `kubeadm init` is the simplest case for configuring a controller node, **kubeadm is capable of managing all sorts of configurations**. This can be achieved by way of the various but somewhat limited number of kubeadm command-line flags, as well as the more capable kubeadm API.

The API looks something like this:

```
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: <address|string>
  bindPort: <int>
etcd:
  endpoints:
  - <endpoint1|string>
  - <endpoint2|string>
  caFile: <path|string>
  certFile: <path|string>
  keyFile: <path|string>
networking:
  dnsDomain: <string>
  serviceSubnet: <cidr>
  podSubnet: <cidr>
kubernetesVersion: <string>
cloudProvider: <string>
authorizationModes:
- <authorizationMode1|string>
- <authorizationMode2|string>
token: <string>
tokenTTL: <time duration>
selfHosted: <bool>
apiServerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
controllerManagerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
schedulerExtraArgs:
  <argument>: <value|string>
  <argument>: <value|string>
apiServerCertSANs:
- <name1|string>
- <name2|string>
certificatesDir: <string>
```

It may be provided to the kubeadm command line with the `--config` flag.

Regardless of whether you, as an administrator, decide to explicitly use this configuration format, one is always generated internally upon executing `kubeadm init`.

Moreover, this configuration is saved as a `ConfigMap` to the just-provisioned cluster. This functionality serves two purposes—first, to provide a reference for those who need to understand how a cluster was configured, and second, it may be leveraged when upgrading a cluster. In the event of upgrading a cluster, a user modifies the values of this `ConfigMap` and then executes a `kubeadm upgrade`.

> The kubeadm configuration is also accessible by standard `kubectl` `ConfigMap` interrogation and is, by convention, named the `cluster-info` `ConfigMap` in the `kube-public` namespace.

### Preflight Checks

After we have run this command, kubeadm first executes a number of preflight checks. These sanity checks ensure that our system is appropriate for an install. “Is the kubelet running?”, “Has swap been disabled?”, and “Are baseline system utilities installed?” are the types of questions that are asked here.

And, naturally, **kubeadm exits with an error if these baseline conditions are not met**.

> Although not recommended, it is possible to sidestep the preflight checks with the `--skip-preflight-checks` command-line option. This should only be exercised by advanced administrators.

### Certificates

After all preflight checks have been satisfied, **kubeadm, by default, generates its own certificate authority (CA) and key**.

**This CA is then used to, subsequently, sign various certificates against it. Some of these certificates are used by the API server when securing inbound requests, authenticating users, making outbound requests (i.e., to an aggregate API server), and for mutual TLS between the API server and all downstream kubelets**.

Others are used to secure service accounts.

All of these public key infrastructure (PKI) assets are placed in the */etc/kubernetes/pki* directory on the control plane node:

```
$ ls -al /etc/kubernetes/pki/
total 56
drwxr-xr-x 2 root root 4096 Mar 15 02:42 .
drwxr-xr-x 4 root root 4096 Mar 15 02:42 ..
-rw-r--r-- 1 root root 1229 Mar 15 02:42 apiserver.crt
-rw------- 1 root root 1675 Mar 15 02:42 apiserver.key
-rw-r--r-- 1 root root 1099 Mar 15 02:42 apiserver-kubelet-client.crt
-rw------- 1 root root 1679 Mar 15 02:42 apiserver-kubelet-client.key
-rw-r--r-- 1 root root 1025 Mar 15 02:42 ca.crt
-rw------- 1 root root 1675 Mar 15 02:42 ca.key
-rw-r--r-- 1 root root 1025 Mar 15 02:42 front-proxy-ca.crt
-rw------- 1 root root 1675 Mar 15 02:42 front-proxy-ca.key
-rw-r--r-- 1 root root 1050 Mar 15 02:42 front-proxy-client.crt
-rw------- 1 root root 1675 Mar 15 02:42 front-proxy-client.key
-rw------- 1 root root 1675 Mar 15 02:42 sa.key
-rw------- 1 root root  451 Mar 15 02:42 sa.pub
```

> Since this default CA is self-signed, any third-party consumers need to also provide the CA certificate chain when attempting to use a client certificate. Fortunately, this is not typically problematic for Kubernetes users, since a `kubeconfig` file is capable of embedding this data, and is done automatically by `kubeadm`.

Self-signed certificates, although extremely convenient, are sometimes not the preferred approach. This is often especially true in corporate environments or for those with more exacting compliance requirements.

In this case, **a user may prepopulate these assets in the */etc/kubernetes/pki* directory prior to executing `kubeadm init`. In this case, kubeadm attempts to use the keys and certificates that may already be in place and to generate those that may not already be present**.

### etcd

In addition to the Kubernetes components that are configured by way of kubeadm, by default, if not otherwise specified, kubeadm attempts to start a local etcd server instance.

This daemon is started in the same manner as the Kubernetes components (static manifests) and persists its data to the control plane node’s filesystem via local host volume mounts.

> At the time of this writing, `kubeadm init`, by itself, does not natively secure the kubeadm-managed etcd server with TLS. This basic command is only intended to configure a single control plane node and, is typically, for development purposes only.
> 
> Users that need kubeadm for production installs should provide a list of TLS-secured etcd endpoints with the `--config` option.

Although having an easily deployable etcd instance is favorable for a simple Kubernetes installation process, it is not appropriate for a production-grade deployment.

In a production-grade deployment, **an administrator deploys a multinode and highly available etcd cluster that sits adjacent to the Kubernetes deployment**.

Since **the etcd data store will contain all states for the cluster**, it is important to treat it with care.

Although Kubernetes components are easily replaceable, etcd is not. And, as a result, **etcd has a component life cycle (install, upgrade, maintenance, etc.) that is quite different**.

For this reason, a production Kubernetes cluster should segregate these responsibilities.

### Secrets data

All data that is written to etcd is unencrypted by default. If someone were to gain privileged access to the disk backing etcd, the data would be readily available. Fortunately, much of the data that Kubernetes persists to disk is not sensitive in nature.

The exception, however, is Secret data. As the name suggests, Secret data should remain, secret.

**To ensure that this data is encrypted on its way to etcd, administrators ought to the `--experimental-encryption-provider-config` `kube-apiserver` parameter**. With this parameter, **administrators can define symmetric keys to encrypt all Secret data**.

> At the time of this writing, `--experimental-encryption-provider-config` is still an experimental kube-apiserver command-line parameter. Since this is subject to change, native support for this feature in kubeadm is limited. You may still make use of this feature by adding `encryption.conf` to the */etc/kubernetes/pki* directory of all control plane nodes and by adding this configuration parameter to the `apiServerExtraArgs` field within your kubeadm MasterConfig prior to `kubeadm init`.

This is accomplished with an `EncryptionConfig`:

```
$ cat encryption.conf
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
    - secrets
    providers:
    - identity: {}
    - aescbc:
        keys:
        - name: encryptionkey
          secret: BHk4lSZnaMjPYtEHR/jRmLp+ymazbHirgxBHoJZqU/Y=
```

For the recommended `aescbc` encryption type, the `secret` field should be a randomly generated 32-byte key.

Now, **by adding `--experimental-encryption-provider-config=/path/to/encryption.conf` to the `kube-apiserver` command-line parameters, all Secrets are encrypted before being written to etcd. This may help prevent the leakage of sensitive data**.

You may have noticed that the `EncryptionConfig` also includes a `resources` field. For our use case, Secrets are the only resources we want to encrypt, but any resource type may be included here. Use this according to your organization’s needs, but remember that encrypting this data does marginally impact performance of the API server’s writes. **As a general rule of thumb, only encrypt the data that you deem sensitive**.

This configuration supports multiple encryption types, some of which may be more or less appropriate for your specific needs. Likewise, **this configuration also supports key rotation, a measure that is necessary to ensure a strong security stance**. Be sure to consult the Kubernetes documentation for additional details on this experimental feature.

> Your requirements for data at rest will have dependencies on your architecture. If you have chosen to colocate your etcd instances on your control plane nodes, utilizing this feature might not serve your needs, since encryption keys would also be colocated with the data. In the event that privileged access to the disk is gained, the keys may be used to unencrypt the etcd data, thus subverting efforts to secure these resources. This is yet another compelling reason for segregating etcd from your Kubernetes control plane nodes.

### kubeconfig

In addition to creating the PKI assets and configuring the static manifests that serve the Kubernetes components, **kubeadm also generates a number of *kubeconfig* files**.

Each of these files will be used for some means of authentication. **Most of these will be used to authenticate each of the Kubernetes services against the API**, but kubeadm also creates a primary administrator *kubeconfig* file at */etc/kubernetes/admin.conf*.

> Because kubeadm so easily creates this *kubeconfig* with cluster administrator credentials, many users tend to use these generated credentials for much more than their intended use. These credentials should be used only for bootstrapping a cluster. Any production deployment should always configure additional identity mechanisms.

### Taints

For production use cases, we recommend that user workloads be isolated from the control plane components.

As such, **kubeadm taints all control plane nodes with the `node-role.kubernetes.io/master` taint. This instructs the scheduler to ignore all nodes with this taint, when determining where a Pod may be placed**.

If your use case is that of a single-node master, you may remove this restriction by removing the taint from the node:

```
kubectl taint nodes <node name> node-role.kubernetes.io/master-
```

## Installing Worker Nodes

Worker nodes follow a very similar installation mechanism. Again, **we require the container runtime and the kubelet on every node**.

But, for workers, **the only other Kubernetes component that we need is the `kube-proxy` daemon**. And, just as with the control plane nodes, **kubeadm starts this process by way of another static manifest**.

Most significantly, **this process performs a TLS bootstrapping sequence**.

Through a shared token exchange process, **kubeadm temporarily authenticates the node against the API server and then attempts to perform a certificate signing request (CSR) against the control plane CA. After the node’s credentials have been signed, these serve as the authentication mechanism at runtime**.

This sounds complex, but, again, kubeadm makes this process extraordinarily simple:

```
$ kubeadm join --token <token> --discovery-token-ca-cert-hash <hash> \
    <api endpoint>
```

Although not as simple as the control plane case, it is pretty straightforward nonetheless. And, **in the case where you use kubeadm manually, the output from the `kubeadm init` command even provides the precise command that needs to be run on a worker node**.

Obviously, **if we are asking a worker node to join itself to the Kubernetes cluster, we need to tell it where to register itself. That is where the `<api endpoint>` parameter comes in. This includes the IP (or domain name) and port of the API server**.

Since this mechanism allows for a node to initiate the join, we want to ensure that this action is secure. For obvious reasons, **we do not want just any node to be able to join the cluster, and similarly, we want the worker node to be able to verify the authenticity of the control plane. This is where the `--token` and `--discovery-token-ca-cert-hash` parameters come into play**.

The `--token` parameter is a bootstrap token that has been predefined with the control plane. In our simple use case, **a bootstrap token has been automatically allocated by way of the `kubeadm init` invocation**. Users may also create these bootstrap tokens on the fly:

```
$ kubeadm token create [--ttl  <duration>]
```

This mechanism is especially handy when adding new worker nodes to the cluster. In this case, **the steps are simply to use `kubeadm token create` to define a new bootstrap token and then use that token in a `kubeadm join` command on the new worker node**.

**The `--discovery-token-ca-cert-hash` provides the worker node a mechanism to validate the CA of the control plane**. By presharing the SHA256 hash of the CA, the worker node may validate that the credentials it received, were, in fact, from the intended control plane.

The whole command may look something like this:

```
$ kubeadm join --token 878b76.ddab3219269370b2 10.1.2.15:6443 \
    --discovery-token-ca-cert-hash \
    sha256:312ce807a9e98d544f5a53b36ae3bb95cdcbe50cf8d1294d22ab5521ddb54d68
```

### Add-Ons

After you install the control plane and bring up a few worker nodes, the obvious next step is to get some workloads deployed. Before we can do so, we need to deploy a few add-ons.

Minimally, **we need to install a Container Network Interface (CNI) plug-in**. **This plug-in provides Pod-to-Pod (also known as *east-west*) network connectivity**. There are a multitude of options out there, each with their own specific life cycles, so kubeadm stays out of the business of trying to manage them. In the simplest of cases, **this is a matter of applying the `DaemonSet` manifest outlined by your CNI provider**.

Additional add-ons that you might want in your production clusters would probably include **log aggregation, monitoring, and maybe even service mesh capabilities**. Again, since these can be complex, kubeadm does not attempt to manage them.

The one special add-on that kubeadm manages is that of cluster DNS.** kubeadm currently supports `kube-dns` and `CoreDNS`, with `kube-dns` being the default**. As with all parts of kubeadm, **you may even choose to forego these standard options and install the cluster DNS provider of your choosing**.

### Phases

kubeadm serves as the basis for a variety of other Kubernetes installation tools. As you might imagine, if we are trying to make use of kubeadm in this way, **we may want some parts of the installation to be managed by kubeadm and others to be handled by the wrapping installer framework. kubeadm supports this use case, as well, with a feature called phases**.

**With phases, a user may leverage kubeadm to perform discrete actions undertaken in the installation process**. For instance, maybe the wrapping tool would like to use kubeadm for its ability to generate PKI assets. Or perhaps that tool wants to leverage kubeadm’s preflight checks in order to ensure that a cluster has best practices in place. All of this—and more—is available with kubeadm phases.

### High Availability

If you have been paying close attention, you probably noticed that we haven’t spoken about a highly available control plane. That is somewhat intentional.

As the purview of kubeadm is primarily from the perspective of a single node at a time, evolving kubeadm into a general-use tool for managing highly available installs would be relatively complicated. Doing so would start to blur the lines of the Unix philosophy of “doing one thing, and doing it well.”

That said, **kubeadm can be (and is) used to provide the components necessary for a highly available control plane**. Although there are a number of precise (and sometimes nuanced) actions that a user needs to take in order to create a highly available control plane, the basic steps are:

1. Create a highly available etcd cluster.

2. Initialize a primary control plane node with `kubeadm init` and a configuration that makes use of the etcd cluster created in step 1.

3. Transfer the PKI assets securely to all of the other control plane nodes.

4. Front the control plane API servers with a load balancer.

5. Join all workers nodes to the cluster by way of the load-balanced endpoints.

> If this is your use case, and you would like to use kubeadm to install a production-grade, highly available cluster, be sure to consult kubeadm high availability documentation. This documentation is maintained with each release of Kubernetes.

### Upgrades

As with any deployment, there will come a time when you want to take advantage of all the new features that Kubernetes has to offer. Similarly, **if you require a critical security update, you want the ability to enable it with as little disruption as possible**. Fortunately, **Kubernetes provides for zero-downtime upgrades**. Your applications continue to run while the underlying infrastructure is modified.

Although there are countless ways to upgrade a cluster, we focus on the kubeadm use case—a feature that has been available since version 1.8.

There are a lot of moving parts in any Kubernetes cluster, and this can make orchestrating the upgrade complicated. **kubeadm simplifies this significantly, as it is able to track well-tested version combinations for the kubelet, etcd, and the container images that serve the Kubernetes control plane**.

The order of operations when performing an upgrade is straightforward. First, **we plan our upgrade, and then we apply our determined plan**.

During the plan phase, **kubeadm analyzes the running cluster and determines the possible upgrade paths**. In the simplest case, we upgrade to a minor or patch release (e.g., from 1.10.3 to 1.10.4).

Slightly **more complicated is the upgrade to a whole new minor release that is two (or more) releases forward (e.g., 1.8 to 1.10). In this case, we need to walk the upgrades through each successive minor version until we reach our desired state**.

**kubeadm performs a number of preflight checks to ensure that the cluster is healthy and then examines the `kubeadm-config` ConfigMap in the `kube-system` namespace**. This ConfigMap helps kubeadm determine the available upgrade paths and ensures that any custom configuration items are carried forward.

Although much of the heavy lifting happens automatically, you may recall that **the kubelet (and kubeadm itself) is not managed by kubeadm**. When performing the plan, kubeadm indicates which unmanaged components also need to be upgraded:

```
root@control1:~# kubeadm upgrade plan
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with
'kubectl -n kube-system get cm kubeadm-config -oyaml'
[upgrade/plan] computing upgrade possibilities
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.9.5
[upgrade/versions] kubeadm version: v1.10.4
[upgrade/versions] Latest stable version: v1.10.4
[upgrade/versions] Latest version in the v1.9 series: v1.9.8

Components that must be upgraded manually after you have upgraded
the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT      AVAILABLE
Kubelet     4 x v1.9.3   v1.9.8

Upgrade to the latest version in the v1.9 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.9.5    v1.9.8
Controller Manager   v1.9.5    v1.9.8
Scheduler            v1.9.5    v1.9.8
Kube Proxy           v1.9.5    v1.9.8
Kube DNS             1.14.8    1.14.8

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.9.8

_____________________________________________________________________

Components that must be upgraded manually after you have upgraded
the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT      AVAILABLE
Kubelet     4 x v1.9.3   v1.10.4

Upgrade to the latest stable version:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.9.5    v1.10.4
Controller Manager   v1.9.5    v1.10.4
Scheduler            v1.9.5    v1.10.4
Kube Proxy           v1.9.5    v1.10.4
Kube DNS             1.14.8    1.14.8

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.10.4

```

**You should upgrade system components consistent with the manner in which they were installed** (typically with the OS package manager).

**After you have determined your planned upgrade approach, begin to execute the upgrades in the order specified by kubeadm**. If there are multiple releases in the upgrade path, perform those on each node, as indicated.

```
root@control1:~# kubeadm upgrade apply v1.10.4
```

**Again, preflight checks are performed, primarily to ensure that the cluster is still healthy, backups of the various static Pod manifests are made, and the upgrade takes place**.

In terms of node order, **ensure that you upgrade the control plane nodes first and then perform the upgrades on each worker node**.

**Control plane nodes should be unregistered as upstreams for any front-facing load balancers, upgraded, and then, after the entire control plane has been successfully upgraded, all control plane nodes should be re-registered as upstreams with the load balancer**.

**This may introduce a short-lived period of time when the API is unavailable**, but it ensures that all clients have a consistent experience.

If you are performing upgrades for each worker in place, **the workers may be upgraded in parallel. Note that this may result in a period of time when there are no kubelets available for scheduling Pods**.

Alternatively, **you may upgrade workers in a rolling fashion. This ensures that there is always a node that may be deployed to**.

> If your upgrade also involves simultaneously performing disruptive upgrades on the worker nodes (e.g., upgrading the kernel), it is advisable to use the `kubectl cordon` and/or `kubectl drain` semantics to ensure that your user workloads are rescheduled prior to the maintenance.


