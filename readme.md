# Kubernetes Dynamic NFS Provisioning

This repository provides a step-by-step guide to set up dynamic NFS (Network File System) provisioning on a Kubernetes cluster. Dynamic provisioning allows Kubernetes to automatically create and manage Persistent Volumes (PVs) using NFS as the storage backend. This setup is useful for applications that require shared storage across multiple pods.

## Table of Contents
- [Kubernetes Dynamic NFS Provisioning](#kubernetes-dynamic-nfs-provisioning)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Optional: Setting Up Additional Block Storage](#optional-setting-up-additional-block-storage)
    - [1. Identify the Raw Block Storage](#1-identify-the-raw-block-storage)
    - [2. Partition the Block Storage](#2-partition-the-block-storage)
    - [3. Format the Partition](#3-format-the-partition)
    - [4. Create a Mount Point](#4-create-a-mount-point)
    - [5. Mount the Partition](#5-mount-the-partition)
    - [6. Make the Mount Permanent](#6-make-the-mount-permanent)
  - [NFS Server Setup](#nfs-server-setup)
  - [Kubernetes Cluster Setup](#kubernetes-cluster-setup)
    - [Install NFS Subdir External Provisioner](#install-nfs-subdir-external-provisioner)
    - [Verify Installation](#verify-installation)
  - [Usage](#usage)
  - [References](#references)

## Prerequisites

Before proceeding, ensure that you have the following:

1. A running Kubernetes cluster.
2. `kubectl` installed and configured to interact with your cluster.
3. `helm` (version 3) installed.
4. `nfs-common` installed on all Kubernetes nodes.
5. An NFS server with a shared directory.

---

## Optional: Setting Up Additional Block Storage

If you have additional block storage (e.g., `/dev/vdb` or `/dev/vdc`) with a larger size, you can use it for the NFS shared directory. Follow these steps to partition, format, and mount the block storage:

### 1. Identify the Raw Block Storage
List all block devices to identify the raw storage:
```bash
lsblk -l
```
Look for the device you want to use (e.g., `/dev/vdb`).

### 2. Partition the Block Storage
Use `fdisk` to create a partition:
```bash
sudo fdisk /dev/sdX  # Replace /dev/sdX with your device (e.g., /dev/vdb)
```
Within `fdisk`:
- Press `n` to create a new partition.
- Press `p` to create a primary partition.
- Follow the prompts to specify the partition number, first sector, and last sector (or size).
- Press `w` to write the changes to the disk.

### 3. Format the Partition
Format the partition with the `ext4` filesystem:
```bash
sudo mkfs.ext4 /dev/sdX1  # Replace /dev/sdX1 with your partition (e.g., /dev/vdb1)
```

### 4. Create a Mount Point
Create a directory to mount the partition:
```bash
sudo mkdir /nfs  # Or any directory you prefer
```

### 5. Mount the Partition
Mount the partition to the directory:
```bash
sudo mount /dev/sdX1 /nfs  # Replace /dev/sdX1 with your partition
```

### 6. Make the Mount Permanent
To ensure the partition is mounted automatically on boot, add an entry to the `/etc/fstab` file:
1. Get the UUID of the partition:
   ```bash
   sudo blkid /dev/sdX1  # Replace /dev/sdX1 with your partition
   ```
2. Open `/etc/fstab` with a text editor:
   ```bash
   sudo nano /etc/fstab
   ```
3. Add the following line:
   ```
   UUID=your_UUID /nfs ext4 defaults 0 2
   ```
   Replace `your_UUID` with the UUID obtained from the `blkid` command.
4. Save the file and exit the editor.
5. Apply the changes:
   ```bash
   sudo mount -a
   ```

---

## NFS Server Setup

On your NFS server, run the following commands to set up the NFS shared directory:

```bash
# Install NFS server packages
apt-get install nfs-common nfs-kernel-server -y

# Create the NFS directory (if not already created)
mkdir -p /nfs/kubernetes

# Set ownership and permissions
chown -R nobody: /nfs/kubernetes/

# Configure NFS exports
echo -e "/nfs/kubernetes\t*(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Apply the exports configuration
exportfs -av

# Restart the NFS server
systemctl restart nfs-server

# Check the status of the NFS server
systemctl status nfs-server

# Verify the NFS share
showmount -e localhost
```

---

## Kubernetes Cluster Setup

Ensure that your Kubernetes cluster is ready and that `nfs-common` is installed on all nodes. Then, proceed to install the NFS subdir external provisioner using Helm.

### Install NFS Subdir External Provisioner

Run the following commands on your Kubernetes cluster:

```bash
# Add the Helm repository
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

# Install the NFS subdir external provisioner
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=10.0.0.11 \
  --set nfs.path=/nfs/kubernetes/ \
  --set storageClass.onDelete=true \
  --set storageClass.name=nfs-storage
```

Replace `10.0.0.11` with the IP address of your NFS server.

### Verify Installation

After installing the provisioner, verify that the pods and storage classes are correctly set up:

```bash
# Check the status of the provisioner pod
kubectl get pod

# Check the storage classes
kubectl get sc
```

You should see a storage class named `nfs-storage` and a running pod for the NFS provisioner.

---

## Usage

To use the dynamically provisioned NFS storage, create a PersistentVolumeClaim (PVC) with the `nfs-storage` storage class. For example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-storage
```

Apply the PVC to your cluster:

```bash
kubectl apply -f nfs-pvc.yaml
```

The NFS provisioner will automatically create a PersistentVolume (PV) and bind it to the PVC.

---

## References

- [How to Setup Dynamic NFS Provisioning in a Kubernetes Cluster](https://hbayraktar.medium.com/how-to-setup-dynamic-nfs-provisioning-in-a-kubernetes-cluster-cbf433b7de29)
- [How to Deploy NFS Client Provisioner](https://medium.com/@dikkumburage/how-to-deploy-nfs-client-provionser-31407a3746c8)

---

Feel free to contribute to this repository by opening issues or pull requests. If you encounter any problems, please check the [References](#references) or open an issue in this repository.

Happy Kubernetes-ing! 🚀