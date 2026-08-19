
Here is your fully integrated, step-by-step execution guide. Every phase now includes the exact commands you need to run, explicitly stating *where* to run them (in Cloud Shell vs. inside a specific VM).

This version drops Nmap, targets a single host, includes the 1-hour fail-safe, extracts clean XML, and walks you entirely through the Greenbone Docker deployment.

---

# Architecture & Execution Guide: Automated OpenVAS Single-Host Investigations

## Overview

This architecture automates OpenVAS/GVM vulnerability investigations for a specific, pre-discovered host inside a private GCP network. The orchestrator VM runs inside the VPC to reach private addresses. Results are extracted automatically and uploaded to Google Cloud Storage (GCS) using a Hive-partitioned structure. Network discovery is handled externally.

---

## Phase 1: Deploy & Configure the OpenVAS VM

The OpenVAS VM is a persistent machine that stores the vulnerability database and runs the Greenbone container stack.

### 1. Create the VM (Run in Cloud Shell)

```bash
gcloud compute instances create openvas-gvm-vm \
  --zone=us-central1-a \
  --machine-type=e2-standard-4 \
  --subnet=private-subnet \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=100GB \
  --tags=openvas-gvm

```

### 2. Connect to the VM (Run in Cloud Shell)

Wait a minute for the VM to boot, then connect to it:

```bash
gcloud compute ssh openvas-gvm-vm \
  --zone=us-central1-a \
  --tunnel-through-iap

```

### 3. Install Docker (Run inside `openvas-gvm-vm`)

```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-v2 curl nano
sudo systemctl enable --now docker
sudo usermod -aG docker $USER

```

*Note: Type `exit` to log out, then run the `gcloud compute ssh` command from Step 2 again to log back in so the Docker permissions take effect.*

### 4. Download and Configure Greenbone (Run inside `openvas-gvm-vm`)

Download the official compose file:

```bash
mkdir -p ~/greenbone-community
cd ~/greenbone-community
curl -f -L https://greenbone.github.io/docs/latest/_downloads/docker-compose-22.4.yml -o docker-compose.yml

```

**Expose the GMP API Port:**
Open the file to edit it:

```bash
nano docker-compose.yml

```

Scroll down to the `gvmd:` service section and add the `ports:` mapping so it looks exactly like this:

```yaml
  gvmd:
    image: greenbone/gvmd:stable
    restart: on-failure
    ports:
      - "9390:9390"
    volumes:
      - gvmd_data_vol:/var/lib/gvm

```

*Save the file (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).*

### 5. Start Greenbone and Sync Feeds (Run inside `openvas-gvm-vm`)

Boot the scanner in the background:

```bash
docker compose -p greenbone-community-edition up -d

```

The scanner must download gigabytes of vulnerability feeds before it will work. Watch the progress logs:

```bash
docker compose -p greenbone-community-edition logs -f notus-data vulnerability-tests scap-data cert-data

```

*Wait until the logs stop heavily scrolling (can take 30-60 mins). Press `Ctrl+C` to exit the logs.*

### 6. Set the OpenVAS Password (Run inside `openvas-gvm-vm`)

Set the password your orchestrator script will use to connect. Replace `YOUR_SECURE_PASSWORD` with a strong password.

```bash
docker compose -p greenbone-community-edition exec -u gvmd gvmd gvmd --user=admin --new-password=YOUR_SECURE_PASSWORD

```

*Type `exit` to return to your Google Cloud Shell.*

---

## Phase 2: Create the Orchestrator IAM Service Account

This account ensures the orchestrator VM only has permission to upload to your specific GCS bucket.

### 1. Create the Service Account (Run in Cloud Shell)

```bash
gcloud iam service-accounts create orchestrator-sa \
  --display-name="OpenVAS Orchestrator Service Account"

```

### 2. Grant Bucket Permissions (Run in Cloud Shell)

```bash
gcloud storage buckets add-iam-policy-binding gs://project_trivy1 \
  --member="serviceAccount:orchestrator-sa@project-trivy-503813.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

```

---

## Phase 3: Create the Orchestrator Startup Script

This script tells the orchestrator VM what to do every time it boots.

### 1. Save the Script (Run in Cloud Shell)

Copy and paste this entire block into Cloud Shell. Hit enter to create the `scan-single-host.sh` file.

