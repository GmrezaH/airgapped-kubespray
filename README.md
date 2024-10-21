# Air-gapped Kubespray

This repository provides documentation and scripts to help you install a production-ready Kubernetes cluster using Kubespray in air-gapped (offline) environments. It has been tested with the following versions:

- Kubernetes: v1.29.3
- Kubespray: v2.24.1

This setup is particularly useful in restricted environments where nodes, including the Ansible master, do not have internet access.

---

## What is Kubespray?

Kubernetes clusters can be created using various automation tools, and **Kubespray** is one of the most flexible options. Kubespray is a set of Ansible playbooks, along with inventory, provisioning tools, and domain knowledge that simplify the configuration and management of Kubernetes clusters. Kubespray allows you to use Ansible to set up and configure Kubernetes in a streamlined, reproducible way.

### Why Use Kubespray?

- **Deployment Flexibility**: Kubespray allows you to deploy Kubernetes clusters quickly with full customization of implementation.
- **Ease of Use**: Kubespray strikes the perfect balance between flexibility and simplicity, making it accessible while still offering extensive customization.
- **Single Playbook Execution**: You only need to run one Ansible playbook, and your cluster is ready to serve production workloads.

The diagram below illustrates the deployment architecture used by Kubespray:

<div align="center">
  <img src="https://bbs-img.huaweicloud.com/blogs/img/20230607/1686127981065342881.png?raw=true" alt="Kubespray Architecture" />
</div>

---

## Disclaimer

This repository is focused on creating Kubernetes clusters in air-gapped environments where both your Kubernetes nodes and your Ansible control machine have no direct internet access. The solution assumes you are working with Red Hat-based distributions, though it can be adapted to other Linux distributions with minor changes.

To achieve this setup, you'll need an internet-connected machine (e.g., your laptop, a VM, or even a container) that will download the necessary resources for Kubespray. This includes container images, RPMs, PyPi packages, and other required files. The downloaded files will be stored in a local repository (such as Nexus or Nginx), which will be accessible by your Kubernetes nodes.

> **Note:** While this documentation focuses on Red Hat-based distributions, adapting it for other distributions like Ubuntu is relatively straightforward. The primary differences lie in the package management tools (e.g., `yum/dnf` for Red Hat vs `apt` for Ubuntu), but the overall Kubespray and Kubernetes deployment process remains similar.

---

## Requirements

Before you begin, ensure that you have the following infrastructure set up:

1. **Repository VM**: A virtual machine running Nginx and Nexus to act as a local repository for Kubernetes nodes to access the necessary resources (container images, RPMs, PyPi packages, etc.).
2. **Download VM (Online Machine)**: A machine with internet access to download RPMs, container images, PyPi packages, and other raw files needed for the cluster setup. This can be your local machine or a dedicated VM.
3. **Ansible Control VM**: A virtual machine running Ansible that can communicate with all Kubernetes nodes and the local repository. This can be a dedicated VM or the same VM as the repository.
4. **Kubernetes Nodes**:
   - At least two VMs for Highly-Available (HA) Kubernetes masters.
   - At least one VM for a worker node.
5. **Load Balancer IP**: One free IP address in the same subnet as the Kubernetes nodes, dedicated for the Kubernetes API server load balancer.

---

## Steps to Deploy the Cluster

### 1. Online Machine Setup

On the Online VM (which has internet access), follow these steps to set up Docker, download the necessary files, and prepare them for the Repository VM:

```bash
# Install Docker and related packages
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# Download Docker packages for offline installation
yumdownloader --assumeyes --destdir=./docker --resolve docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Install Docker
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo groupadd docker
sudo usermod -aG docker $USER

# Log in to Docker and download Nexus and Nginx images
docker login
docker pull sonatype/nexus3
docker pull nginx

# Save the Docker images as tar.gz files
docker save sonatype/nexus3:latest | gzip > ./docker/nexus.tar.gz
docker save nginx:latest | gzip > ./docker/nginx.tar.gz

# Move the ./docker directory to the /opt/docker directory of the Repository VM.
```

### 2. Nexus Setup

On the Repository VM, install Docker and configure Nexus to act as a repository for Docker images, raw files, and PyPi packages.

