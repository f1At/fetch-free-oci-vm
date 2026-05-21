这是一份为你翻译好的英文版 Markdown 文件。你可以直接将以下内容保存为 `.md` 文件，发布到你的英文博客或 GitHub 仓库中：

---

# Oracle Cloud (OCI) ARM Free Tier (A1.Flex) Automated Provisioning Script & Ultimate Troubleshooting Guide

In Oracle Cloud Infrastructure (OCI), the 4-core, 24GB RAM, 100GB disk ARM-based free instance (VM.Standard.A1.Flex) offers incredibly high value and is consistently in an "Out of capacity" state across popular regions (such as Frankfurt `eu-frankfurt-1`, Tokyo, etc.). It is nearly impossible to provision one manually via the web console.

This article summarizes a fully automated provisioning solution based on the **OCI CLI**. It documents all the classic pitfalls (400/404/500 errors) you might encounter—from initial configuration to background execution—along with their ultimate solutions. It's perfectly suited for sharing on GitHub or your personal blog.

---

## Table of Contents

1. [Environment Preparation & CLI Installation](https://www.google.com/search?q=%231-environment-preparation--cli-installation)
2. [Core Parameters Guide](https://www.google.com/search?q=%232-core-parameters-guide)
3. [From 400 to 500: Troubleshooting & Pitfalls](https://www.google.com/search?q=%233-from-400-to-500-troubleshooting--pitfalls)
4. [Bulletproof Automated Provisioning Script (Final Version)](https://www.google.com/search?q=%234-bulletproof-automated-provisioning-script-final-version)
5. [Background Execution & Maintenance Commands](https://www.google.com/search?q=%235-background-execution--maintenance-commands)

---

## 1. Environment Preparation & CLI Installation

The official OCI CLI is the safest and most stable method for provisioning. Unlike third-party scripts, it does not rely on high-risk API proxies and communicates directly with Oracle's official gateways.

### 1.1 Install OCI CLI

Run the official one-click installation script on your Linux server (e.g., Ubuntu 22.04/24.04):

```bash
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

```

### 1.2 Basic Authentication Configuration

Once installed, run the following command to initialize the configuration:

```bash
oci setup config

```

* The system will prompt you for your User OCID, Tenancy OCID, and Region, and will guide you to generate an RSA key pair.
* ⚠️ **Core Pitfall**: The initialization process generates a private and public key (`oci_api_key_public.pem`) locally (usually in `~/.oci/`). You **must** log in to the Oracle Console -> click "User Settings" (top right corner) -> "API Keys" (bottom left) -> "Add API Key", and paste the contents of your local public key. Otherwise, all subsequent commands will fail authentication.

### 1.3 Verify Connectivity

After binding the key, run this command to test it. If it outputs global region JSON data, the OCI CLI is configured successfully:

```bash
oci iam region list

```

---

## 2. Core Parameters Guide

The script requires 5 core parameters. Because `Compartment ID`, `Subnet ID`, and `Image ID` are extremely long, ensure you copy them exactly and entirely.

### 2.1 Compartment ID

Usually, the root tenancy OCID is used. In the console, click your profile icon (top right) -> "Tenancy", and directly copy the string starting with `ocid1.tenancy.oc1..`.

### 2.2 Availability Domain (AD)

Every Oracle account is assigned a system-randomized exclusive prefix (usually 4 English letters). The complete AD format for the Frankfurt region looks like this (where `RupR` should be replaced with your specific prefix):

* `RupR:EU-FRANKFURT-1-AD-1`
* `RupR:EU-FRANKFURT-1-AD-2`
* `RupR:EU-FRANKFURT-1-AD-3`

**How to get it**: Go to the "Create Instance" page in the console, scroll down to the "Placement" section, and you will see the exact AD string exclusive to your account directly on the UI.

### 2.3 Subnet ID

Navigate to the main menu -> Networking -> Virtual Cloud Networks (VCN) -> click your VCN name -> select your **Public Subnet** -> go to the details page and copy the OCID starting with `ocid1.subnet.oc1..`.

### 2.4 ARM Image ID

To provision an A1 instance, you must use an OS image that supports the **ARM architecture (aarch64)**. We can use OCI CLI to accurately query the exact ID of the official Ubuntu 22.04 image in the Frankfurt region:

```bash
oci compute image list \
    --compartment-id "YOUR_TENANCY_ID" \
    --operating-system "Canonical Ubuntu" \
    --operating-system-version "22.04" \
    --shape "VM.Standard.A1.Flex" \
    --query "data[*].{Name:\"display-name\", ID:id}" \
    --output table

```

Copy the ID starting with `ocid1.image.oc1.eu-frankfurt-1..` from the output table.

---

## 3. From 400 to 500: Troubleshooting & Pitfalls

During the refinement of this script, almost everyone encounters the following series of classic errors. Here are their root causes and ultimate solutions:

### 3.1 Error Code 404 NotAuthorizedOrNotFound

* **Cause**: When copying long IDs via OCR or manual typing, missing or extra letters occur due to consecutive letters (like aaaaaaa); alternatively, the API public key was not uploaded properly to the console.
* **Solution**: Completely abandon manual typing or screenshot extraction. You must go to the console and click the "Copy" button to obtain the 100% complete native OCID.

### 3.2 Error Code 400 CannotParseRequest (Phase 1: CLI Quote Escaping)

* **Cause**: When providing `--shape-config '{"ocpus":4,"memoryInGBs":24}'` via CLI, different Linux Shell environments have strict and sometimes conflicting rules for single/double quote escaping for inline JSON, causing Oracle servers to fail in parsing the request.
* **Solution**: Write the JSON configuration directly into an external file and use the `file://` syntax. This entirely bypasses the command-line quotation nightmare.

### 3.3 Error Code 400 CannotParseRequest (Phase 2: Windows Hidden Newlines)

* **Cause**: Editing the script on a Windows PC and pasting it into an Ubuntu server. Windows uses `\r\n` for line breaks, while Linux uses `\n`. Bash reads the invisible `\r` as part of the ID, resulting in a corrupted payload like `ocid1.xxx...\r`.
* **Solution**: Run `sed -i 's/\r//' auto_a1.sh` in the Linux terminal to forcefully scrub all invisible carriage return characters.

### 3.4 Script Freezing / Stuck

* **Cause**: OCI CLI has a built-in Exponential Backoff retry mechanism. When hitting frequent 500 errors, the CLI automatically waits and retries (often getting stuck for 2-3 minutes), making the outer Bash script appear frozen.
* **Solution**: Explicitly append the `--no-retry` parameter to the launch command. This forces the CLI to give up immediately upon encountering an error, returning control of the polling rhythm fully to the Bash script.

### 3.5 Error Code 500 Server Error & Force Disconnect

* **Cause**: High-demand regions like Frankfurt experience massive automated traffic, causing Oracle's backend API gateways to intermittently crash or throttle for self-protection. **This is 100% not a script issue.**
* **Solution**: The script must include robust `else` fallback logic to gracefully catch this exception, sleep for a designated period, and safely resume later.

---

## 4. Bulletproof Automated Provisioning Script (Final Version)

This final version integrates all the bug-fixes discussed: it dynamically generates an absolute path JSON configuration, scrubs hidden characters, adds `--no-retry` for polling efficiency, and flawlessly catches 429 and 500 exceptions.

Create a file named `auto_a1.sh` on your server, paste the following content, and **fill in your actual parameters**:

```bash
#!/bin/bash
# =================================================================
# 🛠️ Configuration Section (Required)
# =================================================================
COMPARTMENT_ID="ocid1.tenancy.oc1..aaaaaaaaxxxxxxxxxxxxxxxxxxxxxxxx"
SUBNET_ID="ocid1.subnet.oc1.eu-frankfurt-1.aaaaaaaaxxxxxxxxxxxxxxxxxxxxxxxx"
IMAGE_ID="ocid1.image.oc1.eu-frankfurt-1.aaaaaaaaxxxxxxxxxxxxxxxxxxxxxxxx"
AD="RupR:EU-FRANKFURT-1-AD-1"   # Remember to update your exclusive AD prefix
SSH_KEY_FILE="/home/ubuntu/.ssh/id_rsa.pub"   # SSH public key path for current user

# =================================================================
# Polling interval in seconds. Recommended: 60s. NEVER set below 30s to avoid account bans!
SLEEP_TIME=60

# Core Security Fix: Forcefully generate JSON config via absolute path to prevent 400 errors
JSON_FILE="/tmp/shape_config.json"
echo '{"ocpus": 4, "memoryInGBs": 24}' > "$JSON_FILE"

echo "========================================================="
echo " 🚀 Starting OCI A1 Auto-Provisioning (Frankfurt | 4-Core 24G 100G) "
echo "========================================================="

while true; do
    # Execute the launch command, --no-retry ensures it returns immediately upon server congestion
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

    # 1. Success Catch
    if echo "$RESULT" | grep -q "PROVISIONING"; then
        echo "========================================="
        echo "$(date '+%Y-%m-%d %H:%M:%S') - 🎉 SUCCESS! A1 Instance Provisioned! Machine is creating..."
        echo "========================================="
        break
    
    # 2. Normal Out of Capacity Catch
    elif echo "$RESULT" | grep -q "Out of host capacity"; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - ⏳ Out of capacity, retrying in ${SLEEP_TIME} seconds..."
        
    # 3. Rate Limit Catch
    elif echo "$RESULT" | grep -q "TooManyRequests"; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - ⚠️ API Rate Limit Triggered! Pausing for 120s for safety..."
        sleep 120
        
    # 4. Parameter / Auth Error Catch
    elif echo "$RESULT" | grep -q "NotAuthorizedOrNotFound"; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - ❌ CRITICAL ERROR: Auth failed or Resource not found. Check API keys and OCIDs."
        break
        
    # 5. Oracle Server 500 Error / Congestion Catch
    else
        echo "$(date '+%Y-%m-%d %H:%M:%S') - 🌀 Oracle Server 500 Error/Congestion encountered. Running stably in background..."
        echo "Automatically launching a new wave in ${SLEEP_TIME} seconds..."
    fi
    
    sleep $SLEEP_TIME
done

```

---

## 5. Background Execution & Maintenance Commands

### 5.1 Clean Hidden Returns & Grant Permissions (Important)

After saving the script, run this line in your terminal to format it and grant execution rights:

```bash
sed -i 's/\r//' auto_a1.sh
chmod +x auto_a1.sh

```

### 5.2 Start Persistent Background Execution

Use `nohup` to run the script. This ensures the provisioning loop keeps running perfectly in the Ubuntu background even if you disconnect your SSH session or shut down your local computer:

```bash
nohup ./auto_a1.sh > a1_log.txt 2>&1 &

```

### 5.3 Check Status in Real-Time

To spot-check the provisioning progress, run the following command to view the last 20 lines of the log:

```bash
tail -n 20 -f a1_log.txt

```

> (Press `Ctrl + C` anytime to exit the log view. This will not affect the background script at all.)

### 5.4 Cleanup Installation Leftovers

Once the OCI CLI is running stably, the downloaded zip and setup scripts are no longer needed. Use these Linux commands to free up space:

```bash
# Optional: Copy script to a backup directory
cp auto_a1.sh /home/ubuntu/backup/

# Safely remove the zip file and installation script
rm -f oci-cli-*.zip install.sh

```

---

**Conclusion & Strategy**: Maintaining the `SLEEP_TIME` at a conservative **60 seconds** is the optimal strategy for long-term execution without triggering Oracle's risk control algorithms. Seeing logs stably output `⏳ Out of capacity` or `🌀 Oracle Server 500 Error/Congestion` means your script is safely communicating with the official datacenters. Good luck, and may you receive the "🎉 SUCCESS!" notification soon!
