# AWS Storage & Instance Initialization Guide
*An educational manual covering EBS Volume Management, Cloud-Init, and Disk Recovery Procedures.*

---

## 1. Core Concepts & Architecture

### A. Can an EBS Volume Be Modified on a Running EC2 Instance?
**Yes.** AWS supports **Elastic Volumes**, which allows you to modify the volume type, increase the size, and adjust the performance (IOPS and throughput) of your EBS volumes on-the-fly while they are attached to a running EC2 instance.

> [!IMPORTANT]
> **Key Constraints for Live Modifications:**
> * You can **only increase** the size of a volume. Decreasing volume size is not supported; to shrink a volume, you must create a new smaller volume and copy files over.
> * There is a **cooldown period** (usually 6 hours) before you can modify the same volume again.
> * While the storage capacity increases on the AWS hardware side immediately, the OS filesystem does not automatically expand to use the new space. You must manually grow the partition and filesystem.

---

### B. Root Volumes vs. Attached Data Volumes & `/dev/xvda`

| Feature | Root Volume | Attached Data Volume |
| :--- | :--- | :--- |
| **Purpose** | Contains the OS, bootloader, system files, and default user configurations. | Used to isolate application data, databases, container storage, or log files. |
| **Default Lifecycle** | Deleted automatically when the instance is terminated (`DeleteOnTermination = true`). | Retained by default when the instance is terminated (`DeleteOnTermination = false`), preserving data. |
| **Performance Needs** | Typically standard performance; low latency for OS system operations. | Often optimized for throughput or transaction rates depending on application. |

#### What is `/dev/xvda`?
* **/dev/xvda** is the device node name assigned by the Linux kernel to the primary virtual disk (virtual disk `a`).
* The `xvd` prefix stands for **Xen Virtual Disk** (traditionally used on AWS Xen hypervisors, and maintained for compatibility on modern AWS Nitro hypervisors).
* `/dev/xvda1` represents the first partition on that disk, which typically houses the root (`/`) filesystem.

---

### C. IOPS vs. Throughput (MiB/s)

```
+-------------------------------------------------------------+
|                      THE HIGHWAY ANALOGY                    |
|                                                             |
|   IOPS (Input/Output Operations Per Second):                 |
|   "How many individual vehicles pass a point per second"    |
|   [o] [o] [o] [o] [o] [o] [o] [o] [o] [o]                   |
|                                                             |
|   THROUGHPUT (MiB/s):                                       |
|   "The total weight of the cargo transported per second"    |
|   [======= TRUCK =======] [======= TRUCK =======]           |
+-------------------------------------------------------------+
```

* **IOPS (Input/Output Operations per Second)**:
  * **Definition**: The number of individual read or write tasks a storage device can perform in one second.
  * **Critical For**: High-transaction workloads requiring rapid access to random data points (e.g., relational databases like PostgreSQL/MySQL, search indexes, transactional applications).
* **Throughput (MiB/s)**:
  * **Definition**: The total volume of data that can be read from or written to the storage device in one second.
  * **Critical For**: Large-block, sequential read/write operations (e.g., log aggregation, backups, data warehouses, streaming analytics, video rendering).

---

### D. GP2 vs. GP3 Volume Types

AWS offers two main generations of General Purpose SSDs. Choosing the right one significantly impacts performance and cost:

| Feature | GP2 (General Purpose SSD) | GP3 (Next-Generation SSD) |
| :--- | :--- | :--- |
| **Performance Binding** | **Coupled**: IOPS and Throughput scale directly with the volume size (3 IOPS per GiB). | **Decoupled**: You can provision IOPS and Throughput independently of volume size. |
| **Baseline Performance** | Min 100 IOPS, scales up to 16,000 IOPS. Baseline throughput scales up to 250 MiB/s. | **3,000 IOPS** and **125 MiB/s** baseline included *free* regardless of volume size (even for a 1 GiB volume). |
| **Maximum Performance** | 16,000 IOPS / 250 MiB/s | 16,000 IOPS / 1,000 MiB/s |
| **Cost Efficiency** | More expensive for small volumes needing high performance. | **~20% cheaper** per GB than GP2, plus savings from not over-provisioning storage for performance. |

---

### E. What is `cloud-init`?
`cloud-init` is a widely-used cloud instance initialization package. When a cloud instance boots for the first time, `cloud-init` executes several bootstrap tasks:
1. **Metadata Querying**: Contacts the AWS Instance Metadata Service (IMDS) at the link-local address `http://169.254.169.254/`.
2. **Key Injection**: Retrieves the public SSH key specified at instance launch and appends it to the default user's authorized keys file (e.g., `/home/ec2-user/.ssh/authorized_keys`).
3. **Provisioning**: Configures hostname, network interfaces, updates packages, and runs custom User Data scripts.

---

## 2. Lab Demo: Sabotage & Recovery Walkthrough

This demo illustrates what happens when you strip an OS of its cloud-aware provisioning agents (`cloud-init`), clone it, exhaust the disk space, and perform emergency system recovery.

### Phase 1: Observing Healthy `cloud-init`
1. Spin up **Instance A** in EC2 using an Amazon Linux 2023 AMI. Launch it with a key pair named `Key-Original`.
2. SSH into **Instance A** using `Key-Original`:
   ```bash
   ssh -i Key-Original.pem ec2-user@<Instance-A-IP>
   ```
3. Inspect `cloud-init` to verify it ran successfully:
   ```bash
   sudo systemctl status cloud-init
   tail -n 20 /var/log/cloud-init.log
   ```
   * **Observation**: You will see logs showing that `cloud-init` queried `169.254.169.254`, retrieved your public key, and placed it inside `/home/ec2-user/.ssh/authorized_keys`.

