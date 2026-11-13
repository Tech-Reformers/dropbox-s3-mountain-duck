# How to Sync Dropbox to Amazon S3 (rclone on EC2)

Copy an entire Dropbox account into an S3 bucket, cloud-to-cloud, without routing data through your own computer.

> **Note:** the tool is **rclone**, not rsync. rsync only works between filesystems; rclone talks directly to the Dropbox and S3 APIs.

---

## Why run it on EC2

Tools that mount cloud storage on your computer, such as Mountain Duck or the Dropbox desktop app, route every file through your machine: download, then upload. Running rclone on a small EC2 instance avoids that:

- **Fast:** data moves at datacenter speeds.
- **Cheap:** Dropbox doesn't charge for downloads, and EC2 to S3 traffic in the same region is free. You pay only for a few hours of EC2 time plus S3 storage.
- **Unattended:** your laptop doesn't need to stay awake or online.

## What you need

- An AWS account and an S3 bucket in the region you'll use. Set up versioning and the lifecycle rule first (Step 1 of the companion guide, **Replacing Dropbox with Amazon S3 + Mountain Duck**), so the migrated data is protected and tiered from day one.
- A computer with a browser and [Homebrew](https://brew.sh), or another way to install rclone, for the one-time Dropbox sign-in
- Your Dropbox login

Throughout this guide, replace **`YOUR-BUCKET`** and **`YOUR-REGION`** (for example, `us-west-2`) with your own values.

> **Tip:** If you run AWS Organizations or Control Tower, put the bucket in a **member account**, not the management account.

---

## Step 1 — Create an IAM role for the instance

1. IAM → Roles → **Create role**
2. Trusted entity: **AWS service**, use case **EC2**
3. Attach the managed policy **AmazonSSMManagedInstanceCore**. This lets you connect through Session Manager without SSH keys.
4. Name it `rclone-dropbox-s3` and create it.
5. Open the role → **Add permissions** → **Create inline policy** → JSON, and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::YOUR-BUCKET"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET/*"
    }
  ]
}
```

## Step 2 — Launch the EC2 instance

EC2 console, in the **same region as your bucket** → **Launch instance**:

| Setting | Value |
|---|---|
| AMI | Amazon Linux 2023, 64-bit (x86) |
| Instance type | A network-optimized type such as `m5n.large` (any recent `m`-class works) |
| Key pair | **Proceed without a key pair** |
| Network | Default VPC, public subnet, **auto-assign public IP: enabled** |
| Security group | No inbound rules needed |
| IAM instance profile | `rclone-dropbox-s3` (under Advanced details) |
| Storage | Default 8 GB is fine, because rclone streams data and doesn't stage files on disk |

Launch the instance and wait about 2 minutes. Then select it → **Connect** → **Session Manager** → **Connect**.

> **Instance size barely matters.** Dropbox's API rate limits set the pace, not the instance. Burstable `t`-class types work too, but their sustained network baseline is lower.

## Step 3 — Get a Dropbox token (on your own computer)

The instance has no browser, so do the Dropbox sign-in on your computer:

```bash
brew install rclone
rclone authorize "dropbox"
```

A browser opens. Sign in and approve access. rclone then prints a JSON token like `{"access_token":"...","token_type":"bearer",...}`. **Copy the whole JSON block.**

Your computer never connects to the instance. You'll paste this text into the instance's config file in the next step.

## Step 4 — Install and configure rclone (on the instance)

In the Session Manager shell, **run each command one at a time.** If you paste them as a block, the lines after `sudo su` get swallowed.

```bash
sudo su - ec2-user
```
```bash
sudo dnf install -y tmux
```
```bash
curl https://rclone.org/install.sh | sudo bash
```
```bash
mkdir -p ~/.config/rclone
```
```bash
nano ~/.config/rclone/rclone.conf
```

Paste the following, filling in your token and region. Save with **Ctrl-O**, then **Enter**, then exit with **Ctrl-X**.

```ini
[dropbox]
type = dropbox
token = PASTE_JSON_TOKEN_HERE