```bash
cat << 'EOF' > scan-single-host.sh
#!/bin/bash
set -euo pipefail

apt-get update
apt-get install -y curl python3-pip python3-gvm
python3 -m pip install --break-system-packages gvm-tools || true

TARGET_IP="10.0.1.8" # Update this to your desired target host
BUCKET_NAME="project_trivy_1"
OPENVAS_HOST="openvas-gvm-vm"
OPENVAS_PORT="9390"
OPENVAS_USER="admin"
OPENVAS_PASSWORD_FILE="/etc/openvas/openvas-password"

YEAR=$(date -u +%Y)
MONTH=$(date -u +%m)
DAY=$(date -u +%d)
RUN_ID=$(date -u +%s)

OPENVAS_DIR="/scan-results/openvas"
mkdir -p "$OPENVAS_DIR"

OPENVAS_PASSWORD=$(cat "$OPENVAS_PASSWORD_FILE")

echo "Creating OpenVAS target for host $TARGET_IP..."
TARGET_ID=$(gvm-cli --gmp-username "$OPENVAS_USER" --gmp-password "$OPENVAS_PASSWORD" \
  tls --hostname "$OPENVAS_HOST" --port "$OPENVAS_PORT" \
  --xml "<create_target><name>single-host-$RUN_ID</name><hosts>$TARGET_IP</hosts></create_target>" \
  | grep -oE 'id="[a-f0-9-]+"' | head -1 | cut -d'"' -f2)

echo "$TARGET_ID" > "$OPENVAS_DIR/openvas-target-id.txt"

echo "Creating OpenVAS task..."
TASK_ID=$(gvm-cli --gmp-username "$OPENVAS_USER" --gmp-password "$OPENVAS_PASSWORD" \
  tls --hostname "$OPENVAS_HOST" --port "$OPENVAS_PORT" \
  --xml "<create_task><name>single-host-scan-$RUN_ID</name><target id=\"$TARGET_ID\"/><scanner id=\"08b69003-5fc2-4037-a479-93b440211c73\"/><config id=\"daba56c8-73ec-11df-a475-002264764cea\"/></create_task>" \
  | grep -oE 'id="[a-f0-9-]+"' | head -1 | cut -d'"' -f2)

echo "$TASK_ID" > "$OPENVAS_DIR/openvas-task-id.txt"

echo "Starting OpenVAS task..."
gvm-cli --gmp-username "$OPENVAS_USER" --gmp-password "$OPENVAS_PASSWORD" \
  tls --hostname "$OPENVAS_HOST" --port "$OPENVAS_PORT" \
  --xml "<start_task task_id=\"$TASK_ID\"/>"

echo "Polling OpenVAS task status..."
MAX_RETRIES=60  # 60 checks * 60 seconds = 1 Hour
COUNTER=0
STATUS=""

while [ $COUNTER -lt $MAX_RETRIES ]; do
  STATUS=$(gvm-cli --gmp-username "$OPENVAS_USER" --gmp-password "$OPENVAS_PASSWORD" \
    tls --hostname "$OPENVAS_HOST" --port "$OPENVAS_PORT" \
    --xml "<get_tasks task_id=\"$TASK_ID\"/>" \
    | grep -oE '<status>[^<]+' | head -1 | sed 's/<status>//' || true)

  echo "OpenVAS task status ($COUNTER/$MAX_RETRIES mins): $STATUS"

  if [ "$STATUS" = "Done" ]; then
    break
  fi

  COUNTER=$((COUNTER + 1))
  sleep 60
done

if [ "$STATUS" != "Done" ]; then
  echo "WARNING: OpenVAS scan did not complete cleanly (Status: $STATUS)."
  echo "Scan failed or timed out at $(date)" > "$OPENVAS_DIR/openvas-timeout-error.txt"
  gcloud storage cp -r "$OPENVAS_DIR"/* "gs://$BUCKET_NAME/year=$YEAR/month=$MONTH/day=$DAY/tool=openvas/run_id=$RUN_ID/"
  poweroff
  exit 1
fi

REPORT_ID=$(gvm-cli --gmp-username "$OPENVAS_USER" --gmp-password "$OPENVAS_PASSWORD" \
  tls --hostname "$OPENVAS_HOST" --port "$OPENVAS_PORT" \
  --xml "<get_tasks task_id=\"$TASK_ID\" details=\"1\"/>" \
  | grep -oE 'report id="[a-f0-9-]+"' | head -1 | cut -d'"' -f2)

echo "$REPORT_ID" > "$OPENVAS_DIR/openvas-report-id.txt"

echo "Exporting OpenVAS XML report..."
gvm-cli --gmp-username "$OPENVAS_USER" --gmp-password "$OPENVAS_PASSWORD" \
  tls --hostname "$OPENVAS_HOST" --port "$OPENVAS_PORT" \
  --xml "<get_reports report_id=\"$REPORT_ID\" format_id=\"a994b278-1f62-11e1-96ac-406186ea4fc5\" details=\"1\"/>" \
  > "$OPENVAS_DIR/raw-response.xml"

# Extract inner <report> payload
python3 -c "
import xml.etree.ElementTree as ET
try:
    tree = ET.parse('$OPENVAS_DIR/raw-response.xml')
    root = tree.getroot()
    report = root.find('.//report')
    if report is not None:
        with open('$OPENVAS_DIR/openvas-report.xml', 'w') as f:
            f.write(ET.tostring(report, encoding='utf-8').decode('utf-8'))
        print('Clean XML report extracted successfully.')
    else:
        print('Could not find <report> node, keeping raw file.')
except Exception as e:
    print(f'XML parsing error: {e}')
"

rm -f "$OPENVAS_DIR/raw-response.xml"

echo "Uploading results to GCS..."
gcloud storage cp -r "$OPENVAS_DIR"/* "gs://$BUCKET_NAME/year=$YEAR/month=$MONTH/day=$DAY/tool=openvas/run_id=$RUN_ID/"

rm -rf /scan-results
poweroff
EOF

```

