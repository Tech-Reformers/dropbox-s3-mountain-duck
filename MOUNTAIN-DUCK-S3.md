# Replacing Dropbox with Amazon S3 + Mountain Duck (macOS & Windows)

Use an S3 bucket as your "Dropbox": browse, open, and save files in Finder or File Explorer, with S3 as the storage behind it. The same bucket works from multiple computers.

Companion to **How to Sync Dropbox to Amazon S3 (rclone on EC2)**. Use that guide first to move your Dropbox content into the bucket, then this one to work with it day to day.

Throughout, replace **`YOUR-BUCKET`** and **`YOUR-REGION`** (for example, `us-west-2`) with your own values.

> **If you use AWS Organizations or Control Tower:** keep the bucket and its IAM user in a **member account**, not the management account. The management account shouldn't hold workloads or long-lived access keys, and SCPs don't apply to it.

---

## How it fits together

| Dropbox feature | S3 + Mountain Duck equivalent |
|---|---|
| Files in Finder | Mountain Duck bookmark in **Integrated** mode |
| Offline files | Mountain Duck "Keep Offline" / local cache |
| Version history and undelete | S3 **Versioning** |
| Cheap long-term storage | S3 **Intelligent-Tiering** (via a lifecycle rule) |
| Share a link | Mountain Duck **Copy URL / Share** (pre-signed S3 link) |
| Protection if your AWS account is compromised | **Cross-account replication** to a locked-down backup bucket (Step 6) |

---

## Step 1 — Harden the bucket

In the S3 console → `YOUR-BUCKET`:

1. **Permissions → Block public access:** all four settings **On**.
2. **Properties → Bucket Versioning:** **Enable**. This replaces Dropbox's version history and deleted-file recovery.
3. **Properties → Default encryption:** SSE-S3 (the default) is fine.
4. **Lifecycle rule.** The quickest way is CloudShell: click the `>_` icon in the console's top bar and paste:

```bash
cat > lifecycle.json <<'EOF'
{
  "Rules": [
    {
      "ID": "dropbox-defaults",
      "Status": "Enabled",
      "Filter": {},
      "Transitions": [{ "Days": 0, "StorageClass": "INTELLIGENT_TIERING" }],
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90 },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 },
      "Expiration": { "ExpiredObjectDeleteMarker": true }
    }
  ]
}
EOF
aws s3api put-bucket-lifecycle-configuration --bucket YOUR-BUCKET --lifecycle-configuration file://lifecycle.json
aws s3api get-bucket-lifecycle-configuration --bucket YOUR-BUCKET
```

The `put` command prints nothing when it succeeds, and `get` prints the rule back.

**Or in the console:** go to **Management → Lifecycle rules → Create rule**, apply it to all objects, and check:
- **Transition current versions** → **Intelligent-Tiering**, **0** days
- **Permanently delete noncurrent versions** → **90** days
- **Delete expired object delete markers** and **Delete incomplete multipart uploads** (**7** days)

**What the rule does:**
- **Intelligent-Tiering:** Mountain Duck uploads files as Standard, and the rule moves them over. Untouched files then drop to the Infrequent Access tier after 30 days and to Archive Instant Access after 90, with **no retrieval fees**. That beats putting files directly in Standard-IA, which charges for every read (including thumbnails and previews) and has a 30-day minimum. Objects under 128 KB stay in Standard, because tiering wouldn't save anything on them.
- **Noncurrent versions after 90 days:** this is your **undo window** for deletes and overwrites. See *How deletes work* in Step 5.
- S3 applies lifecycle rules asynchronously, usually within 24–48 hours.

> **Want deleted files recoverable indefinitely?** Use `"NoncurrentVersionExpiration": { "NoncurrentDays": 90, "NewerNoncurrentVersions": 1 }`. The last version of any deleted file is kept indefinitely, while older edit history still expires. The trade-off is that files you *meant* to delete also stay until you permanently delete their versions by hand.