```bash
# Install Docker on the Repository VM
sudo yum localinstall /opt/docker/*.rpm
sudo groupadd docker
sudo usermod -aG docker $USER

# Create Nexus Docker Compose file and run it
cat <<EOF > nexus-docker-compose.yml
version: "3"
services:
  nexus:
    image: sonatype/nexus3
    container_name: nexus
    restart: always
    volumes:
      - "nexus-data:/sonatype-work"
    ports:
      - "8081:8081"
      - "8082:8082"
volumes:
  nexus-data: {}
EOF

# Load the Nexus Docker image and start Nexus
docker load < /opt/docker/nexus.tar.gz
docker compose -f nexus-docker-compose.yml up -d

# Get the admin password for Nexus
sudo docker exec -it nexus cat /nexus-data/admin.password

# Access the Nexus web UI at http://<Repository_VM_IP>:8081
# Create a Docker hosted repository on port 8082 (named "image"),
# a raw hosted repo (named "raw"), and a PyPi hosted repo (named "pypi").
```

### 3. Nginx RPM Repository Setup

Nexus does not support modular RPMs (yet!), so we'll set up Nginx to serve RPM packages.

```bash
# Create the Nginx configuration file
cat <<EOF > /opt/docker/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    default_type application/octet-stream;
    include /etc/nginx/conf.d/*.conf;

    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;
        include /etc/nginx/default.d/*.conf;

        location / {
            root /usr/share/nginx/html/download;
            autoindex on;
            autoindex_exact_size off;
            autoindex_localtime on;
        }

        error_page 404 /404.html;
        location = /40x.html {}

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {}
    }
}
EOF

# Create Nginx Docker Compose file and run it
cat <<EOF > nginx-docker-compose.yml
version: "3"
services:
  nginx:
    image: nginx
    container_name: nginx
    restart: always
    volumes:
      - /opt/repos:/usr/share/nginx/html/download/rocky8
      - nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "8080:80"
EOF

# Load the Nginx Docker image and start the container
docker load < /opt/docker/nginx.tar.gz
docker compose -f nginx-docker-compose.yml up -d
```

### 4. Download Kubernetes Resources

On the Online VM, follow these steps to download all the necessary resources for Kubespray, including RPM packages, PyPi packages, and container images.

```bash
# Download the Kubespray project and its dependencies
cd /opt
git clone https://github.com/kubernetes-sigs/kubespray.git
VENVDIR=kubespray-venv
python3.11 -m venv $VENVDIR
source $VENVDIR/bin/activate

# Download PyPi packages required by Kubespray
pip download -r kubespray/requirements.txt -d /opt/pip-req
pip download twine -d /opt/pip-req

# Archive the downloaded PyPi packages
tar cvfz pypi.tar.gz ./pip-req
```

#### Generate List of Required Files and Container Images

Kubespray provides a script to generate lists of the required files and container images for offline installation.

```bash
# Generate the list of files and container images needed for the cluster
cd kubespray
./contrib/offline/generate_list.sh
# This will generate files.list and images.list in kubespray/contrib/offline/temp.
```

#### Download Files from `files.list`

Use the following script to download the required files as listed in the generated `files.list` file:

```bash
#!/bin/bash

CURRENT_DIR=$( dirname "$(readlink -f "$0")" )
OFFLINE_FILES_DIR_NAME="offline-files"
OFFLINE_FILES_DIR="${CURRENT_DIR}/${OFFLINE_FILES_DIR_NAME}"
OFFLINE_FILES_ARCHIVE="${CURRENT_DIR}/offline-files.tar.gz"
FILES_LIST=${FILES_LIST:-"${CURRENT_DIR}/kubespray/contrib/offline/temp/files.list"}

# Ensure the files list exists
if [ ! -f "${FILES_LIST}" ]; then
    echo "${FILES_LIST} should exist, run ./generate_list.sh first."
    exit 1
fi

# Clean up previous files and directories
rm -rf "${OFFLINE_FILES_DIR}"
rm -f "${OFFLINE_FILES_ARCHIVE}"
mkdir "${OFFLINE_FILES_DIR}"

# Download each file from the list
while read -r url; do
  if ! wget -x -P "${OFFLINE_FILES_DIR}" "${url}"; then
    exit 1
  fi
done < "${FILES_LIST}"

# Archive the downloaded files
tar -czvf "${OFFLINE_FILES_ARCHIVE}" "${OFFLINE_FILES_DIR_NAME}"
```

