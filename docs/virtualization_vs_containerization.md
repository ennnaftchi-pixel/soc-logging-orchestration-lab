# Research: Virtualization vs. Containerization

This document outlines the architectural differences, performance considerations, and infrastructure choices evaluated while designing the SOC multi-service lab.

## 1. Type-1/Type-2 Hypervisors (Virtualization)
In our architecture, target endpoints like **Active Directory (Windows Server)** and **Linux Servers** are run within virtual machines (VMs). 
* **Isolation:** Virtualization provides strong hardware-level isolation via a Hypervisor (e.g., Proxmox, KVM/QEMU, or VMware). Each VM runs its own full guest operating system kernel.
* **Use Case:** Ideal for simulating realistic corporate endpoints where full OS lifecycle, distinct kernels, or deep kernel-level changes (like Windows Active Directory environments) are necessary.

## 2. OS-Level Virtualization (Containerization)
Central application services like **Wazuh, MISP, Cortex, and TheHive** are deployed via **Docker containers**.
* **Efficiency:** Containers share the host system's Linux kernel, avoiding the heavy memory and CPU overhead of running multiple guest operating systems.
* **Orchestration:** Multi-service stacks are defined declaratively in standard configuration files (`docker-compose.yml`), enabling fast environment reproducibility, microservices-based networking, and rapid deployment.

## ⚖️ Comparative Matrix

| Feature | Virtual Machines (VMs) | Containers (Docker) |
| :--- | :--- | :--- |
| **Isolation** | Hardware-level (Strong) | OS-level / Namespaces (Lighter) |
| **Kernel** | Guest OS has its own kernel | Shares the host OS kernel |
| **Boot Time** | Minutes | Seconds |
| **Resource Overhead**| High (Pre-allocated RAM/Disk) | Low (Dynamic scaling) |
| **Our Lab Use Case** | Windows AD & Linux Target Hosts | Logging, SIEM, and Orchestration Tools |