[s3]
type = s3
provider = AWS
env_auth = true
region = YOUR-REGION
storage_class = INTELLIGENT_TIERING
```

> **Storage class:** use `INTELLIGENT_TIERING` if you'll access files occasionally, or `GLACIER_IR` for a cold archive.

Lock down the file and test both remotes:

```bash
chmod 600 ~/.config/rclone/rclone.conf
rclone lsd dropbox:
rclone lsd s3:YOUR-BUCKET
```

Both commands should list folders, or nothing for an empty bucket, without errors. If S3 returns "access denied," check that the IAM role is attached: Actions → Security → Modify IAM role.

## Step 5 — Run the copy

Start a `tmux` session so the copy keeps running after you disconnect:

```bash
tmux new -s sync
```

A green status bar appears at the bottom. Then run:

```bash
rclone copy dropbox: s3:YOUR-BUCKET --size-only --tpslimit 12 --transfers 8 --checkers 16 --fast-list --progress --log-file ~/rclone.log --log-level INFO
```

This copies the root of your Dropbox to the root of the bucket. To put it under a prefix instead, use `s3:YOUR-BUCKET/dropbox`.

**Detach and leave it running:** press **Ctrl-B**, then **D**. You can now close the browser tab.

### What the flags do

| Flag | Purpose |
|---|---|
| `copy` | Adds and updates files in S3. **Never deletes** anything in the bucket. |
| `--size-only` | Skips files already in S3 at the same path and size. Needed when re-running, or when some files were uploaded earlier by another tool, because Dropbox and S3 share no hash type and modification times won't match. |
| `--tpslimit 12` | Stays under Dropbox's API rate limits. |
| `--transfers 8 --checkers 16` | Parallel uploads and comparisons. |
| `--fast-list` | Fewer API calls on large buckets, at the cost of more memory. |
| `--progress` | Live status display. |
| `--log-file` / `--log-level INFO` | Writes a record of every copied file. |

### Optional excludes

Skip Dropbox sync-conflict folders:
```bash
--exclude "*Selective Sync Conflict*/**"
```

### Should I do a dry run?

You can add `--dry-run` to preview the copy, but it has to list and compare every file, so it takes nearly as long as the real run. Because `copy` never deletes anything, it's usually fine to skip.

## Step 6 — Check progress later

Reconnect through Session Manager, then:

```bash
sudo su - ec2-user
tmux attach -t sync      # live progress (use `tmux ls` if the name differs)
tail -20 ~/rclone.log    # or just read the log
```

To understand what's being copied:

```bash
grep -c "Copied (new)" ~/rclone.log                  # files that were missing from S3
grep -c "Copied (replaced existing)" ~/rclone.log    # files overwritten (should be ~0)

