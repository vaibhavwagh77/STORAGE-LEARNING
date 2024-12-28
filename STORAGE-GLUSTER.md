# GlusterFS Setup Instructions

## Step 1: Create Virtual Machines and Configure Hard Disks

### Virtual Machine Specifications
- **For Servers (3 VMs)**:
  - 1 CPU
  - 1 GB Memory
  - 1 Extra HDD
- **For Client (1 VM)**:
  - 1 CPU
  - 1 GB Memory

### Requirements
- All machines should be on the same network.
- Designate the VMs as:
  - `server1` (Storage Node 1)
  - `server2` (Storage Node 2)
  - `server3` (Storage Node 3)
  - `client` (Client Machine)

### Configure the New Hard Disk on Each Storage Node
1. **Create a mount point for the new HDD**
   ```bash
   mkdir -p /mnt/disk1
   ```
2. **Format and create a filesystem on the new disk** (adjust the disk path if needed):
   ```bash
   mkfs.ext4 /dev/sdb
   ```
3. **Mount the disk**:
   ```bash
   mount /dev/sdb /mnt/disk1
   ```
4. **Persist the mount by adding it to `/etc/fstab`**:
   ```bash
   echo "/dev/sdb /mnt/disk1 ext4 defaults 0 0" >> /etc/fstab
   ```

## Step 2: Update the `/etc/hosts` File
1. On each machine, update the `/etc/hosts` file with the following entries (replace `xxx` with the actual IP addresses):
   ```
   192.168.76.xxx  server1.hpcsa.com server1
   192.168.76.xxx  server2.hpcsa.com server2
   192.168.76.xxx  server3.hpcsa.com server3
   192.168.76.xxx  client.hpcsa.com client
   ```
2. Save the file and exit.

## Step 3: Install and Configure GlusterFS on Storage Nodes

### Install GlusterFS Server on All Storage Nodes
1. **Update the package list**:
   ```bash
   apt update
   ```
2. **Install GlusterFS server**:
   ```bash
   apt install -y glusterfs-server
   ```
3. **Start and enable the GlusterFS service**:
   ```bash
   systemctl start glusterd
   systemctl enable glusterd
   ```

### Add Other Storage Nodes to the GlusterFS Trusted Pool
1. **Run these commands on `server1` to add `server2` and `server3` to the pool**:
   ```bash
   gluster peer probe server2.hpcsa.com
   gluster peer probe server3.hpcsa.com
   ```
2. **Verify the trusted pool**:
   ```bash
   gluster peer status
   gluster pool list
   ```

## Step 4: Configure the GlusterFS Volume

### Create Directory for GlusterFS Volume
1. **On all storage nodes (`server1`, `server2`, `server3`), create the directory**:
   ```bash
   mkdir -p /mnt/disk1/diskvol
   ```

### Create Directory for GlusterFS Volume
1. **On all storage nodes (`server1`, `server2`, `server3`), create the directory**:
   ```bash
   mkdir -p /mnt/disk1/diskvol
   ```

### Create and Start the GlusterFS Volume
1. **Run these commands on `server1`**:
   ```bash
   gluster volume create gdisk1 replica 3 \
     server1:/mnt/disk1/diskvol/gdisk1 \
     server2:/mnt/disk1/diskvol/gdisk1 \
     server3:/mnt/disk1/diskvol/gdisk1
   ```
2. **Start the GlusterFS volume**:
   ```bash
   gluster volume start gdisk1
   ```
3. **View the GlusterFS volume information**:
   ```bash
   gluster volume info gdisk1
   ```

## Step 5: Configure the Client Machine

### Install GlusterFS Client
1. **Update the package list**:
   ```bash
   apt update
   ```
2. **Install GlusterFS client**:
   ```bash
   apt install -y glusterfs-client
   ```

### Mount the GlusterFS Volume
1. **Create a mount point**:
   ```bash
   mkdir -p /mnt/gdrive
   ```
2. **Mount the GlusterFS volume**:
   ```bash
   mount -t glusterfs server1:/gdisk1 /mnt/gdrive
   ```
3. **Verify the mount**:
   ```bash
   df -h /mnt/gdrive
   ```

### (Optional) Add the GlusterFS Volume to `/etc/fstab`
1. **Persist the mount by adding an entry to `/etc/fstab`**:
   ```bash
   echo "server1:/gdisk1 /mnt/gdrive glusterfs defaults,_netdev 0 0" >> /etc/fstab
   ```

## Step 6: Test the Configuration

1. **On the client machine, create files or directories in `/mnt/gdrive`**:
   ```bash
   dd if=/dev/zero of=/mnt/gdrive/1GB-file bs=1MB count=1024
   ```
2. **Verify that the changes are replicated across `server1`, `server2`, and `server3`.**
