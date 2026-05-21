
---

# Oracle Cloud (OCI) ARM Free Tier Auto-Provisioning Script & Troubleshooting Guide

This guide provides a fully automated provisioning solution for OCI ARM instances (4-core, 24GB RAM) using the official OCI CLI, including core configurations, troubleshooting, and background execution.

---

## 1. Environment Setup & OCI CLI Installation

```bash
# Install the official OCI CLI
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

# Initialize configuration (Enter your User OCID, Tenancy OCID, Region, and generate keys)
oci setup config

```

*⚠️ **Note**: After generating the key pair, you must copy the contents of your local public key (`~/.oci/oci_api_key_public.pem`) and add it to the OCI Console -> User Settings (top right) -> API Keys (bottom left).*

```bash
# Verify connectivity (Should return a JSON list of regions)
oci iam region list

```

---

## 2. Retrieve Core Parameters

* **Compartment ID**: Profile (top right) -> Tenancy -> Copy `ocid1.tenancy...`
* **Availability Domain (AD)**: Check the "Placement" section on the "Create Instance" page to find your exclusive AD string (e.g., `RupR:EU-FRANKFURT-1-AD-1`).
* **Subnet ID**: Networking -> Virtual Cloud Networks -> Click your VCN -> Public Subnet -> Copy `ocid1.subnet...`
* **Image ID**: Run the following command to query the exact ID of the official Ubuntu 22.04 ARM image:
```bash
oci compute image list --compartment-id "YOUR_TENANCY_ID" --operating-system "Canonical Ubuntu" --operating-system-version "22.04" --shape "VM.Standard.A1.Flex" --query "data[*].{Name:\"display-name\", ID:id}" --output table

```



---

## 3. Key Pitfalls & Solutions

* **404 NotAuthorizedOrNotFound**: Incomplete/typo in OCIDs, or the API public key was not bound successfully in the console.
* **400 CannotParseRequest (Quote Escaping)**: Inline JSON configuration often fails due to shell syntax. *Solution: Write the configuration into an external JSON file and pass it via `file://`.*
* **400 CannotParseRequest (Windows Newlines)**: Editing the script on Windows inserts hidden `\r` characters. *Solution: Run `sed -i 's/\r//' auto_a1.sh` to format it.*
* **Script Hanging / Freezing**: OCI CLI automatically retries on 500 errors. *Solution: Add the `--no-retry` parameter to return control immediately to the Bash script.*

---

## 4. Auto-Provisioning Script (`auto_a1.sh`)

```bash
#!/bin/bash
# =================================================================
# 🛠️ Configuration Section
# =================================================================
COMPARTMENT_ID="YOUR_COMPARTMENT_ID"
SUBNET_ID="YOUR_SUBNET_ID"
IMAGE_ID="YOUR_IMAGE_ID"
AD="YOUR_EXCLUSIVE_AD"
SSH_KEY_FILE="/home/ubuntu/.ssh/id_rsa.pub"
SLEEP_TIME=60   # Polling interval in seconds (Keep >= 30s to avoid bans)

# Avoid inline JSON escaping errors
JSON_FILE="/tmp/shape_config.json"
echo '{"ocpus": 4, "memoryInGBs": 24}' > "$JSON_FILE"

echo "========================================================="
echo " 🚀 Starting OCI A1 Auto-Provisioning (4-Core 24G 100G) "
echo "========================================================="

while true; do
    RESULT=$(oci compute instance launch \
        --compartment-id "$COMPARTMENT_ID" \
        --availability-domain "$AD" \
        --shape "VM.Standard.A1.Flex" \
        --shape-config "file://$JSON_FILE" \
        --subnet-id "$SUBNET_ID" \
        --image-id "$IMAGE_ID" \
        --ssh-authorized-keys-file "$SSH_KEY_FILE" \
        --boot-volume-size-in-gbs 100 \
        --assign-public-ip true \
        --no-retry \
        2>&1)

    if echo "$RESULT" | grep -q "PROVISIONING"; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - 🎉 SUCCESS! Instance is being created!" && break
    elif echo "$RESULT" | grep -q "Out of host capacity"; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - ⏳ Out of capacity, retrying in ${SLEEP_TIME}s..."
    elif echo "$RESULT" | grep -q "TooManyRequests"; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - ⚠️ Rate limit hit! Pausing for 120s..." && sleep 120
    elif echo "$RESULT" | grep -q "NotAuthorizedOrNotFound"; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - ❌ CRITICAL: Check your IDs or API key configuration!" && break
    else
        echo "$(date '+%Y-%m-%d %H:%M:%S') - 🌀 Server congested / temporary disconnect. Retrying..."
    fi
    
    sleep $SLEEP_TIME
done

```

---

## 5. Background Execution & Maintenance

```bash
# 1. Clean hidden Windows carriage returns and grant execution permissions
sed -i 's/\r//' auto_a1.sh
chmod +x auto_a1.sh

# 2. Start persistent background execution (keeps running after closing SSH)
nohup ./auto_a1.sh > a1_log.txt 2>&1 &

# 3. View the provisioning log in real-time
tail -n 20 -f a1_log.txt

# 4. Clean up installation leftovers
rm -f oci-cli-*.zip install.sh

```
