# Kubernetes Notes

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
