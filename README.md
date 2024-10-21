# airgapped-kubespray

This repository provides documentation and scripts for installing production-ready Kubernetes cluster using kubespray in air-gapped (offline) environments.

# What is Kubespray

Kubernetes clusters can be created using various automation tools. Kubespray is a composition of Ansible playbooks, inventory, provisioning tools, and domain knowledge for generic OS/Kubernetes clusters configuration management tasks. That means we can install Kubernetes using Ansible.

What are the benefits of using kubespray?

- Kubespray provides deployment flexibility. It allows you to deploy a cluster quickly and customize all aspects of the implementation.
- Kubespray strikes a balance between flexibility and ease of use.
- You only need to run one Ansible playbook and your cluster ready to serve.

![Kubespray](https://bbs-img.huaweicloud.com/blogs/img/20230607/1686127981065342881.png?raw=true)

The above diagram shows the deployment architecture of kubespray.

You can visit Kubespray official website at: https://kubespray.io/

# Disclaimer

This document aims for creating a K8s cluster in such environment where your K8s nodes and even your Ansible master machine is OFFLINE and RedHat based. For achieving this you need an ONLINE machine (your laptop, a VM or even a container!) beside your Ansible master and target machines for downloading resources needed by Kubespray. It also assumes that you have a local repository (ex. Nexus and Nginx) for storing images, linux packages like RPM, PyPi files and raw files like tar and binary files.

# Requirements

The Requirements for this tutorial is:

- A VM for Nginx and Nexus to act as a repository that all kubernetes nodes have access to it.
- A VM with internet access for downloading rpms, images, raw files and pypi packages.
- A VM for Ansible server that has access to all kubernetes nodes and repository (this can be a separate VM or even the repository VM itself).
- At least three VMs for Kubernetes nodes (two for HA masters and one for worker).
- And one free IP (in your nodes network subnet) for APIserver loadbalancer.
