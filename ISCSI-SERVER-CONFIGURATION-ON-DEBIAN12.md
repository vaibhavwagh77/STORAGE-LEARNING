# PVFS Lab Setup Instructions

## Resources
- **Steps Recording**: [OrangeFS.zip](http://192.168.82.230/OrangeFS.zip)
- **VM Image**: [CentOS8.zip](http://192.168.82.230/CENTSOS8.zip)
- **Credentials**:
  - Username: `root`
  - Password: `cdac`

## Create Virtual Machines

### Server
- **Add Extra HDD**.
- Set hostname: `ofs-srv-1`.

### Client
- Set hostname: `ofs-client-1`.

### Hosts File Entry
Update `/etc/hosts` on all servers/clients with:
```plaintext
192.168.1.x ofs-srv-1
192.168.1.x ofs-client-1
```
*(Replace `192.168.1.x` with actual IP addresses assigned to each server.)*

---

## Server Setup

### Prepare the Environment
1. Navigate to the yum repository directory and update repository settings:
   ```bash
   cd /etc/yum.repos.d/
   sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-* \
   && sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-* \
   && systemctl stop firewalld \
   && systemctl disable firewalld
   ```

2. Install `nano` and `epel-release`:
   ```bash
   yum install -y nano
   yum -y install epel-release
   ```

3. Edit SELinux configuration to disable it:
   ```bash
   nano /etc/selinux/config
   ```
   *(Change `SELINUX=enforcing` to `SELINUX=disabled`.)*

4. Import ELRepo GPG key:
   ```bash
   rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
   ```

5. Add ELRepo repository:
   ```bash
   echo "[elrepo]
   name=ELRepo.org Community Enterprise Linux Repository - el8
   baseurl=http://elrepo.org/linux/elrepo/el8/\$basearch/
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org

   [elrepo-kernel]
   name=ELRepo.org Community Enterprise Linux Kernel Repository - el8
   baseurl=http://elrepo.org/linux/kernel/el8/\$basearch/
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org" | sudo tee /etc/yum.repos.d/elrepo.repo
   ```

6. Install the latest kernel and reboot:
   ```bash
   yum -y --enablerepo=elrepo-kernel install kernel-ml
   reboot
   ```

7. Install OrangeFS server and create directories:
   ```bash
   yum -y install orangefs orangefs-server
   mkdir /mnt/ofsmnt
   lsblk
   ```

### Configure OrangeFS
1. Generate OrangeFS configuration:
   ```bash
   pvfs2-genconfig /etc/orangefs/orangefs.conf
   ```
   Follow the prompts:
   - Protocol type: `tcp`
   - Port number: `3334`
   - Data directory: `/mnt/ofsmnt`
   - Metadata directory: *(default)*
   - Hostname: `ofs-srv-1`
   - Verify server list: `y`
   - Metadata and I/O server list confirmation: `y`

2. Format and mount the new HDD:
   ```bash
   # Use appropriate disk tools to format and mount to `/mnt/ofsmnt`
   ```

3. Start the OrangeFS server:
   ```bash
   pvfs2-server -f /etc/orangefs/orangefs.conf
   ```

4. Add entry to `/etc/pvfs2tab`:
   ```bash
   nano /etc/pvfs2tab
   tcp://ofs-srv-1:3334/orangefs /mnt/ofsmnt pvfs2
   ```

5. Enable and start OrangeFS service:
   ```bash
   systemctl enable orangefs-server
   systemctl start orangefs-server
   systemctl status orangefs-server
   ```

6. Verify the server:
   ```bash
   pvfs2-ping -m /mnt/ofsmnt
   ```

---

## Client Setup

### Prepare the Environment
1. Navigate to the yum repository directory and update repository settings:
   ```bash
   cd /etc/yum.repos.d/
   sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-* \
   && sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-* \
   && systemctl stop firewalld \
   && systemctl disable firewalld
   ```

2. Install `nano` and `epel-release`:
   ```bash
   yum install -y nano
   yum -y install epel-release
   ```

3. Import ELRepo GPG key:
   ```bash
   rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
   ```

4. Add ELRepo repository:
   ```bash
   echo "[elrepo]
   name=ELRepo.org Community Enterprise Linux Repository - el8
   baseurl=http://elrepo.org/linux/elrepo/el8/\$basearch/
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org

   [elrepo-kernel]
   name=ELRepo.org Community Enterprise Linux Kernel Repository - el8
   baseurl=http://elrepo.org/linux/kernel/el8/\$basearch/
   enabled=1
   gpgcheck=1
   gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org" | sudo tee /etc/yum.repos.d/elrepo.repo
   ```

5. Install the latest kernel and reboot:
   ```bash
   yum -y --enablerepo=elrepo-kernel install kernel-ml
   reboot
   ```

6. Install OrangeFS client and FUSE module:
   ```bash
   yum -y install orangefs orangefs-fuse
   modprobe orangefs
   ```

### Mount the OrangeFS Volume
1. Create a mount point:
   ```bash
   mkdir /mnt/ofsmnt
   ```

2. Add entry to `/etc/pvfs2tab`:
   ```bash
   nano /etc/pvfs2tab
   tcp://ofs-srv-1:3334/orangefs /mnt/ofsmnt pvfs2
   ```

3. Verify the client configuration:
   ```bash
   pvfs2-ping -m /mnt/ofsmnt
   ```

4. Mount the OrangeFS volume:
   ```bash
   pvfs2fuse /mnt/ofsmnt/ -o fs_spec=tcp://ofs-srv-1:3334/orangefs
   df -h
   ```

### Test the Configuration
1. Navigate to the mount point and create a test file:
   ```bash
   cd /mnt/ofsmnt
   dd if=/dev/zero of=1GB-file bs=1MB count=1024
   
