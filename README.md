# 🔐 AWS EC2 Lab – Recover EC2 Instance When `.pem` Key Is Lost

In this lab, I learned how to **regain access to an EC2 instance** after losing the original **SSH private key (.pem)** by injecting a new public key through **User Data** and **IAM instance profile permissions**.
I also verified that the instance was running an application on port 80 using **Docker** and confirmed recovery via the new key.

---

## 📋 Lab Overview

**Goal:**

* Regain SSH access to an EC2 instance after losing its key pair
* Modify **User Data** to inject a new authorized key
* Restore application accessibility once the instance is recovered

**Learning Outcomes:**

* Understand how **EC2 User Data** can be used to inject a new SSH public key
* Learn to safely stop, edit, and restart an EC2 instance for recovery
* Configure security groups to re-enable SSH temporarily

---

## 🛠 Step-by-Step Journey

### Step 1 – Log In & Set Region

Logged into the **AWS Management Console** using provided credentials.
Region: **US East (N. Virginia)** → `us-east-1`.
✅ Environment ready.

---

### Step 2 – Provision Base Resources via Terraform

From the lab terminal:

```bash
cd /app/terraform_files/stack && terraform init && terraform apply -auto-approve
```

This script provisions:

* EC2 instance `app1`
* Security group `allow_tls`
* Region: `us-east-1`

✅ Infrastructure successfully deployed.

---

### Step 3 – Understand the Recovery Challenge

* EC2 instance `app1` is running a Docker-based web application on **port 80**.
* The original `.pem` SSH key was **lost**.
* Goal: Recover SSH access without recreating the instance.

---

### Step 4 – Locate Replacement Key Files

In the terminal:

```bash
cat ~/.ssh/test.pub      # Public key
cat ~/.ssh/test          # Private key
```

✅ Verified replacement key pair (`test`).

---

### Step 5 – Stop the Affected EC2 Instance

In **AWS Console → EC2 → Instances:**

1. Select **app1**
2. Choose **Instance State → Stop instance**
3. Wait until the state = *stopped*

✅ Instance safely stopped.

---

### Step 6 – Inject New Public Key via User Data

1. With instance stopped → **Actions → Instance Settings → Edit User Data**
2. Clear any existing content
3. Paste this script (replace `<YOUR_PUBLIC_KEY>` with output of `cat ~/.ssh/test.pub`):

```bash
#!/bin/bash
mkdir -p /home/ubuntu/.ssh
echo "<YOUR_PUBLIC_KEY>" >> /home/ubuntu/.ssh/authorized_keys
chmod 700 /home/ubuntu/.ssh
chmod 600 /home/ubuntu/.ssh/authorized_keys
chown -R ubuntu:ubuntu /home/ubuntu/.ssh
```

4. Save changes

✅ User Data updated to insert the new key at startup.

---

### Step 7 – Allow SSH Inbound Access

In **EC2 → Security Groups → allow_tls → Edit Inbound Rules:**

| Type | Protocol | Port | Source    |
| ---- | -------- | ---- | --------- |
| SSH  | TCP      | 22   | 0.0.0.0/0 |

✅ Rule added to allow temporary SSH access.

---

### Step 8 – Restart the Instance

Return to **Instances → app1 → Instance State → Start instance**
Wait until:

* **State:** Running
* **Status Check:** 2/2 checks passed

✅ Instance booted successfully.

---

### Step 9 – Verify SSH Access with New Key

In the terminal:

```bash
ssh -i ~/.ssh/test ubuntu@<PUBLIC_IP>
```

When prompted:

```
Are you sure you want to continue connecting (yes/no)? yes
```

✅ Successfully connected to `app1` using the new key pair.

---

## 🏁 End of Lab

### ✅ Key Command Summary

| Task                 | Command                                 |
| -------------------- | --------------------------------------- |
| Initialize Terraform | `terraform init`                        |
| Apply Infrastructure | `terraform apply -auto-approve`         |
| Show Public Key      | `cat ~/.ssh/test.pub`                   |
| Edit User Data       | *via Console → Instance Settings*       |
| SSH into EC2         | `ssh -i ~/.ssh/test ubuntu@<public-ip>` |

---

### 💡 Notes / Tips

* **User Data** executes as *root* at instance launch — perfect for injecting new keys.
* Always **stop** the instance before modifying User Data.
* Revoke the temporary **0.0.0.0/0 SSH rule** after regaining access for security.
* Maintain at least one **backup key pair** for production environments.
* Consider using **AWS Systems Manager Session Manager** as a no-key fallback.

---

### ✅ References

* [AWS Docs: Recover Access When Key Pair Is Lost](https://repost.aws/knowledge-center/ec2-recover-access-key-pair)
* [Amazon EC2 User Data Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
* [AWS Security Groups Guide](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
