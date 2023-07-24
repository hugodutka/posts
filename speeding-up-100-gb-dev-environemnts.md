# Speeding Up 100+ GB Dev Environments

When you need a fresh compile of a project at a large company, you might as well start GCC and go home, since you have so much code that it's going to take hours.

We've found that this is the daily reality for many engineers, and we realized we could do something about it.

We figured out how to cut this time down from hours to seconds for any codebase. In this post, I'm going to explain the method behind it and the technical challenges we tackled to make it work.

## Core Idea

The main insight is that you can prepare the dev environment ahead of time. You can do it in CI. The way you automatically run tests, you can create an image of your dev environment that anyone can later download and use.[^prebuild]

[^prebuild]: This is not a new idea. Gitpod does it, GitHub Codespaces too. They call this process a prebuild. What we've done differently is optimize it for large dev envs and designed our software for self-hosting, so you can run it on your own infrastructure behind a VPN.

To leverage that insight, we need a way to define a dev environment, and a way to run it. To prepare it ahead of time, you start it, run all your setup tasks inside, and export its disk. You later use a copy of this disk to get a fresh, ready-to-code dev environment.

Defining a dev environment boils down to listing system dependencies and tasks to run during a prebuild. Runtime is where it gets tricky.[^standard]

[^standard]: We've decided to use a Dockerfile for system dependencies and a yaml file for defining prebuild tasks. There are other standards, such as devcontainer.json. We didn’t choose them because we haven't found one that doesn't assume that the development environment is virtualized with a container. We wanted to use virtual machines for reasons outlined in [this section](#containers-and-vms) of the post.

Our first idea was to use Docker. But when we looked closer, we started seeing problems. The first one was Docker's filesystem.

## Overlayfs is a Nuisance

Docker uses overlayfs, and it's incompatible with a lot of software. In fact, it's incompatible with Docker itself! It's one of the reasons why when you run Docker in Docker, instead of starting a separate Docker daemon, you usually mount the host's Docker socket inside the container.

We wanted a storage layer that doesn't limit the developer. We wanted the entire dev env to be backed by an ext4 filesystem, so that software could just work inside. We also wanted storage with support for layers, so when you have multiple dev environments running on the same computer, they could all share the same base without duplicating files on disk.

We were aware of one storage format that supported arbitrary filesystems and layered storage - QCOW2. QCOW2 was created for use in QEMU, the virtual machine monitor (VMM). We weren't thrilled with it though - we didn't find a performant way to use QCOW2 without using QEMU too, and we didn't want to be limited to QEMU as our runtime. QCOW2 performance also degrades significantly with a couple dozen layers.

