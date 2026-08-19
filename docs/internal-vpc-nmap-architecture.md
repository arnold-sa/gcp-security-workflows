# Architecture Document: Automated Internal VPC Scanning (GCP Native)

## Overview

This architecture deploys a standalone, scheduled Virtual Machine within a private Google Cloud VPC. The system lies dormant to avoid costs, wakes up automatically on a cron schedule, performs an Nmap security sweep of a target subnet, uploads the results directly to Google Cloud Storage (GCS) using Hive partitioning, and instantly powers itself off.

## Components Used

- **Compute Engine:** Executes the Nmap scan inside the VPC.
- **Startup Scripts:** Handles the runtime logic, including installing tools, scanning, and uploading on boot.
- **Cloud Storage:** Long-term, durable storage for audit logs.
- **Cloud Scheduler:** Triggers the Compute Engine API every Monday at 4:00 AM.

## Phase 1: Create the Automation Payload

This script tells the VM exactly what to do when it boots up. It generates a Unix timestamp to serve as the unique `run_id` for your bucket path and includes a flag to log the scan's progress every 30 seconds.

Open Cloud Shell. Create a file named `scan-script.sh` by pasting this block:

```bash
cat << 'EOF' > scan-script.sh
#!/bin/bash
# 1. Update and install required packages
apt-get update
apt-get install -y nmap curl

# 2. Define target parameters
TARGET_CIDR="Target CIDR"
BUCKET_NAME="bucket_name"

# 3. Calculate time for GCS Hive Partitioning
YEAR=$(date -u +%Y)
MONTH=$(date -u +%m)
DAY=$(date -u +%d)
RUN_ID=$(date -u +%s)

# 4. Prepare workspace
mkdir -p /nmap-results
echo $TARGET_CIDR > /nmap-results/targets.txt

# 5. Execute Nmap scan with progress updates every 30 seconds
nmap -Pn -sV -T4 --stats-every 30s -iL /nmap-results/targets.txt -oX /nmap-results/nmap.xml -oN /nmap-results/nmap.txt

# 6. Upload results to Cloud Storage
gcloud storage cp /nmap-results/* gs://$BUCKET_NAME/year=$YEAR/month=$MONTH/day=$DAY/tool=nmap/run_id=$RUN_ID/

# 7. Clean up locally
rm -rf /nmap-results

# 8. Shut down the scanner VM
poweroff
EOF
```

## Phase 2: Provision the Scanner Instance

Next, create the Virtual Machine and permanently attach the startup script. The `--scopes=storage-rw,compute-rw` flags give the VM authorization to write to your GCS bucket and manage its own power state without hardcoded credentials.

Run this command in Cloud Shell:

```bash
gcloud compute instances create internal-scanner-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --subnet=private-subnet \
  --scopes=storage-rw,compute-rw \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=internal-scanner \
  --metadata-from-file=startup-script=scan-script.sh
```

Once this command completes, the VM will boot up, run its first scan, upload the results to your bucket, and then shut itself down.

## Phase 3: Automate the Schedule

To make the VM wake up every Monday at 4:00 AM, configure Cloud Scheduler to call the Compute Engine API.

### 1. Create a Service Account for the Scheduler

The scheduler needs permission to start the VM.

```bash
gcloud iam service-accounts create vm-scheduler-sa --display-name="VM Scheduler Account"

gcloud projects add-iam-policy-binding project-trivy-503813 \
  --member="serviceAccount:vm-scheduler-sa@project-trivy-503813.iam.gserviceaccount.com" \
  --role="roles/compute.instanceAdmin.v1"
```

### 2. Create the Cron Job

This job targets the Google Cloud API directly and uses the scheduler service account as the auth identity to start the scanner VM.

```bash
gcloud scheduler jobs create http weekly-nmap-scan \
  --location=us-central1 \
  --schedule="0 4 * * 1" \
  --time-zone="UTC" \
  --uri="https://compute.googleapis.com/compute/v1/projects/project-trivy-503813/zones/us-central1-a/instances/internal-scanner-vm/start" \
  --http-method=POST \
  --oauth-service-account-email="vm-scheduler-sa@project-trivy-503813.iam.gserviceaccount.com"
```

## Phase 4: Verification and Operations

To test the schedule manually:

```bash
gcloud scheduler jobs run weekly-nmap-scan --location=us-central1
```

To check the results, go to the Google Cloud Console, navigate to Cloud Storage, and open `project_trivy1`. You should see a nested structure similar to:

```text
year=2026/month=08/day=14/tool=nmap/run_id=1723554000/
```

Because the script executes `poweroff` as its final command, the VM transitions to the `TERMINATED` state after the scan finishes. You only pay for the compute minutes used by the scan.

## Phase 5: Monitoring and Logging

Because the VM runs without human intervention, monitor the startup script execution to watch the Nmap scan and GCS upload in progress. The `--stats-every 30s` flag makes Nmap print ETA and completion progress every 30 seconds.

### Method 1: Stream Logs Live

SSH into the machine and tail the system logs:

```bash
gcloud compute ssh internal-scanner-vm --zone=us-central1-a
sudo journalctl -u google-startup-scripts.service -f
```

Because the final command is `poweroff`, your SSH session will disconnect when the job succeeds.

### Method 2: Check Logs from Cloud Shell

If you do not want to SSH into the machine, pull a snapshot of the serial-port logs directly from Cloud Shell. The `grep` filter reduces noisy boot logs and shows startup-script lines, including the 30-second Nmap progress updates.

```bash
gcloud compute instances get-serial-port-output internal-scanner-vm \
  --zone=us-central1-a | grep -i "startup-script"
```