---

## Phase 4: Provision the Orchestrator VM & Save Credentials

The orchestrator VM will try to run the script above as soon as it's created, but it will fail the very first time because the password file (`/etc/openvas/openvas-password`) doesn't exist yet. We will create the VM, let it fail safely, save the password, and power it off.

### 1. Create the VM (Run in Cloud Shell)

```bash
gcloud compute instances create internal-openvas-orchestrator-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --subnet=private-subnet \
  --service-account="orchestrator-sa@project-trivy-503813.iam.gserviceaccount.com" \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=internal-scanner \
  --metadata-from-file=startup-script=scan-single-host.sh

```

### 2. Save the Password (Run in Cloud Shell)

Run this command to remotely inject your OpenVAS password into the required file on the Orchestrator VM. (Replace `YOUR_SECURE_PASSWORD` with the one you set in Phase 1).

```bash
gcloud compute ssh internal-openvas-orchestrator-vm \
  --zone=us-central1-a \
  --tunnel-through-iap \
  --command="sudo mkdir -p /etc/openvas && echo 'YOUR_SECURE_PASSWORD' | sudo tee /etc/openvas/openvas-password > /dev/null && sudo chmod 600 /etc/openvas/openvas-password"

```

### 3. Turn the VM off (Run in Cloud Shell)

The VM is now ready for future automated runs. Turn it off so you aren't billed for it:

```bash
gcloud compute instances stop internal-openvas-orchestrator-vm --zone=us-central1-a

```

---

## Phase 5: Automate the Schedule

Tell Google Cloud Scheduler to turn the Orchestrator VM *on* every Monday at 5:00 AM. (The VM will run its script and automatically turn itself back off).

### 1. Create Scheduler Service Account (Run in Cloud Shell)

```bash
gcloud iam service-accounts create vm-scheduler-sa --display-name="VM Scheduler Account"

```

### 2. Grant Permission to Start VMs (Run in Cloud Shell)

```bash
gcloud projects add-iam-policy-binding project-trivy-503813 \
  --member="serviceAccount:vm-scheduler-sa@project-trivy-503813.iam.gserviceaccount.com" \
  --role="roles/compute.instanceAdmin.v1"

```

### 3. Create the Scheduled Job (Run in Cloud Shell)

```bash
gcloud scheduler jobs create http weekly-openvas-investigation \
  --location=us-central1 \
  --schedule="0 5 * * 1" \
  --time-zone="UTC" \
  --uri="https://compute.googleapis.com/compute/v1/projects/project-trivy-503813/zones/us-central1-a/instances/internal-openvas-orchestrator-vm/start" \
  --http-method=POST \
  --oauth-service-account-email="vm-scheduler-sa@project-trivy-503813.iam.gserviceaccount.com"

```

---

## Phase 6: Verification & Testing

### 1. Run the Scan Manually (Run in Cloud Shell)

Trigger the scheduler immediately to test the entire pipeline:

```bash
gcloud scheduler jobs run weekly-openvas-investigation --location=us-central1

```

### 2. Monitor the Orchestrator (Run in Cloud Shell)

Watch the Orchestrator VM's logs to see the script running:

```bash
gcloud compute instances get-serial-port-output internal-openvas-orchestrator-vm \
  --zone=us-central1-a | grep -i "startup-script"

```

### 3. Check for the Output File

Once the script says `Uploading results to GCS` and the VM shuts down, verify the file is in your bucket:

```bash
gcloud storage ls gs://project_trivy1/year=$(date -u +%Y)/month=$(date -u +%m)/day=$(date -u +%d)/tool=openvas/run_id=*/

```