#### Download and Save Container Images

Next, use the following script to download and save the required container images:

```bash
#!/bin/bash

NEXUS_REPO="192.168.1.20:8082"  # Update this with your Nexus repository IP and port
IMAGES_ARCHIVE="${CURRENT_DIR}/container-images.tar.gz"
IMAGES_DIR="${CURRENT_DIR}/container-images"
IMAGES_LIST=${IMAGES_LIST:-"${CURRENT_DIR}/kubespray/contrib/offline/temp/images.list"}

# Ensure the images list exists
if [ ! -f "${IMAGES_LIST}" ]; then
    echo "${IMAGES_LIST} should exist, run ./generate_list.sh first."
    exit 1
fi

# Clean up previous images
rm -f "${IMAGES_ARCHIVE}"
rm -rf "${IMAGES_DIR}"
mkdir "${IMAGES_DIR}"

# Pull each image from the list
while read -r image; do
  if ! docker pull "${image}"; then
    exit 1
  fi
done < "${IMAGES_LIST}"

# Tag and save each image to a tar.gz file
IMAGES=$(docker images --format "{{.Repository}}:{{.Tag}}")
for i in $IMAGES; do
  NEW_IMAGE=${NEXUS_REPO}/${i}
  TAR_FILE=$(echo ${i} | sed 's/[\/:]/-/g')
  docker tag $i ${NEW_IMAGE}
  docker save ${NEW_IMAGE} | gzip > "${IMAGES_DIR}/${TAR_FILE}.tar.gz"
done

# Archive the saved images
tar cvfz "${IMAGES_ARCHIVE}" "${IMAGES_DIR}"
```

#### Download RPM Repositories

If RPM repositories are required (e.g., for Red Hat-based distributions), use the following script to sync them:

```bash
#!/bin/bash

yum install -y reposync epel-release

# Sync required RPM repositories
reposync --repoid=appstream --downloaddir=./repos --download-metadata --newest-only
reposync --repoid=baseos --downloaddir=./repos  --download-metadata  --newest-only
reposync --repoid=docker-ce-stable --downloaddir=./repos  --download-metadata  --newest-only
reposync --repoid=extras --downloaddir=./repos  --download-metadata  --newest-only
reposync --repoid=kubernetes-1.29-stable --downloaddir=./repos  --download-metadata
reposync --repoid=epel --downloaddir=./repos --download-metadata  --newest-only

# Archive the RPM repositories
tar cvfz ./rpm-repos.tar.gz ./repos
```

### 5. Upload Resources to Repository VM

After downloading all the necessary resources (RPMs, container images, and other files), transfer the files to the Repository VM and upload them to Nexus and Nginx.

#### Upload RPM Packages to Nginx

```bash
# Extract and upload RPM packages to Nginx
tar xvfz /opt/rpm-repos.tar.gz
rm -rf /etc/yum.repos.d/*
cat <<EOF > /etc/yum.repos.d/local.repo
[appstream]
name=Nginx Local RockyLinux 8.8 Appstream Repository
baseurl="http://<Repository_VM_IP>:8080/appstream/"
gpgcheck=0
enabled=1

[baseos]
name=Nginx Local RockyLinux 8.8 Baseos Repository
baseurl="http://<Repository_VM_IP>:8080/baseos/"
gpgcheck=0
enabled=1

[docker-ce-stable]
name=Nginx Local Docker-CE Repository
baseurl="http://<Repository_VM_IP>:8080/docker-ce-stable/"
gpgcheck=0
enabled=1

[epel]
name=Nginx Local EPEL Repository
baseurl="http://<Repository_VM_IP>:8080/epel/"
gpgcheck=0
enabled=1

[extras]
name=Nginx Local Extras Repository
baseurl="http://<Repository_VM_IP>:8080/extras/"
gpgcheck=0
enabled=1

[kubernetes-1.29-stable]
name=Nginx Local Kubernetes 1.29 Repository
baseurl="http://<Repository_VM_IP>:8080/kubernetes-1.29-stable/"
gpgcheck=0
enabled=1
EOF

yum clean all
```

