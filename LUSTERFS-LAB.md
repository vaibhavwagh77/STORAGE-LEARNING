# LustreFS Lab Setup

This guide provides detailed instructions and explanations for setting up LustreFS with Meta Server (MDS), Object Storage Servers (OST), and a client node on CentOS 8.

---

### Hosts Configuration

Add the host entries for all nodes in `/etc/hosts` on each server and client:

```bash
# Replace '192.168.230.xxx' with the actual IP addresses of each node
echo "192.168.230.xxx  node1" | sudo tee -a /etc/hosts
echo "192.168.230.xxx  node2" | sudo tee -a /etc/hosts
echo "192.168.230.xxx  node3" | sudo tee -a /etc/hosts
echo "192.168.230.xxx  client" | sudo tee -a /etc/hosts
```
- `sudo tee -a /etc/hosts`: Appends the specified entry to the `/etc/hosts` file, mapping IP addresses to hostnames for name resolution.

---

## Meta Server (MDS) Configuration

1. **Update Repositories and Disable Firewalld**

```bash
cd /etc/yum.repos.d/ && \
    sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-* && \
    sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-* && \
    systemctl stop firewalld && systemctl disable firewalld
```
- **`sed`**: Edits repository configuration to use `vault.centos.org` instead of default mirrors.
- **`systemctl stop`** and **`disable firewalld`**: Stops and prevents the firewall from starting automatically.

2. **Install Nano and Dependencies**

```bash
yum install -y nano
dnf config-manager --set-enabled powertools
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
dnf install https://zfsonlinux.org/epel/zfs-release-2-3$(rpm --eval "%{dist}").noarch.rpm
dnf install kernel-headers kernel-devel
dnf upgrade kernel
reboot
```
- **`nano`**: A text editor for editing configuration files.
- **`dnf config-manager`**: Enables PowerTools repository, required for additional software.
- **`dnf install`**: Installs EPEL and ZFS repositories and upgrades the kernel.
- **`reboot`**: Reboots the system to apply the new kernel.

3. **Install ZFS and Lustre Server**

```bash
dnf install zfs
modprobe -v zfs
```
- **`dnf install zfs`**: Installs ZFS filesystem tools.
- **`modprobe`**: Loads the ZFS kernel module.

4. **Add Lustre Server Repository**

```bash
echo "[lustre-server]
name=lustre-server
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.4/el8.9/server/
exclude=*debuginfo*
enabled=1
gpgcheck=0" | sudo tee /etc/yum.repos.d/lustre.repo
```
- **`sudo tee`**: Writes the Lustre server repository configuration to `/etc/yum.repos.d/lustre.repo`.

5. **Install Lustre Packages**

```bash
dnf install -y lustre-dkms lustre-osd-zfs-mount lustre kmod-lustre
modprobe -v lustre
lsmod | grep lustre
```
- **`dnf install`**: Installs Lustre server components.
- **`modprobe -v`**: Verifies Lustre modules are loaded.
- **`lsmod`**: Lists loaded modules, confirming Lustre is active.

6. **Prepare ZFS Storage for MDS**

```bash
lsblk
parted /dev/nvme0n2 mklabel gpt
parted -a optimal /dev/nvme0n2 mkpart primary 0% 100%
zpool create mds_pool /dev/nvme0n2p1
zfs create mds_pool/mdt0
zfs set atime=off mds_pool/mdt0
zpool list
```
- **`lsblk`**: Displays block devices to identify the target disk.
- **`parted`**: Creates a GPT partition table and a partition on the specified disk.
- **`zpool create`**: Creates a ZFS pool for metadata storage.
- **`zfs create`**: Sets up a dataset for the Lustre metadata target (MDT).
- **`zfs set atime=off`**: Disables access time updates for performance.

7. **Format and Mount MDT**

```bash
umount /mds_pool/mdt0
umount /mds_pool
mkfs.lustre --reformat --mdt --fsname=lustrefs --mgs --index=0 --backfstype=zfs mds_pool/mdt0
mkdir /mnt/mdt0/
mount -t lustre mds_pool/mdt0 /mnt/mdt0
```
- **`mkfs.lustre`**: Formats the dataset as an MDT, associating it with the Management Server (MGS).
- **`mount -t lustre`**: Mounts the MDT for use.

8. **Configure LNET and Test Connectivity**

```bash
nano /etc/modprobe.d/lnet.conf
# Add: options lnet networks="tcp0(ens160)"

modprobe lnet
lsmod | grep lnet
lctl network up
lctl ping <MGS-SERVER-IP>@tcp0
```
- **`lnet`**: Configures the Lustre Network Toolkit for communication between nodes.
- **`lctl ping`**: Tests connectivity to the Management Server.

---

## Object Storage Servers (OST)

Follow similar steps for configuring ZFS, Lustre, and networking. Replace the `mkfs.lustre` command for OST configuration:

```bash
mkfs.lustre --ost --fsname=lustrefs --mgsnode=<MGS SERVER IP>@tcp --index=0 --backfstype=zfs ost_pool1/ost0
```

---

## Client Configuration

1. **Install Lustre Client Packages**

```bash
dnf install -y lustre-client lustre-client-dkms kmod-lustre-client
modprobe lustre
```

2. **Mount Lustre Filesystem**

```bash
mkdir /mnt/lustre
mount -t lustre <MGS-SERVER>@tcp0:/lustrefs /mnt/lustre
lctl dl
```
- **`mount -t lustre`**: Mounts the Lustre filesystem from the Management Server.
- **`lctl dl`**: Lists Lustre devices on the client.

---

### Verification

- Check Lustre devices:

```bash
lfs check servers
lfs osts
```
- Verify mounted filesystem:

```bash
df -h
