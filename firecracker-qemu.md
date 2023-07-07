# Why We Replaced Firecracker with QEMU

Firecracker, the microVM hypervisor, is renowned for being lightweight, fast, and secure. It's excellent for running short-lived workflows, which is why it's the backbone of AWS Lambda. Our initial prototype for Hocus, a self-hosted alternative to Gitpod and GitHub Codespaces, utilized Firecracker. However, after weeks of testing, we decided to entirely replace it with QEMU. What's less known about Firecracker is its lack of support for many modern hypervisor features, such as dynamic RAM management, which is vital for long-lived workflows. In this post, I will explain why Firecracker might not be the best hypervisor choice and when you should consider avoiding it.

## Firecracker Optimizes for Short-Lived Workloads

The creators of Firecracker [state that](https://github.com/firecracker-microvm/firecracker/blob/dbd9a84b11a63b5e5bf201e244fe83f0bc76792a/README.md?plain=1#L24):

> Firecracker has a minimalist design. It excludes unnecessary devices and guest-facing functionality to reduce the memory footprint and attack surface area of each microVM.

The term "unnecessary" is intriguing - if this functionality is unnecessary, why was it incorporated into other hypervisors? It turns out that the definition of unnecessary must be narrowly understood - these excluded features are unnecessary for AWS Lambda, which spins up VMs to run short function calls and then shuts them down. If you're running a different kind of workload, like a VM that contains your development environment or a self-hosted GitHub Actions agent, these features cease to be unnecessary. Your VM will run for hours, days, or even months without stopping, unlike Firecracker, which is optimized for seconds or minutes.

## Firecracker is Not So Lightweight

Here are the two most significant features Firecracker lacks:

- Dynamic memory management - Firecracker's RAM footprint starts low, but once a workload inside allocates RAM, Firecracker will never return it to the host system. After running several workloads inside, you end up with an idling VM that consumes 32 GB of RAM on the host, even though it doesn't need any of it.
- Discard operations on storage - if you create a 10 GB file inside a VM and then delete it, the backing space won't be reclaimed on the host. The VM will occupy that disk space until you delete the entire VM drive.

These deficiencies make Firecracker a memory and drive-space hog.

## What Workloads Firecracker is Not the Right Choice For

Long-running ones. Our specific use case is disposable development environments - you click a button in the browser, and Hocus spins up a VM with your code, then gives you SSH access. Initially, Firecracker is indeed lightweight. It will consume ~20MB of RAM on your host, compared to ~150MB for QEMU. But run some workloads in the VM, create a few GBs of temporary files, and memory usage will surge to a couple of GBs. Once everything finishes running, a properly configured QEMU VM can return these resources to the host, but a Firecracker microVM will hold onto them until you shut it down and delete the backing drive. This is particularly noticeable if you're running multiple VMs on your host system. Firecracker will significantly limit the maximum number of concurrent long-running VMs because they will deplete your host's memory.

## QEMU is Not Perfect Though

The main issue with QEMU is that it has too many options you need to configure. For instance, enabling your VM to return unused RAM to the host requires at least three challenging tasks:

- Knowing that the feature even exists (it's called [free page reporting](https://docs.kernel.org/mm/free_page_reporting.html) and you have to specifically enable it in QEMU)
- Understanding that an obscure feature of Linux called [DAMON](https://www.kernel.org/doc/html/v5.17/vm/damon/index.html) exists, knowing what it's for, knowing how to configure it, and compiling a guest Linux kernel that supports it
- Knowing that you need to disable transparent huge pages on the guest, otherwise the VM will never return large amounts of memory

It took us two months of experimentation, reading through the source code of Firecracker, QEMU, and other hypervisors to develop a reliable QEMU proof of concept. To comprehend DAMON configuration, my co-founder spent days [running benchmarks and conversing with its author](https://lore.kernel.org/damon/20230504171749.89225-1-sj@kernel.org/T/).

## In Praise of Firecracker

For its designed purpose, Firecracker works excellently. It provides a secure runtime out of the box, which you can confidently use to run untrusted workloads. Its creators took great care to package it in a way that's easy to use. Not only is it written in a memory-safe language, but it also correctly minimizes its privileges on the host system. Even if an untrusted workload manages to escape the hypervisor, it finds itself with no privileges on the host. Technically, nothing prevents you from minimizing the privileges in the same way on QEMU, but you need to know how to do it step by step. Unless you're a security expert, you won't.

## Conclusion

QEMU has the features you need to run general-purpose workloads, but configuring it requires a lot of time and patience. If you want to run short-lived workloads, Firecracker is a great choice. However, if you just want to run your development environment in a VM, you can use [Hocus](https://github.com/hocus-dev/hocus). We've done all the hard work for you already. Except that it's still in alpha and missing some features, but don't let that discourage you from checking it out on [GitHub](https://github.com/hocus-dev/hocus).