#### Upload Docker Images to Nexus

```bash
# Extract and upload Docker images to Nexus
tar xvfz /opt/container-images.tar.gz
IMAGE_ARCHIVES=$(ls ./container-images)

# Load each Docker image from the archive
for i in ${IMAGE_ARCHIVES}; do docker load -i $i; done

# Push the images to Nexus
IMAGES=$(docker images --format "{{.Repository}}:{{.Tag}}" | grep 8082)
for image in $IMAGES; do docker push "$image"; done
```

#### Upload Raw Files to Nexus

```bash
# Extract and upload raw files to the raw repository in Nexus
tar xvfz /opt/offline-files.tar.gz
mv /opt/offline-files /opt/raw  # Ensure the folder name matches the raw repository name in Nexus

RAW_FILES=$(find /opt/raw/ -type f)
for file in $RAW_FILES; do
  curl -v --user '<Nexus_User>:<Nexus_Password>' --upload-file "$file" http://<Nexus_IP>:8081/repository/raw/
done
```

#### Upload PyPi Packages to Nexus

```bash
# Extract PyPi packages and upload them to Nexus
tar xvfz /opt/pypi.tar.gz
yum install python3.11 python3.11-pip
VENVDIR=kubespray-venv
python3.11 -m venv $VENVDIR
source $VENVDIR/bin/activate
pip3.11 install --no-index --find-links /opt/pip-req/ twine

# Create a .pypirc file for Nexus configuration
cat <<EOF > $HOME/.pypirc
[distutils]
index-servers =
    nexus

[nexus]
repository: http://<Nexus_IP>:8081/repository/pypi/
username: admin
password: admin
EOF

# Upload each PyPi package to Nexus
for file in /opt/pip-req/*.whl; do
    twine upload --repository nexus "$file"
done
```

### 6. Kubernetes Nodes Setup

On each Kubernetes node, configure the local repository and prepare the system for Kubernetes installation.

#### Add Local RPM Repository

Replace existing repositories with the local Nexus and Nginx repositories:

```bash
# Remove existing repositories and add local ones
rm -rf /etc/yum.repos.d/*
cat <<EOF > /etc/yum.repos.d/local.repo
[appstream]
name=Nexus Local RockyLinux 8.8 Appstream Repository
baseurl="http://<Repository_VM_IP>:8080/appstream/"
gpgcheck=0
enabled=1

[baseos]
name=Nexus Local RockyLinux 8.8 Baseos Repository
baseurl="http://<Repository_VM_IP>:8080/baseos/"
gpgcheck=0
enabled=1

[docker-ce-stable]
name=Nexus Local RockyLinux 8.8 Docker-CE Repository
baseurl="http://<Repository_VM_IP>:8080/docker-ce-stable/"
gpgcheck=0
enabled=1

[epel]
name=Nexus Local RockyLinux 8.8 EPEL Repository
baseurl="http://<Repository_VM_IP>:8080/epel/"
gpgcheck=0
enabled=1

[extras]
name=Nexus Local RockyLinux 8.8 Extras Repository
baseurl="http://<Repository_VM_IP>:8080/extras/"
gpgcheck=0
enabled=1

[kubernetes-1.29-stable]
name=Nexus Local Kubernetes 1.29 Repository
baseurl="http://<Repository_VM_IP>:8080/kubernetes-1.29-stable/"
gpgcheck=0
enabled=1
EOF

# Clean YUM cache
yum clean all
```

#### Disable Firewall and SELinux

To ensure smooth operation, disable the firewall and SELinux on the Kubernetes nodes:

```bash
# Disable firewall
systemctl stop firewalld
systemctl disable firewalld

# Disable SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# Reboot the node
reboot
```

#### Update DNS Settings

Add a DNS record if necessary:

```bash
# Add DNS server
echo "nameserver 192.168.0.1" >> /etc/resolv.conf
```

### 7. Ansible Control Machine Setup

On the Ansible control machine, install the necessary packages and configure the environment for running Kubespray.

#### Install Required Packages

