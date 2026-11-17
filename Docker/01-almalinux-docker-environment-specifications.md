# AlmaLinux Docker Learning Environment

## Overview
This document defines the recommended system resources and software environment for learning Docker on AlmaLinux.

## Host Hardware Requirements

| Resource | Available |
| --- | --- |
| System memory (RAM) | 12 GB |
| Available storage | 50 GB |

## AlmaLinux Virtual Machine Specifications

Use the following allocation if AlmaLinux is running in VMware, VirtualBox, Hyper-V, or another virtualization platform.

| Component | Recommended Allocation |
| --- | --- |
| Operating system | AlmaLinux |
| RAM | 4 GB (4096 MB) |
| Virtual CPUs | 2 |
| Virtual disk | 30 GB |
| Network | NAT or bridged |
| Architecture | x86_64, if supported by the host |

### Memory Guidance
Allocate 4 GB of the host's 12 GB RAM to the AlmaLinux VM. This leaves approximately 8 GB for the host operating system and other applications. If the host becomes memory-constrained, reduce the VM allocation or close other applications.

### Storage Guidance
A 30 GB virtual disk is a practical starting point for the operating system, Docker images, containers, and practice projects. Keep an eye on image and container storage because it can grow over time. If the 50 GB storage limit refers to the total space available on the host, use a dynamically allocated virtual disk and ensure the host retains free space.

## Docker Resource Allocation

Docker Engine runs directly on the AlmaLinux operating system. It does not require a separate virtual machine or a fixed RAM allocation of its own. Containers use host resources as needed. Resource limits can be applied to individual containers when required.

| Resource | Configuration |
| --- | --- |
| Docker installation | Docker Engine on AlmaLinux |
| Dedicated Docker RAM | None |
| Container memory | Shared dynamically from the AlmaLinux VM's available memory |
| Container CPU | Shared dynamically from the AlmaLinux VM's available CPU capacity |
| Container storage | Uses the VM's available disk space |

## Recommended Baseline

For a 12 GB RAM host with approximately 50 GB of available storage, start with:
- AlmaLinux VM RAM: 4 GB
- Virtual CPUs: 2
- Virtual disk: 30 GB, preferably dynamically allocated
- Docker: Docker Engine installed directly inside AlmaLinux

This baseline provides a practical environment for learning core Docker concepts without dedicating excessive host resources.

## Installation Reference

For official Docker Engine installation instructions on AlmaLinux (RHEL-based), refer to the Docker documentation:
https://docs.docker.com/engine/install/rhel/