## Step 2 — Create a dedicated IAM user

Mountain Duck uses an access key, so give it a user that can reach **only this bucket**.

1. IAM → Users → **Create user** → name it `mountainduck-YOUR-BUCKET`. **No console access.**
2. Attach an inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListAllMyBuckets", "s3:GetBucketLocation"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:ListBucketVersions", "s3:GetBucketVersioning"],
      "Resource": "arn:aws:s3:::YOUR-BUCKET"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET/*"
    }
  ]
}
```

3. User → **Security credentials → Create access key** → use case **Third-party service** (or "Other"). Copy the **Access key ID** and **Secret access key**. The secret is shown only once.

> **Why an IAM user instead of SSO?** IAM Identity Center is AWS's preferred approach, but its sessions expire after at most 12 hours. For a drive that syncs in the background, the drive would silently stop syncing until you signed in again. A long-lived key limited to one bucket is the practical choice here. The policy deliberately leaves out `s3:DeleteObjectVersion`, so even a stolen key can't erase your version history.

> **One key per computer.** Create a separate access key for each machine. An IAM user can hold two, so one for the Mac and one for the PC. For more than two machines, create another user with the same policy. If one computer is lost, you deactivate only its key, and the others keep working.

> **Key hygiene:** Mountain Duck stores the secret in the macOS Keychain. Never put the key in a file or a shared document. Rotate it periodically, by creating a new key, updating the bookmark, and deleting the old key, and immediately if a Mac is lost. A key scoped to one bucket limits the damage if it leaks.

## Step 3 — Install Mountain Duck

1. Download from [mountainduck.io](https://mountainduck.io) and install. A license is required after the trial.
2. If macOS prompts you, allow the **File Provider** extension: System Settings → General → Login Items & Extensions → File Provider → Mountain Duck **on**.

## Step 4 — Create the S3 bookmark

Mountain Duck menu bar icon → **New Bookmark**:

| Field | Value |
|---|---|
| Connection type | **Amazon S3** |
| Server | `s3.amazonaws.com` (default) |
| Access Key ID | Key from Step 2 |
| Secret Access Key | Secret from Step 2 (prompted on first connect, then saved to Keychain) |
| Path | `/YOUR-BUCKET` (opens straight to your bucket instead of listing all buckets) |
| Nickname | Something short, for example `S3 Files`. This is the name you'll see in Finder. |
| Connect Mode | **Integrated** |

Connect. The bucket appears in the Finder sidebar under **Locations**, and its files live under `~/Library/CloudStorage/`.

### About Integrated mode

Integrated mode uses macOS **File Provider**, the same framework Dropbox and iCloud Drive use:

- Files show as placeholders until you open them, then download on demand.
- Right-click a file or folder → **Keep Offline** (wording may vary by version) to pin it locally.
- Changes upload in the background. Check status from the Mountain Duck menu bar icon.

## Step 4b — Windows PC

1. Download the Windows version from [mountainduck.io](https://mountainduck.io) and install it. Check your license terms, since a license may or may not cover both platforms.
2. System tray icon → **New Bookmark**, with the same settings as the Mac:

| Field | Value |
|---|---|
| Connection type | **Amazon S3** |
| Server | `s3.amazonaws.com` |
| Access Key ID / Secret | This PC's own key (Step 2) |
| Path | `/YOUR-BUCKET` |
| Nickname | Same name as on the Mac, so both machines look alike |
| Connect Mode | **Integrated** |

3. Connect. On Windows, Integrated mode uses the same Cloud Files framework as OneDrive. The bucket appears in the **File Explorer navigation pane**, with the familiar cloud and check-mark status icons.
   - Right-click → **Always keep on this device** pins a file locally.
   - Right-click → **Free up space** returns it to online-only.
4. The secret is stored in **Windows Credential Manager**.

## Using more than one computer

- **Both machines see the same files.** Changes from one appear on the other after Mountain Duck refreshes. Reopen the folder, or use Refresh from the menu bar or tray icon.
- **There's no file locking.** If the same file is edited on both machines before either syncs, the **last save wins**. With versioning on (Step 1), the other edit isn't lost: recover it from **Show versions** in the S3 console. Avoid having the same document open on both machines at once.
- **Keep Offline / Always keep on this device is per machine.** Pin what each computer needs.
- **Rename and move folders on one machine at a time.** In S3, a folder rename is a copy plus a delete of every object under it, so large folders take a while.

## Step 5 — Daily use

- **Open, edit, and save** files in place, like any Finder folder.
- **Share a file:** right-click → Mountain Duck → **Copy URL** / **Share**. This creates a **pre-signed link that expires**. Links signed with an IAM user key last at most **7 days**, and the bucket stays private.
- **Recover a deleted or overwritten file:** S3 console → bucket → turn on **Show versions** → find the older version → download it, or delete the delete marker to restore the file.

### How deletes work

With versioning on, deleting a file, whether in Finder, File Explorer, or with `aws s3 rm`, **doesn't erase anything right away**:

1. S3 adds a **delete marker** as the file's new current version. The file disappears from Mountain Duck and from `aws s3 ls`.
2. The real data stays underneath as a **noncurrent version**, and you're still billed for it.

| When | What happens |
|---|---|
| Day 0–90 | **Undo window.** To restore, go to S3 console → **Show versions**, find the file's **delete marker**, and delete the marker. The file reappears. |
| Day 90 | The lifecycle rule permanently removes the noncurrent version, and billing stops. |
| Shortly after | The leftover delete marker is cleaned up automatically. |

Overwriting a file works the same way: the previous version becomes noncurrent and is kept for 90 days.

**To delete something permanently right away** (for example, files you're sure about), turn on **Show versions** in the console, select the versions themselves, and choose **Delete**. This requires admin credentials; the Mountain Duck key can't do it, by design.

---

## Step 6 — Protect it: cross-account replication (recommended)

Versioning protects against ransomware on your computers and accidental deletes. It doesn't protect against someone with **admin access to the AWS account**, such as stolen root or admin credentials, or a mistake made with them. Replicating to a bucket in a **separate account** closes that gap: your day-to-day credentials can't touch the copy.

> **Why not Object Lock on the main bucket?** Compliance mode can't be undone, not even by root, which is a trap for personal files. Governance mode can be bypassed by any admin. Replication to another account is simpler and protects against the right threat. Object Lock does fit on the *replica* bucket (Step 6d).

**The moving parts:** cross-account replication involves two accounts and three permission pieces: a replication role (created for you in 6b), a bucket policy on the replica (6c), and a separate role for the one-time **Batch Replication** of files that already exist (6c, scripted). Do the steps in order.

| # | Account | Step |
|---|---|---|
| 6a | Backup | Create the replica bucket with versioning on |
| 6b | Source | Create the replication rule, then copy its role ARN |
| 6c | Backup, then Source | Add the replica bucket policy, then run the Batch Replication script |
| 6d | Backup | Add the lifecycle rule (plus optional SCP and Object Lock) |
| 6e | Source | Verify |

**Terms used below:**
- **Source account:** where `YOUR-BUCKET` lives.
- **Backup account:** a separate account, ideally a dedicated one in your AWS Organization that you rarely sign in to.
- `SOURCE-ACCOUNT-ID` and `BACKUP-ACCOUNT-ID`: the 12-digit account IDs.
- `YOUR-BUCKET-replica`: the destination bucket.

### 6a. Create the destination bucket (backup account)

1. S3 → **Create bucket** → name it `YOUR-BUCKET-replica`.
   - **Same region:** no data transfer charges.
   - **Another region:** also survives a regional outage, at about $0.02/GB transferred.
2. **Block all public access:** On.
3. **Bucket Versioning:** **Enable**. Replication requires it.
4. **Default encryption:** SSE-S3.

### 6b. Create the replication rule (source account)

In S3 → `YOUR-BUCKET` → **Management → Replication rules → Create replication rule**:

| Setting | Value |
|---|---|
| Rule scope | Apply to **all objects** in the bucket |
| Destination | **Specify a bucket in another account** → `BACKUP-ACCOUNT-ID` and `YOUR-BUCKET-replica` |
| Object ownership | **Change object ownership to destination bucket owner** |
| IAM role | **Create new role** |
| Destination storage class | **Glacier Instant Retrieval**, for fast restores. Use **Deep Archive** if this is a last resort only. |
| Delete marker replication | **Leave unchecked.** Deletes in the source should *not* propagate to the backup. |
| Replication Time Control | Off (not needed) |

Save. When the console asks whether to **replicate existing objects**, choose **No for now**. The destination bucket doesn't allow writes yet, so a job started now would fail.

Then open the rule and copy the **IAM role ARN** it created. You need it for the next step.

### 6c. Allow the source to write (backup account)

`YOUR-BUCKET-replica` → **Permissions → Bucket policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReplicationFromSource",
      "Effect": "Allow",
      "Principal": { "AWS": "SOURCE-REPLICATION-ROLE-ARN" },
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags",
        "s3:ObjectOwnerOverrideToBucketOwner"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-replica/*"
    },
    {
      "Sid": "AllowVersioningCheck",
      "Effect": "Allow",
      "Principal": { "AWS": "SOURCE-REPLICATION-ROLE-ARN" },
      "Action": ["s3:GetBucketVersioning", "s3:ListBucket"],
      "Resource": "arn:aws:s3:::YOUR-BUCKET-replica"
    }
  ]
}
```

> `s3:ReplicateDelete` only lets delete markers replicate if you enable that in the rule, and you didn't. Permanent version deletes never replicate.

Once that policy is saved, **new and changed files replicate automatically.** The only thing left is copying the files that were already in the bucket.

**Now copy the existing objects (source account, CloudShell).** Replication only copies *new* writes, so existing objects need a one-time **Batch Replication** job. The console wizard for this requires a separate IAM role and has many options. This script handles everything in one paste: it creates the role, attaches its policy, and starts the job.

```bash
BUCKET=YOUR-BUCKET
ACCT=SOURCE-ACCOUNT-ID
REGION=$(aws s3api get-bucket-location --bucket $BUCKET --query LocationConstraint --output text)
[ "$REGION" = "None" ] && REGION=us-east-1
echo "Region: $REGION"

# Confirm the replication rule exists (step 6b)
aws s3api get-bucket-replication --bucket $BUCKET --query 'ReplicationConfiguration.Rules[0].Destination.Bucket' --output text || { echo "No replication rule - stop here"; }

# Create or update the Batch Operations role
aws iam create-role --role-name s3-batch-replication --assume-role-policy-document '{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Allow","Principal":{"Service":"batchoperations.s3.amazonaws.com"},"Action":"sts:AssumeRole"}]
}' >/dev/null 2>&1 && echo "Role created" || echo "Role already exists"

aws iam put-role-policy --role-name s3-batch-replication --policy-name batch-replication --policy-document '{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["s3:InitiateReplication"],"Resource":"arn:aws:s3:::'$BUCKET'/*"},
    {"Effect":"Allow","Action":["s3:GetReplicationConfiguration","s3:PutInventoryConfiguration"],"Resource":"arn:aws:s3:::'$BUCKET'"}
  ]
}' && echo "Policy attached"

echo "Waiting for IAM to propagate..."; sleep 15

# Create and start the Batch Replication job
aws s3control create-job --region $REGION --account-id $ACCT \
  --operation '{"S3ReplicateObject":{}}' \
  --report '{"Enabled":false}' \
  --manifest-generator '{"S3JobManifestGenerator":{"SourceBucket":"arn:aws:s3:::'$BUCKET'","EnableManifestOutput":false,"Filter":{"EligibleForReplication":true,"ObjectReplicationStatuses":["NONE","FAILED"]}}}' \
  --priority 10 \
  --role-arn arn:aws:iam::$ACCT:role/s3-batch-replication \
  --no-confirmation-required
```

**Expected output:** the region, then the replica bucket's ARN, then "Role created" and "Policy attached," and finally a **JobId**. The JobId means the job is running. `--no-confirmation-required` skips the "Run job" click the console would otherwise need.

**Check progress:**

```bash
aws s3control list-jobs --region $REGION --account-id $ACCT --query 'Jobs[].{Status:Status,Done:ProgressSummary.NumberOfTasksSucceeded,Failed:ProgressSummary.NumberOfTasksFailed}'
```

The job shows `Active` with `Done: 0` while it builds its object list, which takes several minutes. Then the Done count climbs, and finally the status changes to `Complete`. `Failed` should stay at 0.

### 6d. Lock down the backup

1. **Lifecycle rule** on `YOUR-BUCKET-replica`: permanently delete **noncurrent versions after 180 days**, longer than the source's 90 days, so the backup always reaches further back. In CloudShell, **signed in to the backup account**:

```bash
cat > replica-lifecycle.json <<'EOF'
{"Rules":[{"ID":"replica-defaults","Status":"Enabled","Filter":{},
"NoncurrentVersionExpiration":{"NoncurrentDays":180},
"AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}}]}
EOF
aws s3api put-bucket-lifecycle-configuration --bucket YOUR-BUCKET-replica --lifecycle-configuration file://replica-lifecycle.json
```
2. **SCP (AWS Organizations or Control Tower):** deny destructive actions on the replica bucket for everyone except a break-glass role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectReplicaBucket",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObjectVersion",
        "s3:DeleteBucket",
        "s3:PutBucketPolicy",
        "s3:DeleteBucketPolicy",
        "s3:PutLifecycleConfiguration",
        "s3:PutBucketVersioning",
        "s3:PutReplicationConfiguration"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-replica",
        "arn:aws:s3:::YOUR-BUCKET-replica/*"
      ],
      "Condition": {
        "ArnNotLike": { "aws:PrincipalArn": "arn:aws:iam::BACKUP-ACCOUNT-ID:role/BreakGlassAdmin" }
      }
    }
  ]
}
```

Attach the SCP to the backup account's OU, or to the account directly. SCPs don't apply to the management account, which is one more reason the backup bucket should live in a member account.

3. **Optional:** turn on **Object Lock in governance mode** on the replica bucket only, with a 14–30 day default retention. It suits the backup well, because nobody edits files there interactively. Don't use compliance mode.

### 6e. Verify

1. Upload a test file through Mountain Duck.
2. In the source bucket, select the object → **Properties** → **Replication status** should change from `PENDING` to `COMPLETED` within a few minutes.
3. Confirm the file appears in `YOUR-BUCKET-replica`, with the storage class you chose.
4. Rerun the `list-jobs` command from 6c until the Batch Replication job shows `Complete` with `Failed: 0`. Then browse the replica bucket in the backup account and spot-check a few folders.

### Restoring from the backup

- **A few files:** sign in to the backup account, download from the replica bucket (use **Show versions** for older copies), then put them back in the source bucket.
- **Everything:** copy the replica back with rclone or `aws s3 sync` from a role in the backup account. Objects in Glacier Instant Retrieval can be read directly, with no restore request needed.

### Cost notes

- **Replica storage:** at the Glacier Instant Retrieval or Deep Archive rate, much cheaper than the source.
- **Replication requests:** a small per-object charge on each write, plus a one-time Batch Replication charge for existing objects.
- **Glacier Instant Retrieval** bills a minimum of 128 KB per object and 90 days of storage. Lots of tiny files cost a little more than their size suggests.

---

## Do's and don'ts

**Don't** point `~/Downloads`, `~/Desktop`, or `~/Documents` at the Mountain Duck folder, whether with symlinks or with an app's save location. File Provider folders don't reliably accept files written incrementally. That's exactly what breaks browser downloads when Downloads is symlinked into Dropbox. Keep those folders local and move files into S3 when you're done with them.

**Don't** keep live databases, app libraries, or files that sync tools write constantly in the bucket. Examples include Photos and Music libraries, Quicken files open in place, VM disks, git repositories, and `node_modules`. Object storage isn't a live filesystem, so copy these in as backups instead.

**Do** keep using **Time Machine** for the Mac itself. S3 stores your files; Time Machine backs up the machine.

**Do** let large uploads finish before closing the laptop lid. Check the menu bar icon.

---

## Cost notes

- **Storage:** Intelligent-Tiering moves cold files to cheaper tiers automatically. A small monitoring fee per object applies to objects larger than 128 KB.
- **Requests:** browsing many small files generates LIST and GET requests. They're cheap but not free, and noticeable only with very heavy use.
- **Data out:** downloading to your Mac costs standard S3 internet egress beyond the free allowance. Uploads are free.
- **Versioning:** old versions take up storage, which is why the lifecycle rule in Step 1 expires them.
- **What resets Intelligent-Tiering:** browsing folders and placeholders doesn't, because it reads only metadata. Opening a file, Quick Look, Finder or Explorer thumbnails, and Keep Offline do, and the file moves back to the Frequent tier for 30 days. Use **List view** for large photo or scan folders, and pin only the folders you work in. Even heavy accidental access costs cents.
- **Real-world example:** about 160 GB comes to roughly $3.70/month in Standard, falling toward $1.50–2/month as Intelligent-Tiering moves cold files down. A Glacier Instant Retrieval replica adds about $0.65/month.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Bucket doesn't appear in Finder | Turn on Mountain Duck's File Provider extension (Step 3), then disconnect and reconnect the bookmark. |
| "Access denied" on connect | Check that the key belongs to the right user and that the bucket name in the policy is exact. If the bookmark has no Path, the `ListAllMyBuckets` statement is required. |
| "Access denied" on upload or delete | Missing `s3:PutObject` / `s3:DeleteObject` on `YOUR-BUCKET/*` (note the `/*`). |
| Files show 0 bytes or won't open | Not downloaded yet. Open again, or use Keep Offline, and check the menu bar icon for errors. |
| Storage costs higher than expected | Confirm the Intelligent-Tiering lifecycle rule exists, and check noncurrent-version storage in **S3 Storage Lens**. |
| Lost computer or exposed key | IAM → user → deactivate and delete **that machine's** access key immediately, then create a new one. Other machines are unaffected. |
| Bucket missing from File Explorer (Windows) | Sign out and back in to Windows, or restart Mountain Duck. The Cloud Files sync root registers at login. |
| Edit on one machine not visible on the other | Refresh the folder. If it's still missing, check the uploading machine's icon for pending or failed uploads. |
| Replication status stays `FAILED` | The destination bucket policy is missing the replication role ARN or the owner-override permission, or versioning is off on the replica bucket. |
| Existing files not in the replica | The Batch Replication job wasn't created or didn't finish. Rerun the 6c script, which is safe to repeat because it only picks up objects that haven't replicated or failed. |
| Batch Operations wizard asks for an IAM role | Use the 6c script instead. It creates the role and the job together. |
| Batch job shows `Active` with `Done: 0` | Normal for the first several minutes while it builds the object list. |
| `AccessDenied` on `aws s3 rm` | You're using credentials without delete permission, such as the rclone instance role or the Mountain Duck key. Run admin cleanup from CloudShell. |
