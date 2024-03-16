# An Overview of Kubernetes

> Reference Book:
> 
> Managing Kubernetes
> 
> By Brendan Burns and Craig Tracey

---

## Containers

Because Kubernetes is a *container orchestrator* to understand Kubernetes, it’s important to understand what we mean when we say *container*.

In reality, a container is made up of two different pieces, and a group of associated features. A container includes:

- A container image

- A set of operating system concepts that isolates a running process or processes

The *container image* contains the application runtime, which consists of binaries, libraries, and other data needed to run the container.

Developer can package up their application as a container image on their development laptop and have faith that when that image is deployed and run in a different setting—be it another user’s laptop or a server in a datacenter—the container will behave exactly as it did on the developer’s laptop.

This portability and consistent execution in a variety of environments are among the primary values of container images.

When a container image is run, it is also executed using namespaces in the operating system. These namespaces contain the process and provide isolation for it and its peers from other things running on the machine.

This isolation means, for example, that each running container has its own separated filesystem (like a `chroot`). Additionally, each container has its own network and PID namespaces, meaning that process number 42 in one container is a different process than number 42 in another container. There are many other namespaces within the kernel that separate various running containers from each other.

Additionally, control groups (cgroups) allow the isolation of resource usage, like memory or CPU.

Finally, standard operating system security features, like SELinux or AppArmor, can also be used with running containers.

Combined, all of this isolation makes it more difficult for different processes running in separate containers to interfere with each other.

> When we say *isolation*, it is incredibly important to know that this is in terms of resources, like CPU, memory, or files. Containers as implemented in Linux and Windows do *not* currently provide strong security isolation for different processes. Containers when combined with other kernel-level isolation can provide reasonable security isolation for some use cases. However, in the general case, only hypervisor-level security is strong enough to isolate truly hostile workloads.

In order to make all of this work, a number of different tools were created to help build and deploy containerized applications.

The first is the container image builder. Typically the `docker` command-line tool is used to build a container image. However, the image format has been standardized through the Open Container Initiative (OCI) standard. This has enabled the development of other image builders, available via cloud API, CI/CD, or new alternative tools and libraries.

The `docker` tool uses a *Dockerfile*, which specifies a set of instructions for how to construct the container image.

After a container image has been built, we need a way to distribute that image from a user’s laptop up to other users, the cloud, or a private datacenter. This is where the *image registry* comes in.

The image registry is an API for uploading and managing images. After an image has been built, it is pushed to the image registry. After the image is in the registry, it can be pulled, or downloaded, from that registry to any machine that has access to the registry.

Every registry requires some form of authorization to push an image, but some registries are *public*, meaning that once an image is pushed, anyone in the world can pull and start running the image. Others are *private* and require authorization to pull an image, as well.

At this point, there are registries as a service available from every public cloud, and there are open source registry servers, which you can download and run in your own environment.

---

## Container Orchestration

After you have a container image stored in a registry somewhere, you need to run it to create a working application. This is where a container orchestrator like Kubernetes comes into the picture. **Kubernetes’ job is to take a group of machines that provide resources, like CPU, memory, and disk, and transform them into a container-oriented API that developers can use to deploy their containers**.

The Kubernetes API enables you to declare your desired state of the world, for example, “I want this container image to run, and it needs 3 cores and 10 gigabytes of memory to run correctly.” The Kubernetes system then reviews its fleet of machines, finds a good place for that container image to run, and *schedules* the execution of that container on that machine. Developers see their container image running, and more often than not, they don’t need to concern themselves with the specific location where their container is executing.

Of course, running just a single container is neither that interesting nor that reliable, so the Kubernetes API also provides easy ways to say, “I want three copies of this container image running on different machines, each with 3 cores and 10 gigabytes of memory.”

But the orchestration system is about more than scheduling containers to machines. In addition to that, the **Kubernetes orchestrator knows how to heal those containers if they fail**. If the process inside your container crashes, Kubernetes restarts it. If you define custom health checks, Kubernetes can use them to determine whether your application is deadlocked and needs to be restarted (*liveness checks*) or if it should be part of a load-balanced service (*readiness checks*).

Speaking of load balancing, **Kubernetes also provides API objects for defining a way to load balance traffic between these various replicas**. It provides a way to say, “Please create this load balancer to represent these running containers.” These load balancers are also given easy-to-discover names so that linking different services together within a cluster is easy.

