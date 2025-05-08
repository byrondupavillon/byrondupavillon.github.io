---
title: "🛠️ How to Restore an AWS RDS Database"
date: 2025-05-08
description: "A step-by-step guide to restoring an AWS RDS database using automated backups and manual snapshots. Includes MySQL support, CLI tips, pitfalls to avoid, and best practices for disaster recovery."
categories:
  - aws
  - databases
tags:
  - aws
  - rds
  - database recovery
  - backups
  - cloud
  - devops
  - sysadmin
---

Restoring an AWS RDS database is a critical task for disaster recovery, fixing accidental data loss, or replicating production environments for testing. In this guide, we'll walk through restoring RDS instances using **automated backups** and **manual snapshots**, focusing primarily on **MySQL**, but the steps apply similarly to **PostgreSQL**.

> ⚠️ **Note**: AWS **does not support restoring snapshots to an existing RDS instance**. Restoring always creates a **new DB instance** with a different identifier.

---

## 🔄 Method 1: Restore from Automated Backups

**Automated backups** are enabled by default and allow point-in-time restore within the retention period (up to 35 days).

### 📝 Step-by-Step

1. **Log in** to the AWS Management Console → navigate to **RDS**.
2. In the left navigation pane, click **Automated backups**.
3. Locate your database and click **Actions → Restore to point in time**.
4. On the restore screen, configure:
   - **Restore time**: Specific point or latest
   - **DB instance identifier**: Unique name
   - **Instance class**: Choose size (e.g., `db.t3.medium`)
   - **Multi-AZ deployment**: Enable if needed for HA
   - **VPC/Subnet group**: Must align with your network
   - **Public access**: Enable only if necessary
   - **Availability Zone**: Optional for AZ-specific placement
   - **Other DB options**: As required for your app

   ![Restore Options](assets/img/posts/2025-05-08-aws-rds-database-restore/aws-restore-options.png)

5. Click **Restore to point in time**.
6. Track restore progress in the **RDS Console**.
7. When the instance becomes **Available**, connect using your DB client.
8. Update your **application’s connection string**.
9. **Test** thoroughly before going live.
10. Delete the old DB instance (if not needed) to avoid cost.
11. Set up **new backup settings** for the restored instance.
12. Document the restore and notify your team.

---

## 📦 Method 2: Restore from Manual Snapshots

**Manual snapshots** are created by you and retained until deleted. They’re ideal for long-term backups and controlled restore points.

### 📝 Step-by-Step

1. In the RDS Console, go to the **Snapshots** tab.
2. Select your **manual snapshot**.
3. Click **Actions → Restore snapshot**.
4. Provide configuration details:
   - **DB instance identifier** (must be new)
   - **Instance class**, **network settings**, etc.
5. Click **Restore DB Instance**.
6. Wait for the new instance to reach **Available** status.
7. Connect to it, test it, update apps, and clean up as needed.

---

## 🧠 Common Pitfalls

- 🔗 **Forgetting to update application endpoints** after restore
- ❌ **Using a conflicting DB instance identifier**
- 💸 **Leaving restored instances running** post-testing
- 🔐 **Restoring into a public subnet unintentionally**
- 👥 **Overlooking IAM or DB user permissions** on new instance

---

## 🧰 Bonus: Use AWS CLI

Prefer automation or scripting? Here’s how to restore via CLI:

```bash
# From automated backup (point in time)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier prod-db-restore \
  --restore-time 2025-05-01T03:45:00Z

# From manual snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-new-db \
  --db-snapshot-identifier my-snapshot-id
```