While researching QCOW2 performance, my cofounder came across a [paper](https://www.usenix.org/conference/atc20/presentation/li-huiba) published by the Chinese big tech Alibaba about overlaybd, a novel storage format that they developed and implemented in house. It had everything we needed. It supports arbitrary filesystems, it is optimized for long layer chains, and exposes volumes as Linux block devices. We could plug it into any runtime we wanted, be it containers or even virtual machines.[^obd-open-source]

[^obd-open-source]: Overlaybd is [open-sourced](https://github.com/containerd/overlaybd) under containerd's organization on GitHub, and [integrated](https://github.com/containerd/accelerated-container-image) with the OCI ecosystem.

We found a solution. If we backed our dev environments with overlaybd, we could use ext4 and developers would not run into problems with overlayfs.

<h2 id="containers-and-vms">Docker is Not Powerful Enough</h3>

The second problem with Docker was the limitations that containers impose on security and what software they can virtualize. Most developers use Docker in their dev environment, and the default way to run Docker in Docker is with a privileged container. This gives the developer root access to the machine where their dev environment is hosted. It was not a viable route for us. We wanted to create a system that lets you run multiple isolated dev environments for different users on a single computer, so you could run dev environments on a powerful server securely shared by multiple developers.

To solve this, we could use a different container runtime. There is the Docker-compatible [Sysbox](https://github.com/nestybox/). If you step outside of the OCI ecosystem, there is [LXC](https://linuxcontainers.org/), which is more mature. These runtimes add more security layers on top of what Docker does. Most notably, they use [Linux user namespaces](https://man7.org/linux/man-pages/man7/user_namespaces.7.html) to give you root privileges inside the container, but none outside. This allows you to run Docker in a container without host root privileges, but still limits you in some ways. For example, Sysbox [does not support GPUs](https://github.com/nestybox/sysbox/issues/50), and blocks some syscalls. In many common development workflows these limitations are not noticeable. If you are developing a web app, you will not run into any issues. But if you're building on top of lower-level, uncommon Linux features, you may hit a wall.

There is also another option, virtual machines. They would impose minimal limits on what a developer could do, but they would use more compute resources than containers. They would also limit where dev environments could be hosted. For example, we wouldn't be able to run them in VM-based AWS EC2 instances, since they don't support nested virtualization.

There was no silver bullet that would solve all of our problems. The choice was between containers backed by Sysbox or LXC and virtual machines. We picked VMs for the proof of concept, but designed the system in a way that allows us to add container runtime support too. Hocus makes use of low-level Linux features that containers struggle with, so we used VMs in the first version so we could start developing Hocus with Hocus.

## Reimplementing Containerd

Once we figured out what technologies we wanted to use, we had to make them work together. We needed software that would run a QEMU VM using a disk managed by overlaybd.

We knew we could use [containerd](https://github.com/containerd/containerd). We could plug in [Kata Containers](https://katacontainers.io/) to run VMs instead of containers, and use [overlaybd-snapshotter](https://github.com/containerd/accelerated-container-image) so that containerd would leverage overlaybd for storage. Rather than integrate the components ourselves, we could use a ready-made solution. However, we decided against it and wrote a custom implementation instead. Our choice was dictated by the amount of tech risk we would take on and the need to remove abstraction.

Containerd is a mature technology, but overlaybd is not. Even though Alibaba is using it on thousands of servers, at the time of writing, overlaybd is at [version v0.6.12](https://github.com/containerd/overlaybd/releases/tag/v0.6.12). We expected to encounter issues, and we were not eager to find more in the snapshotter.

And even though containerd is mature, it consists of over 1M lines of code. By integrating directly with QEMU and overlaybd, we only need to worry about bugs in what we wrote ourselves, which is less than 5K LoC. We do not rely on an opaque abstraction layer between our system, runtime, and storage, so we can swap out any component in the future. This allows us to implement support for LXC later, which containerd does not offer.[^lxc]

[^lxc]: I would like to find out why Sysbox was created instead of building a containerd runtime that is a wrapper over LXC. If you know of any technical limitations that would prevent it, please let me know. It seems like a relatively low-effort project that would bring mature system containers into the OCI ecosystem.

## Summary

We now have a proof of concept - an alpha version of Hocus that can start a ready-to-code dev environment from a prebuild in under 5 seconds. What I covered in this post is just a part of what makes Hocus work. We also built a CI system from scratch[^temporal] to run prebuilds on new commits. We figured out how to start 100 GB+ dev environments in seconds even when you haven't downloaded them onto your host server yet.[^lazypulling]

[^temporal]: It's built on top of [Temporal](https://temporal.io/).

[^lazypulling]: If your dev environment's image size exceeds 100 GB, and you have 10 Gbps of network bandwidth, it's going to take at least 80 seconds to download. Overlaybd supports lazy pulling, which lets you download parts of the image on demand and start with only the data that you need to boot. You then pull the rest in the background while the developer is already inside the dev environment. However, there are still [some issues](https://github.com/containerd/overlaybd/issues/231) we need to iron out before we fully integrate it into Hocus.

Hocus is a work in progress, and we want to finish it in collaboration with people who need it. We are looking for individuals who can't stand their huge, slow dev environments at work and want to do something about it. If that's you, you can sign up for the closed beta of [Hocus Enterprise](https://hocus.dev/enterprise). We will work with you to introduce Hocus at your company, and adapt it to your needs. But, if you're just interested in what we've built so far, you can check out the [proof of concept on GitHub](https://github.com/hocus-dev/hocus).