---

### Phase 2: The Sabotage (Creating a "Naked" OS)
Now, simulate a bare metal OS that has lost its cloud capabilities.

1. In your active terminal on **Instance A**, run the following commands to strip out cloud-init, SSM components, and key injection configurations:
   ```bash
   # Remove cloud-init and EC2 Instance Connect
   sudo dnf remove cloud-init -y
   sudo dnf remove ec2-instance-connect -y

   # Delete cached state and configurations
   sudo rm -rf /etc/cloud/ /var/lib/cloud/

   # Restart SSH to apply clean state
   sudo systemctl restart sshd

   # Power down the machine safely
   sudo poweroff
   ```
2. Navigate to the AWS Console, select **Instance A**, and choose **Actions > Image and templates > Create image** to create a custom AMI (e.g., `AMI-Sabotaged`).

---

### Phase 3: Launching Server B & Key Behavior
1. Launch a new instance, **Instance B**, using your custom `AMI-Sabotaged`.
2. During launch, select a **new** key pair: `Key-New`.
3. Try to SSH using `Key-New`:
   ```bash
   ssh -i Key-New.pem ec2-user@<Instance-B-IP>
   ```
   * **Result**: **FAIL (Permission denied / Connection closed)**.
   * **Why?** Since `cloud-init` was uninstalled, there is no service running on boot to query the IMDS metadata and write the public key of `Key-New` into the authorized keys file.
4. Try to SSH using the original key, `Key-Original`:
   ```bash
   ssh -i Key-Original.pem ec2-user@<Instance-B-IP>
   ```
   * **Result**: **SUCCESS**.
   * **Why?** During AMI creation, AWS clones the root disk snapshot. The file `/home/ec2-user/.ssh/authorized_keys` already contained the public key for `Key-Original` from **Instance A**. Since we did not delete that file, it remains baked into the AMI.

---

### Phase 4: Disk Exhaustion (SSM Agent vs. SSH Agent)
1. Log into **Instance B** using `Key-Original`.
2. Deliberately fill up the root disk partition to 100% capacity:
   ```bash
   # Create a massive file to exhaust disk space
   sudo dd if=/dev/zero of=/var/tmp/large_file_2.img bs=1M status=progress
   
   # Write remaining space blocks to lock up the storage fully
   sudo dd if=/dev/zero of=/home/ec2-user/exhaust_disk bs=1k
   ```
3. Run `df -h` to verify that `/` is at **100% usage**.

#### Which is heavier: SSM Agent or SSH?
* **SSM Agent (AWS Systems Manager Agent)** is significantly **heavier** and more fragile under disk exhaustion than SSH.
* **Why SSM Agent Fails**: SSM is a daemon written in Go. It maintains persistent HTTPS WebSockets with the Systems Manager service, frequently reading/writing session state, telemetry, and log files in `/var/log/amazon/ssm/`. When the disk is 100% full, the SSM agent cannot write to its logs, create local temp files, or initialize sub-processes, causing the SSM connection to drop and lock you out.
* **Why SSH Survives**: SSH (`sshd`) is a lightweight C daemon. It handles connections directly via standard TCP/22 and requires very few system resources. Even when the disk is completely full, SSH can usually authenticate you (unless system logs like `/var/log/secure` block write calls dramatically, which is rare) allowing you to connect and run local commands to clean up.

---

### Phase 5: Step-by-Step Volume Expansion & Filesystem Resizing

When your disk is 100% full, SSM is disconnected, and you need to scale the volume size from **8 GB to 20 GB**, follow this recovery procedure.

```
                  LVM/EBS EXPANSION WORKFLOW
                  
   +---------------+      +-----------------+      +---------------------+
   | 1. AWS Console|      | 2. Partition    |      | 3. Filesystem       |
   | Modify EBS    | ---> | growpart        | ---> | xfs_growfs          |
   | (8G -> 20G)   |      | (Resize xvda1)  |      | (Expand to fill /)  |
   +---------------+      +-----------------+      +---------------------+
```

#### Step 1: Modify the EBS Volume in AWS Console
1. Navigate to the **EC2 Dashboard** and click **Volumes**.
2. Select the volume attached to your instance.
3. Click **Actions > Modify volume**.
4. Change the size from `8` GiB to `20` GiB.
5. Click **Modify** and confirm. (Wait until the Volume State changes to `in-use - optimizing` or `in-use - completed`).

#### Step 2: Inspect Disk State inside Instance B
Log in via SSH (using `Key-Original`) and run:
```bash
lsblk
```
*You will see:*
* The parent disk device (`xvda`) is now **20G**.
* The partition (`xvda1`) is still only **8G**.

Now run:
```bash
df -h
```
*You will see:*
* The root filesystem (`/`) is still only **8G** in size and **100% full**.

#### Step 3: Expand the Partition
Expand partition `1` on disk `/dev/xvda` to use the newly allocated EBS space:
```bash
sudo growpart /dev/xvda 1
```
* **Verification**: Run `lsblk` again. You should now see that `/dev/xvda1` has grown to **20G**.

#### Step 4: Expand the Filesystem
Depending on the filesystem type (Amazon Linux 2023 defaults to **XFS**), execute the corresponding command:

* **For XFS Filesystem (default):**
  ```bash
  sudo xfs_growfs /
  ```
  *(Note: The `/` argument specifies the mount point you are expanding).*

* **For Ext4 Filesystem (if applicable):**
  ```bash
  sudo resize2fs /dev/xvda1
  ```

#### Step 5: Verify Final Space Allocation
Check the disk space one final time:
```bash
df -h
```
* **Result**: The root filesystem (`/`) now reflects **20G** total capacity with plenty of free space, and the system is fully operational again.