**Kubernetes also has objects that perform zero-downtime rollouts and that manage configurations, persistent volumes, secrets, and much more**.

### The Kubernetes API

**The Kubernetes API is a RESTful API based on HTTP and JSON and provided by an *API server***. All of the components in Kubernetes communicate through the API.

As an open source project, the Kubernetes API is always evolving, but the core objects have been stable for years and the Kubernetes community provides a strong deprecation policy that ensures that developers and operators don’t have to change what they are doing with each revision of the system.

Kubernetes provides an OpenAPI specification for the API, as well as numerous [client libraries in a variety of languages](https://github.com/kubernetes-client).

#### Basic Objects: Pods, ReplicaSets, and Services

##### Pods

A *Pod* is the atomic unit of scheduling in a Kubernetes cluster. **A Pod is made up of a collection of one or more running containers**. (A Pod is a collection of whales, derived from Docker’s whale logo.)

 When we say that a Pod is *atomic*, what we mean is that **all of the containers in a Pod are guaranteed to land on the same machine in the cluster**.

Pods also share many resources between the containers. For example, they **all share the same network namespace, which means that each container in a Pod can see the other containers in the Pod on `localhost`**. **Pods also share the process and interprocess communication namespaces so that different containers can use tools, like shared memory and signaling, to coordinate between the different processes in the Pod**.

This close grouping means that **Pods are ideally suited for symbiotic relationships between their containers, such as a main serving container and a background data-loading container**. Keeping the container images separate generally makes it more agile for different teams to own or reuse the container images, but grouping them together in a Pod at runtime enables them to operate cooperatively.

When people first encounter Pods in Kubernetes, they sometimes spring to the wrong assumptions. For example, a user may see a Pod and think, “Ah yes, a frontend and a database server make up a Pod.” But this is generally the wrong level of granularity. To see why, consider that the **Pod is also the unit of scaling and replication**, which means that, if you group your frontend and your database in the same container, you will replicate your database at the same rate that you replicate your frontends. It is unlikely that you want to do things this way.

P**ods also do things to keep your application running**. If the process in a container crashes, Kubernetes automatically restarts it. **Pods can also define application-level health checks that can provide a richer, application-specific way of determining whether the Pod should be automatically restarted**.

##### ReplicaSets

In general, **one of the main reasons for container orchestration is to make it easier to build replicated, reliable systems**.

Although individual containers may fail or may be incapable of serving the load of a system, **replicating an application out to a number of different running containers dramatically reduces the probability that your service will completely fail at a particular moment in time**.

Plus, horizontal scaling enables you to grow your application in response to load. In the Kubernetes API, **this sort of stateless replication is handled by a `ReplicaSet` object**.

A `ReplicaSet` ensures that, for a given Pod definition, a number of replicas exists within the system.

The actual replication is handled by the Kubernetes controller manager, which creates Pod objects that are scheduled by the Kubernetes scheduler.

> `ReplicaSet` is a newer object. At its v1 release, Kubernetes had an API object called a `ReplicationController`. Due to the deprecation policy, `ReplicationControllers` continue to exist in the Kubernetes API, but their usage is strongly discouraged in favor of `ReplicaSets`.

##### Services

After you can replicate your application out using a replica set, the next logical goal is to create a load balancer to spread traffic to these different replicas. To accomplish this, Kubernetes has a `Service` object.

**A `Service` represents a TCP or UDP load-balanced service**. Every `Service` that is created, whether TCP or UDP, gets three things:

- Its own IP address

- A DNS entry in the Kubernetes cluster DNS

- Load-balancing rules that proxy traffic to the Pods that implement the `Service`

When a `Service` is created, it is assigned a fixed IP address. This IP address is virtual—it does not correspond to any interface present on the network. Instead, it is programmed into the network fabric as a load-balanced IP address. When packets are sent to that IP, they are load balanced out to a set of Pods that implements the `Service`.

**The load balancing that is performed can either be round robin or deterministic, based on source and destination IP address tuples**.

Given this fixed IP address, **a DNS name is programmed into the Kubernetes cluster’s DNS server**. This DNS address provides a semantic name (e.g., “frontend”), which is the same as the name of the Kubernetes `Service` object and which enables other containers in the cluster to discover the IP address of the `Service` load balancer.

Finally, **the `Service` load balancing is programmed into the network fabric of the Kubernetes cluster so that any container that tries to talk to the `Service` IP address is correctly load balanced to the corresponding Pods**. **This programming of the network fabric is dynamic, so as Pods come and go due to failures or scaling of a `ReplicaSet`, the load balancer is constantly reprogrammed to match the current state of the cluster**. This means that clients can rely on connections to the `Service` IP address always resolving to a Pod that implements the `Service`.

##### Storage: Persistent Volumes, ConfigMaps, and Secrets

Kubernetes provides several different API objects to help you manage your files.

The first storage concept introduced in Kubernetes was Volume, which is actually a part of the Pod API. **Within a Pod, you can define a set of Volumes. Each Volume can be one of a large number of different types**. At present, there are more than 10 different types of Volumes you can create, including NFS, iSCSI, gitRepo, cloud storage–based Volumes, and more.

> Though the Volume interface was initially a point of extensibility via writing code within Kubernetes, the explosion of different Volume types eventually showed how unsustainable this model was. Now, new Volume types are developed outside of the Kubernetes code and use the Container Storage Interface (CSI), an interface for storage that is independent of Kubernetes.

**When you add a Volume to your Pod, you can choose to mount it to an arbitrary location in each running container**. This enables your running container to have access to the storage within the Volume. Different containers can mount these Volumes at different locations or can ignore the Volume entirely.

In addition to basic files, there are several types of Kubernetes objects that can themselves be mounted into your Pod as a Volume. The first of these is the `ConfigMap` object. **A `ConfigMap` represents a collection of configuration files**. In Kubernetes, you want to have different configurations for the same container image. When you add a `ConfigMap`-based Volume to your Pod, the files in the `ConfigMap` show up in the specified directory in your running container.

**Kubernetes uses the Secret configuration type for secure data, such as database passwords and certificates**. In the context of Volumes, a Secret works identically to a `ConfigMap`. It can be attached to a Pod via a Volume and mounted into a running container for use.

Over time, deploying applications with Volumes revealed that the tight binding of Volumes to Pods was actually problematic. For example, when creating a replicated container (via a `ReplicaSet`) the same exact volume must be used by all replicas. In many situations, this is acceptable, but in some cases, you migth want a different Volume for each replica. Additionally, specifying a precise volume type (e.g., an Azure disk-persistent Volume) binds your Pod definition to a specific environment (in this case, the Microsoft Azure cloud), but it is often desirable to have a Pod definition that requests a generic type of storage (e.g., 10 gigabytes of network storage) without specifying a provider. To accomplish this, **Kubernetes introduced the notion of `PersistentVolumes` and `PersistentVolumeClaims`**. **Instead of binding a Volume directly into a Pod, a `PersistentVolume` is created as a separate object. This object is then claimed to a specific Pod by a `PersistentVolumeClaim` and finally mounted into the Pod via this claim**. At first, this seems overly complicated, but the abstraction of Volume and Pod enables both the portability and automatic volume creation required by the two previous use cases.

#### Organizing Your Cluster with Namespaces, Labels, and Annotations

##### Namespaces

You can think of a **`Namespace` as something like a folder for your Kubernetes API objects. `Namespace`s provide directories for containing most of the other objects in the cluster**.

**`Namespace`s can also provide a scope for role-based access control (RBAC) rules. Like a folder, when you delete a `Namespace`, all of the objects within it are also destroyed**, so be careful!

**Every Kubernetes cluster has a single built-in `Namespace` named `default`, and most installations of Kubernetes also include a `Namespace` named `kube-system`, where cluster administration containers are created.**

> Kubernetes objects are divided into *namespaced* and *non-namespaced* objects, depending on whether they can be placed in a `Namespace`. Most common Kubernetes API objects are namespaced objects. But some objects that apply to an entire cluster (e.g., `Namespace` objects themselves, or cluster-level RBAC), are not namespaced.

In addition to organizing Kubernetes objects, **`Namespace`s are also placed into the DNS names created for `Service`s and the DNS search paths that are provided to containers**.

The complete DNS name for a `Service` is something like *my-service.svc.my-namespace.cluster.internal*, which means that two different `Service`s in different `Namespace`s will end up with different fully qualified domain names (FQDNs).

Additionally, the DNS search paths for each container include the `Namespace`, thus a DNS lookup for `frontend` will be translated to *frontend.svc.foo.cluster.internal* for a container in the `foo` `Namespace` and *frontend.svc.bar.cluster.internal* for a container in the `bar` `Namespace`.

##### Labels and label queries

Every object in the Kubernetes API can have an arbitrary set of *labels* associated with it.

**Labels are string key-value pairs that help identify the object**.

For example, a label might be `"role"`: `"frontend"`, which indicates that the object is a frontend.

**These labels can be used to query and filter objects in the API**. For example, you can request that the API server provide you with a list of all Pods where the label `role` is `backend`. These requests are called *label queries* or *label selectors*.

**Many objects within the Kubernetes API use label selectors as a way to identify sets of objects that they apply to**. For example, a Pod can have a *node selector*, which identifies the set of nodes on which the Pod is elegible to run (nodes with GPUs, for example). Likewise, a `Service` has a *Pod selector*, which identifies the set of Pods that the `Service` should load balance traffic to.

**Labels and label selectors are the fundamental manner in which Kubernetes loosely couples its objects together**.

##### Annotations

Not every metadata value that you want to assign to an API object is identifying information.

**Some of the information is simply an *annotation* about the object itself**.

**Thus every Kubernetes API object can also have arbitrary annotations**. These might include something like the icon to display next to the object or a modifier that changes the way that the object is interpreted by the system.

Often, experimental or vendor-specific features in Kubernetes are initially implemented using annotations, since they are not part of the formal API specification. In these cases, the annotation itself should carry some notion of the stability of the feature (e.g., `beta.kubernetes.io/activate-some-beta-feature`).

#### Advanced Concepts: Deployments, Ingress, and StatefulSets

##### Deployments

Although `ReplicaSets` are the primitive for running many different copies of the same container image, applications are not static entities. They evolve as developers add new features and fix bugs. This means that the act of rolling out new code to a `Service` is as important a feature as replicating it to reliably handle load.

**The `Deployment` object was added to the Kubernetes API to represent this sort of safe rollout from one version to another**.

**A `Deployment` can hold pointers to multiple `ReplicaSets`, (e.g., `v1` and `v2`), and it can control the slow and safe migration from one `ReplicaSet` to another**.

To understand how a `Deployment` works, imagine that you have an application that is deployed to three replicas in a `ReplicaSet` named `rs-v1`. When you ask a `Deployment` to roll out a new image (`v2`), the `Deployment` creates a new `ReplicaSet` (`rs-v2`) with a single replica. The `Deployment` waits for this replica to becomes healthy, and when it is, the `Deployment` reduces the number of replicas in `rs-v1` to two. It then increases the number of replicas in `rs-v2` to two also, and waits for the second replica of `v2` to become healthy. This process continues until there are no more replicas of `v1` and there are three healthy replicas of `v2`.

> Deployments feature a large number of different knobs that can be tuned to provide a safe rollout for the specific details of an application. Indeed, in most modern clusters, users exclusively use `Deployment` objects and don’t manage `ReplicaSets` directly.

##### HTTP load balancing with Ingress

Although `Service` objects provide a great way to do simple TCP-level load balancing, they don’t provide an application-level way to do load balancing and routing. The truth is that most of the applications that users deploy using containers and Kubernetes are HTTP web-based applications. These are better served by a load balancer that understands HTTP.

To address these needs, the `Ingress` API was added to Kubernetes. **`Ingress` represents a path and host-based HTTP load balancer and router**. When you create an `Ingress` object, it receives a virtual IP address just like a `Service`, but instead of the one-to-one relationship between a `Service` IP address and a set of Pods, an `Ingress` can use the content of an HTTP request to route requests to different `Service`s.

To get a clearer understanding of how `Ingress` works, imagine that we have two Kubernetes `Service`s named “foo” and “bar.” Each has its own IP address, but we really want to expose them to the internet as part of the same host. For example, *foo.company.com* and *bar.company.com*. We can do this by creating an `Ingress` object and associating its IP address with both the *foo.company.com* and *bar.company.com* DNS names. In the `Ingress` object, we also map the two different hostnames to the two different Kubernetes `Service`s. That way, when a request for *https:/​/foo.company.com* is received, it is routed to the “foo” `Service` in the cluster, and similarly for *https:/​/bar.company.com*. With `Ingress`, the routing can be based on either host or path or both, so *https:/​/company.com/bar* can also be routed to the “bar” `Service`.

> The `Ingress` API is one of the most decoupled and flexible APIs in Kubernetes. By default, although Kubernetes will store `Ingress` objects, nothing happens when they are created. Instead, you need to also run an *Ingress Controller* in the cluster to take appropriate action when the `Ingress` object is created. One of the most popular Ingress Controllers is `nginx`, but there are numerous implementations that use other HTTP load balancers or that use cloud or physical load-balancer APIs.

##### StatefulSets

Most applications operate correctly when replicated horizontally and treated as identical clones. Each replica has no unique identity independent of any other. For representing such applications, a Kubernetes `ReplicaSet` is the perfect object. However, some applications, especially stateful storage workloads or sharded applications, require more differentiation between the replicas in the application. Although it is possible to add this differentiation at the application level on top of a `ReplicaSet`, doing so is complicated, error prone, and repetitive for end users.

To resolve this, Kubernetes has recently introduced `StatefulSets` as a complement to `ReplicaSets`, but for more stateful workloads. **Like `ReplicaSets`, `StatefulSets` create multiple instances of the same container image running in a Kubernetes cluster, but the manner in which containers are created and destroyed is more deterministic, as are the names of each container**.

In a `ReplicaSet`, each replicated Pod receives a name that involves a random hash (e.g., *frontend-14a2*), and there is no notion of ordering in a `ReplicaSet`.

**In contrast, with `StatefulSets`, each replica receives a monotonically increasing index (e.g., *backed-0*, *backend-1*, and so on)**.

Further, **`StatefulSets` guarantee that replica zero will be created and become healthy before replica one is created and so forth**. When combined, this means that applications can easily bootstrap themselves using the initial replica (e.g., *backend-0*) as a bootstrap master. All subsequent replicas can rely on the fact that *backend-0* has to exist.

Likewise, **when replicas are removed from a `StatefulSet`, they are removed at the highest index**. If a `StatefulSet` is scaled down from five to four replicas, it is guaranteed that the fifth replica is the one that will be removed.

Additionally, **`StatefulSets` receive DNS names so that each replica can be accessed directly, in addition to the complete `StatefulSet`**. This allows clients to easily target specific shards in a sharded service.

#### Batch Workloads: Job and ScheduledJob

In addition to stateful workloads, another specialized class of workloads are *batch* or *one-time workloads*. In contrast to the previously discussed workloads, these are not constantly serving traffic. Instead, they perform some computation and are then destroyed when the computation is complete.

In Kubernetes, **a `Job` represents a set of tasks that needs to be run**.

Like `ReplicaSets` and `StatefulSets`, **`Job`s operate by creating Pods to execute work by running container images. However, unlike `ReplicaSets` and `StatefulSets`, the Pods created by a `Job` only run until they complete and exit**.

**A `Job` contains the definition of the Pods it creates, the number of times the `Job` should be run, and the maximum number of Pods to create in parallel**. For example, a `Job` with 100 repetitions and a maximum parallelism of 10 will run 10 Pods simultaneously, creating new Pods as old ones complete, until there have been 100 successful executions of the container image.

**`ScheduledJobs` build on top of the `Job` object by adding a schedule to a `Job`**. A `ScheduledJob` contains the definition of the `Job` object that you want to create, as well as the schedule on which that `Job` should be created.

#### Cluster Agents and Utilities: DaemonSets

One of the most common questions that comes up when people are moving to Kubernetes is, “How do I run my machine agents?” Examples of agents’ tasks include intrusion detection, logging and monitoring, and others. Many people attempt non-Kubernetes approaches to enable these agents, such as adding new systemd unit files or initialization scripts. Although these approaches can work, they have several downsides.

The first is that **Kubernetes does not include agents’ activity in its accounting of resources in use on the cluster**.

The second is that **container images and Kubernetes APIs for health checking, monitoring, and more cannot be applied to these agents**.

Fortunately, **Kubernetes makes the `DaemonSet` API available to users to install such agents on their clusters**.

**A `DaemonSet` provides a template for a Pod that should be run on every machine. When a `DaemonSet` is created, Kubernetes ensures that this Pod is running on each node in the cluster**.

If, at some later point, a new node is added, Kubernetes creates a Pod on that node, as well. **Although by default Kubernetes places a Pod on every node in the cluster, a `DaemonSet` can also provide a node selector label query, and Kubernetes will only place that `DaemonSet`’s Pods onto nodes that match that label query**.