```bash
# Remove existing repositories and add local ones
rm -rf /etc/yum.repos.d/*
cat <<EOF > /etc/yum.repos.d/local.repo
[appstream]
name=Nexus Local RockyLinux 8.8 Appstream Repository
baseurl="http://<Repository_VM_IP>:8080/appstream/"
gpgcheck=0
enabled=1

[baseos]
name=Nexus Local RockyLinux 8.8 Baseos Repository
baseurl="http://<Repository_VM_IP>:8080/baseos/"
gpgcheck=0
enabled=1

[docker-ce-stable]
name=Nexus Local RockyLinux 8.8 Docker-CE Repository
baseurl="http://<Repository_VM_IP>:8080/docker-ce-stable/"
gpgcheck=0
enabled=1

[epel]
name=Nexus Local RockyLinux 8.8 EPEL Repository
baseurl="http://<Repository_VM_IP>:8080/epel/"
gpgcheck=0
enabled=1

[extras]
name=Nexus Local RockyLinux 8.8 Extras Repository
baseurl="http://<Repository_VM_IP>:8080/extras/"
gpgcheck=0
enabled=1

[kubernetes-1.29-stable]
name=Nexus Local Kubernetes 1.29 Repository
baseurl="http://<Repository_VM_IP>:8080/kubernetes-1.29-stable/"
gpgcheck=0
enabled=1
EOF

# Clean YUM cache and install required packages
yum clean all
dnf install -y git python3.11 python3.11-pip python3.11-netaddr yum-utils container-selinux rsync unzip bash-completion ipvsadm socat
```

#### Set Up Kubespray Environment

```bash
# Change directory to where Kubespray is located
cd /opt/ansible

# Set up a Python virtual environment for Kubespray
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
python3.11 -m venv $VENVDIR
source $VENVDIR/bin/activate

# Configure pip to use the local Nexus PyPi repository
cat <<EOF > /etc/pip.conf
[global]
index-url = http://<Repository_VM_IP>:8081/repository/pypi/simple
trusted-host = <Repository_VM_IP>
EOF

# Install Kubespray dependencies
cd $KUBESPRAYDIR
pip3.11 install -r requirements.txt --trusted-host <Repository_VM_IP>

# Set up Docker to use the local Nexus image registry
vim /etc/docker/daemon.json   # Add the following line: { "insecure-registries": ["<Repository_VM_IP>:8082"] }
docker login <Repository_VM_IP>:8082
```

#### Configure Kubernetes Inventory

```bash
# Copy the sample inventory to a new cluster configuration
cp -rfp inventory/sample inventory/mycluster

# Define the IP addresses of the Kubernetes nodes
declare -a IPS=(<K8s_node1_IP> <K8s_node2_IP> <K8s_node3_IP>)

# Generate the inventory file based on the node IPs
CONFIG_FILE=inventory/mycluster/hosts.yml KUBE_CONTROL_HOSTS=2 python3.11 contrib/inventory_builder/inventory.py ${IPS[@]}

# Review and modify the group variables as necessary
vim inventory/mycluster/group_vars/all/all.yml
vim inventory/mycluster/group_vars/all/containerd.yml
vim inventory/mycluster/group_vars/all/offline.yml
vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
vim inventory/mycluster/group_vars/k8s_cluster/addons.yml
```

### 8. Deploy the Kubernetes Cluster

Once everything is set up, run the Ansible playbook to deploy the Kubernetes cluster.

```bash
# Reset any existing cluster on the nodes (if applicable)
ansible-playbook -i inventory/mycluster/hosts.yml -b --become-user root reset.yml

# Run the main playbook to create your Kubernetes cluster
ansible-playbook -i inventory/mycluster/hosts.yml -b --become-user root cluster.yml
```

If you're deploying a high-availability cluster with kube-vip, run the cluster playbook without kube-vip first, then install kube-vip afterward.

### 9. Post-Deployment Tasks

Once the playbook finishes successfully, the cluster should be up and running. To prevent Kubernetes pods from scheduling on the master nodes, you can apply a "NoSchedule" taint:

```bash
kubectl taint nodes <control-plane_node_name> node-role.kubernetes.io/control-plane:NoSchedule
```

---

## Contributions and Support

Feel free to open issues or contribute to this repository. For further questions or support, refer to the [Kubespray documentation](https://kubespray.io) or contact the maintainers of this project.