# uploads grouped by top-level folder
grep "Copied (new)" ~/rclone.log | sed -E 's/.*INFO  : ([^/]+)\/.*/\1/' | sort | uniq -c | sort -rn
```

> **Skipped files don't appear in the log at INFO level.** rclone only logs skips at DEBUG, so a skip count of 0 is normal.

To leave a shell: type `exit`, or just close the tab. To leave tmux without stopping the copy: **Ctrl-B**, then **D**.

## Step 7 — Verify and clean up

When the copy finishes, verify that everything made it. **Replace `YOUR-BUCKET` with your bucket name.** With the placeholder left in, the check runs against a bucket that doesn't exist.

```bash
rclone check dropbox: s3:YOUR-BUCKET --size-only --one-way --tpslimit 4 --checkers 4 --progress -v
```

> **Always include `--tpslimit` on the check.** It calls the Dropbox API just like the copy, and without a throttle Dropbox starts rate-limiting you (see below).
>
> **Leave out `--fast-list` for the check.** On Dropbox it makes one huge recursive listing call that's all-or-nothing: no progress shows while it waits, and a rate limit can stall it for a long time. Without it, rclone lists folder by folder and progresses steadily.

### What you'll see while it runs

| Message | Meaning |
|---|---|
| `ERROR : No common hash found - not using a hash for checks` | **Expected and harmless.** Dropbox and S3 use different hash types. With `--size-only`, rclone compares paths and sizes instead. |
| `Checks: 6451 / 6451, 100%, Listed 13809` | **Normal.** Checks can only reach files that have been listed, so they stay at 100% of a growing total. Watch the **Listed** number. As long as it keeps climbing, it's working. |
| `NOTICE: Error. Too many requests or write operations. Trying again in 300 seconds.` | Dropbox is rate-limiting you. rclone retries on its own, but each wait is 5 minutes. Press Ctrl-C and rerun with a lower `--tpslimit`. |
| Listed count frozen for 5+ minutes | Probably in a rate-limit backoff, often a longer cooldown after an unthrottled run. Press Ctrl-C, wait 15–20 minutes if `-v` still shows "Too many requests," then rerun the command above without `--fast-list`. |

### Reading the result

The summary lines (`N differences found`, `N matching files`) print just *above* the final stats block, so scroll up to see them. A quick shortcut is to run this right after the check finishes:

```bash
echo $?     # 0 = no differences, 1 = differences found
```

For reference, a real run of about 36,000 files took roughly 55 minutes.

| Result | What to do |
|---|---|
| `0 differences found` (exit code `0`) | Done. Clean up (below). |
| `... not in s3:YOUR-BUCKET` | Files are missing. Rerun the Step 5 copy command; it only picks up what's missing. |
| `Sizes differ` | Usually Dropbox **Paper docs** (the exported file never matches its original size) or the `.dbx-vault` **Vault placeholder**. You can ignore these if they're the only differences. |

### Optional: byte-for-byte verification

`--download` reads both copies and compares their actual contents. It's slow (hours for tens of GB), but on the EC2 instance it costs next to nothing: Dropbox doesn't charge for downloads, and reading S3 within the same region is free.

```bash
rclone check dropbox: s3:YOUR-BUCKET --download --one-way --tpslimit 4 --checkers 4 --progress
```

### Clean up

When the check reports **0 differences**:

1. **Terminate** the EC2 instance.
2. **Revoke rclone's access** at dropbox.com → Settings → **Connected apps**.
3. Delete the IAM role if you won't reuse it.

### Removing unwanted files from the bucket afterward

The instance's role deliberately can't delete anything, so do cleanup from **CloudShell** in the console, which uses your console credentials. For example, to remove every file with a given extension anywhere in the bucket:

```bash
aws s3 rm s3://YOUR-BUCKET/ --recursive --exclude "*" --include "*.sookasa" --dryrun   # preview
aws s3 rm s3://YOUR-BUCKET/ --recursive --exclude "*" --include "*.sookasa"            # delete
```

With versioning on, these become delete markers, and the data is permanently removed when the lifecycle rule expires the noncurrent versions.

To check the bucket's total size:

```bash
aws s3 ls s3://YOUR-BUCKET --recursive --summarize --human-readable | tail -2
```

This counts current files only. The bucket's **Metrics** tab shows everything you're billed for, including old versions, and updates daily.

---

## Things rclone can't copy

Check these by hand before canceling Dropbox:

- **Dropbox Vault:** PIN-protected, so the API only sees a `.dbx-vault` placeholder. Download Vault contents manually from dropbox.com.
- **Dropbox Paper docs:** these are exported or skipped depending on rclone settings. Spot-check them in S3.
- **Files encrypted by third-party tools** (for example, `.sookasa` files): they copy byte-for-byte but stay encrypted. Make sure you can still decrypt them.
- **Shared folders and shared links:** anything shared with others, or sent as Dropbox links, stops working when you cancel.

## After the migration

- **Work with the files day to day:** see **Replacing Dropbox with Amazon S3 + Mountain Duck**.
- **Your Mac's Desktop, Documents, or Downloads may be Dropbox symlinks.** Run `find ~ -maxdepth 2 -type l -lname '*Dropbox*' -ls`. If any appear, replace them with real folders before uninstalling Dropbox (`sudo rm -v ~/Downloads && mkdir ~/Downloads`). Don't use a trailing slash or `-r`. Then reset each browser's download location. Otherwise browser downloads break.

## Troubleshooting

| Problem | Fix |
|---|---|
| Nothing happens after pasting commands | Run commands one at a time, especially after `sudo su - ec2-user`. |
| `dnf` or `curl` hangs | The instance has no internet access. Use a public subnet with a public IP, or add a NAT gateway. |
| Command looks frozen with `\| tail` | `tail` waits for the command to finish. Use `--progress` instead. |
| `rclone lsd s3:` access denied | The IAM role isn't attached to the instance, or the bucket name in the policy is wrong. |
| `rclone lsd dropbox:` fails | The token was pasted incorrectly. Re-run `rclone authorize "dropbox"` and paste the full JSON on one line. |
| Re-uploading files that are already in S3 | Use `--size-only`, and compare `rclone lsd dropbox:` with `rclone lsd s3:YOUR-BUCKET` to confirm the folder paths line up. |
| Instance type not offered | Arm (`g`) types need the 64-bit (Arm) AMI. Any x86 `m`- or `t`-class type also works. |
| "Too many requests... Trying again in 300 seconds" | Dropbox rate limit. Add or lower `--tpslimit` (try 8, then 4) and reduce `--checkers`. This applies to `check` as well as `copy`. |
| "No common hash found" | Expected between Dropbox and S3. Use `--size-only`, or `--download` for a content comparison. |
| Check seems stuck at 100% | It's still listing. Watch the **Listed** counter. If it's frozen, rerun without `--fast-list` (see Step 7). |
| `AccessDenied ... s3:DeleteObject` when cleaning up | You ran it on the EC2 instance, whose role is copy-only by design. Run deletes from CloudShell instead. |
