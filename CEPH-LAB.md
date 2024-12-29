***********************************************************************************
**CEPH-LAB**
***********************************************************************************

--------------------------------------------------------------------------------------
##################    SERVERS     ######################
--------------------------------------------------------------------------------------

### Step 1: Install Ceph Repository on All Nodes

1. **Add the Ceph Quincy release and install Ceph dependencies on all server nodes**:
   - The `centos-release-ceph-squid` repository contains the required packages for installing Ceph.
   - `podman` is used as a container runtime.
   - `python3-jinja2` is required for templating.
   - Commands:
     ```bash
     sudo yum install -y centos-release-ceph-squid podman
     sudo yum update -y
     sudo yum install -y cephadm python3-jinja2
     ```

2. **Update the `/etc/hosts` file to include node details**:
   - This ensures that all nodes can communicate using hostname resolution.
   - Replace `192.168.230.xxx` with the actual IP addresses of your nodes.
   - Commands:
     ```bash
     echo "192.168.230.xxx node1" | sudo tee -a /etc/hosts
     echo "192.168.230.xxx node2" | sudo tee -a /etc/hosts
     echo "192.168.230.xxx node3" | sudo tee -a /etc/hosts
     echo "192.168.230.xxx client-node" | sudo tee -a /etc/hosts
     ```

### Step 2: SSH Configuration

1. **Generate SSH keys and distribute them across nodes**:
   - SSH keys are required for password-less authentication, which simplifies cluster deployment.
   - Commands:
     ```bash
     ssh-keygen
     ssh-copy-id root@node1
     ssh-copy-id root@node2
     ssh-copy-id root@node3
     ```

### Step 3: Ceph Cluster Deployment

1. **Bootstrap the Ceph cluster on the admin node**:
   - The bootstrap process sets up a minimal Ceph cluster with a monitor and manager daemon.
   - Replace `192.168.1.101` with the IP address of the admin node.
   - Command:
     ```bash
     cephadm bootstrap --mon-ip 192.168.1.101
     ```

2. **Distribute the configuration and keyring to other nodes**:
   - These files allow other nodes to join the cluster securely.
   - Commands:
     ```bash
     /usr/sbin/cephadm shell ceph cephadm get-pub-key > ceph.pub
     ssh-copy-id -f -i ceph.pub root@node2
     ssh-copy-id -f -i ceph.pub root@node3
     ```

3. **Add OSDs (Object Storage Daemons) to the cluster**:
   - OSDs are responsible for storing data, handling replication, and recovery.
   - Commands:
     ```bash
     /usr/sbin/cephadm shell ceph orch host add node1
     /usr/sbin/cephadm shell ceph orch host add node2
     /usr/sbin/cephadm shell ceph orch host add node3

     /usr/sbin/cephadm shell ceph orch daemon add osd node1:/dev/nvme1
     /usr/sbin/cephadm shell ceph orch daemon add osd node1:/dev/nvme2
     /usr/sbin/cephadm shell ceph orch daemon add osd node2:/dev/nvme1
     /usr/sbin/cephadm shell ceph orch daemon add osd node2:/dev/nvme2
     /usr/sbin/cephadm shell ceph orch daemon add osd node3:/dev/nvme1
     /usr/sbin/cephadm shell ceph orch daemon add osd node3:/dev/nvme2
     ```

4. **Verify the Ceph cluster status**:
   - Commands:
     ```bash
     cephadm shell ceph health
     cephadm shell ceph -s
     ```

5. **Create and configure CephFS**:
   - CephFS is a distributed file system that provides shared access to data stored in Ceph.
   - Commands:
     ```bash
     cephadm shell ceph osd pool create cephfs_metadata 8
     cephadm shell ceph osd pool create cephfs_data 8
     cephadm shell ceph fs new cephfs cephfs_metadata cephfs_data
     cephadm shell ceph fs ls
     ```

--------------------------------------------------------------------------------------
##################    CLIENT     ######################
--------------------------------------------------------------------------------------

### Step 4: Ceph Client Setup

1. **Install Ceph common packages**:
   - These packages allow the client to interact with the Ceph cluster.
   - Commands:
     ```bash
     sudo yum install -y centos-release-ceph-squid
     dnf install epel-release -y
     sudo yum install -y ceph-common
     ```

2. **Configure Ceph on the client node**:
   - Copy the Ceph configuration and keyring files to the client node.
   - Commands:
     ```bash
     scp /etc/ceph/ceph.conf root@client-node:/etc/ceph/
     scp /etc/ceph/ceph.client.admin.keyring root@client-node:/etc/ceph/
     sudo chmod +r /etc/ceph/ceph.client.admin.keyring
     ```

3. **Check the health and status of the Ceph cluster**:
   - Commands:
     ```bash
     sudo ceph health
     sudo ceph health detail
     sudo ceph -s
     ```

4. **Create a Ceph storage pool**:
   - Pools are logical partitions within the Ceph cluster for storing data.
   - Command:
     ```bash
     sudo ceph osd pool create rbd 128 3
     ```

### Step 5: Ceph Client Configuration

1. **Create and map a new block device**:
   - Block devices provide block storage using Ceph RADOS.
   - Commands:
     ```bash
     rbd create disk01 --size 4096
     rbd ls -l
     modprobe rbd
     rbd feature disable disk01 exclusive-lock object-map fast-diff deep-flatten
     rbd map disk01
     rbd showmapped
     ```

2. **Create a filesystem on the block device and mount it**:
   - Commands:
     ```bash
     mkfs.xfs /dev/rbd0
     mkdir -p /mnt/mydisk
     mount /dev/rbd0 /mnt/mydisk
     df -h
     dd if=/dev/zero of=/mnt/mydisk/file1.txt bs=1024 count=220040
     ```

3. **Mount the CephFS**:
   - CephFS allows clients to access the file system directly.
   - Commands:
     ```bash
     ceph fs ls
     ceph orch apply mds cephfs --placement="count:1"
     ceph mds stat
     ceph auth get-key client.admin
     echo -n "AQC7wHBnt+4GFhAAoUkcntGGFVfw2jXdEnWfUQ==" | base64
     sudo mkdir /mnt/cephfs
     sudo mount -t ceph node1:6789:/ /mnt/cephfs -o name=admin,secret=QVFDTjlIQm4zMmZyREJBQUZwZFVMNHBnUzBrUWlKSWdKTmJ3dFE9PQ==
     ```

### Notes:
- Ensure proper IP configuration for all nodes before starting the installation.
- Regularly check cluster health and resolve any issues reported by `ceph health`.
- Replace placeholder values (e.g., IP addresses, pool names) with actual values from your environment.

  **VAIBHAV ASHOK WAGH**
