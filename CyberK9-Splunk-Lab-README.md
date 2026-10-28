# CyberK9 Splunk Lab

**A hands-on SIEM learning environment using Windows 11, Oracle VirtualBox, Ubuntu Server 24.04 LTS, and Splunk Enterprise.**

> **Project status:** Lab deployed; Splunk Web accessible from the Windows host; initial internal-log search and visualization completed.
>
> **Purpose:** Personal cybersecurity learning and practical experience with virtualization, Linux administration, SSH, log analysis, and Splunk Search Processing Language (SPL).

## Architecture

```text
Windows 11 host (Chrome/Edge + PowerShell)
             |
             | VirtualBox NAT port forwarding
             | 127.0.0.1:2222 -> VM:22 (SSH)
             | 127.0.0.1:8000 -> VM:8000 (Splunk Web)
             v
Oracle VirtualBox: CyberK9-Splunk-Lab
  Ubuntu Server 24.04 LTS (VM NAT address observed: 10.0.2.15)
    - OpenSSH Server
    - Splunk Enterprise
    - Splunk internal logs (_internal index)
```

**Note:** `10.0.2.15` is the address observed in this particular NAT VM; it is not a permanent address or a Windows browser URL. `127.0.0.1` refers to the Windows host in the browser and PowerShell commands below.

## 1. Prerequisites

- Windows 11 computer with hardware virtualization enabled.
- Oracle VirtualBox installed.
- Ubuntu Server **24.04 LTS** installation ISO (this lab was observed running Ubuntu 24.04.5 LTS after updates).
- Internet connection for Ubuntu package updates and Splunk download.
- Splunk account/download access and sufficient free disk space. This lab showed a roughly 98 GB Ubuntu root filesystem; your allocation may differ.
- Administrator access to the Windows host and sudo access within Ubuntu.

**Lab resource starting point (adjust to your computer):** 2–4 vCPUs, 4–8 GB RAM, and 80–100 GB virtual disk. These are suggested lab settings, **not verified measurements of the original VM** and not a substitute for Splunk's current published requirements.

## 2. Create the Ubuntu virtual machine

1. Download the Ubuntu Server 24.04 LTS ISO from the [official Ubuntu site](https://ubuntu.com/download/server).
2. Open VirtualBox and select **New**.
3. Name the VM `CyberK9-Splunk-Lab` and select the Ubuntu ISO.
4. Choose Linux / Ubuntu (64-bit) where prompted.
5. Allocate CPU, memory, and a dynamically allocated virtual disk appropriate to your system.
6. Ensure **Network > Adapter 1 > Attached to: NAT**.
7. Start the VM and follow the Ubuntu Server installer. Create a non-root administrator account; this lab uses `k9admin` as its example username.
8. If offered, select **Install OpenSSH server**. Complete installation and reboot.
9. Log in and update Ubuntu:

```bash
sudo apt update
sudo apt upgrade -y
```

Do **not** upgrade to a new Ubuntu release just because a newer release is offered; first confirm Splunk compatibility and back up the VM.

## 3. Enable SSH

In Ubuntu, install and start OpenSSH if it was not installed during setup:

```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

Check that the VM is on a NAT network:

```bash
ip -4 addr
```

In VirtualBox, open the VM's **Settings > Network > Adapter 1 > Advanced > Port Forwarding** (menu placement varies by version). Add these two **TCP** rules:

| Name | Host IP | Host port | Guest IP | Guest port | Purpose |
|---|---|---:|---|---:|---|
| SSH | `127.0.0.1` | `2222` | `10.0.2.15` | `22` | SSH from Windows |
| SplunkWeb | `127.0.0.1` | `8000` | `10.0.2.15` | `8000` | Splunk Web from Windows |

**Important:** The SSH rule above was verified in the lab. The Splunk Web rule reflects the observed working host browser address, but the exact rule settings were not captured. Verify or create it if needed. If the guest's NAT IP changes, update the Guest IP or leave that field blank where VirtualBox supports it. Keep **Host IP = `127.0.0.1`** to avoid unintentionally exposing these ports to your network.

From **Windows PowerShell** (not the Ubuntu VM console), connect:

```powershell
ssh -p 2222 k9admin@127.0.0.1
```

Accept the SSH host fingerprint only after verifying it belongs to your VM, then enter your Ubuntu account password. You can now paste commands into your PowerShell SSH session, even if clipboard integration does not work in the VirtualBox text console.

## 4. Download Splunk Enterprise for Ubuntu

1. Visit the [official Splunk Enterprise download page](https://www.splunk.com/en_us/download/splunk-enterprise.html) on Windows.
2. Select **Linux**, then the **64-bit `.deb`** package appropriate for Ubuntu.
3. Select **Copy wget link**. The link and version may change over time.
4. Paste the supplied command into your **Windows PowerShell SSH session** connected to Ubuntu, not into Windows PowerShell before SSH connects.

The lab used **Splunk Enterprise 10.6.1.1**, with the following package name:

```text
splunk-10.6.1.1-b2ae2962645c-linux-amd64.deb
```

The actual download command should come from Splunk's official site. Its general form is:

```bash
wget -O splunk-package.deb 'PASTE_OFFICIAL_SPLUNK_DOWNLOAD_URL_HERE'
```

Replace the placeholder URL with the **complete** URL provided by Splunk. Do not run the placeholder verbatim. A successful download should finish with a saved-file confirmation.

## 5. Install Splunk

If you used the generic download command above:

```bash
sudo apt install ./splunk-package.deb
```

If you instead downloaded the exact package used in this lab:

```bash
sudo apt install ./splunk-10.6.1.1-b2ae2962645c-linux-amd64.deb
```

Wait for installation to complete. Splunk is installed under `/opt/splunk` in this setup.

**Lab observation:** The package installation displayed a message about a missing `/opt/splunk/lib/python3.7/site-packages` path, but setup completed and Splunk later started successfully. Investigate any error if the package installation or service startup actually fails; do not assume all such messages are harmless.

## 6. Start Splunk for the first time

This lab initially started Splunk with:

```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```

On the first successful startup, Splunk prompted for an **administrator username and password**. Create a strong, unique password. **Do not put credentials in GitHub, screenshots, or scripts.**

The lab displayed a successful startup message and a Splunk Web URL ending in `:8000`.

> **Security note:** Running Splunk as root was the method used to complete this personal lab. Splunk warned that root operation is deprecated. For a more secure, maintainable setup, migrate to a dedicated non-root Splunk service account and follow the official Splunk instructions for systemd/boot-start configuration. Do not assume `--run-as-root` will remain supported in future releases.

## 7. Access Splunk Web from Windows

In your **Windows** browser, open:

**http://127.0.0.1:8000**

Sign in using the **Splunk administrator** credentials created at first startup (not necessarily the Ubuntu login). The lab reached the Splunk Enterprise home screen from Windows using this address.

If the page does not load, confirm the VM is running, Splunk is started, and TCP port `8000` is forwarded from Windows localhost to the VM.

## 8. First SPL search and visualization

In Splunk Web:

1. Open **Search & Reporting**.
2. Select a suitable time range, such as **All time**, for the initial exercise.
3. Enter:

```spl
index=_internal | stats count by sourcetype
```

4. Run the search and open **Statistics** to view the count by sourcetype.
5. Open **Visualization** and select **Column Chart**.
6. Optionally choose **Save As > Dashboard Panel** and create a dashboard named `CyberK9 Security Monitoring` with a panel named `Internal Log Events by Sourcetype`.

**Observed outcome:** The initial search returned **42,204 internal events** and **32 sourcetype rows** at the time of the screenshot. These are **point-in-time observations**, not expected results for every installation. The column chart rendered successfully.

**Interpretation:** `_internal` contains Splunk operational logs. This exercise confirms indexing, searching, aggregation, and charting; it does **not** establish that endpoint or security logs are being collected yet.

## 9. Daily startup and shutdown

### Startup

1. Start `CyberK9-Splunk-Lab` in VirtualBox and wait for Ubuntu to boot.
2. Open **Windows PowerShell** and connect:

```powershell
ssh -p 2222 k9admin@127.0.0.1
```

3. In the SSH session, start Splunk using the lab's current configuration:

```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```

4. In Windows Chrome or Edge, open **http://127.0.0.1:8000** and sign in.
5. Open the Udemy course in another tab and practice in the lab.

### Shutdown

In the SSH session:

```bash
sudo /opt/splunk/bin/splunk stop
sudo shutdown -h now
```

Wait for VirtualBox to show the VM is powered off. The SSH session will disconnect as Ubuntu shuts down.

## 10. Troubleshooting

| Symptom | What to check |
|---|---|
| Cannot paste into VirtualBox's Ubuntu text console | Use Windows PowerShell over SSH; paste into the SSH terminal instead. |
| SSH connection refused on port 2222 | VM running? OpenSSH active? NAT port forwarding host `2222` to guest `22`? |
| Splunk Web not loading at localhost:8000 | Splunk started? NAT forwarding host `8000` to guest `8000`? |
| Ubuntu login works but Splunk login fails | Splunk administrator credentials are separate from Ubuntu account credentials. |
| Download fails | Recopy the current official Linux `.deb` link; verify Ubuntu internet access. |
| Splunk search returns different counts | Event counts vary with uptime, time range, and indexed data. |
| Splunk says root startup is deprecated | Plan migration to a non-root service account; avoid relying on root long-term. |

Useful checks inside Ubuntu:

```bash
sudo systemctl status ssh
sudo /opt/splunk/bin/splunk status
ip -4 addr
```

## 11. Security and responsible documentation

- Bind forwarded host ports to **`127.0.0.1`**, not all interfaces.
- Use unique passwords and do not publish credentials, tokens, private keys, or personal data.
- Review screenshots before publishing; redact sensitive paths, identifiers, and browser tabs if needed.
- Take a VirtualBox snapshot before significant configuration changes or upgrades.
- Use the lab for authorized learning only.
- Check Splunk licensing and product limits on the official site; terms and versions may change.
- Treat this as a **learning environment**, not a production-hardened SIEM deployment.

## 12. Suggested evidence screenshots

Create a `screenshots/` folder and add your **own redacted screenshots** as you document the project:

1. `virtualbox-vm.png` — VirtualBox showing the running Ubuntu VM.
2. `ubuntu-ssh.png` — Successful Windows PowerShell SSH connection.
3. `splunk-login.png` — Splunk Web sign-in page (no passwords visible).
4. `splunk-home.png` — Splunk Enterprise home screen.
5. `splunk-spl-search.png` — Search with `index=_internal | stats count by sourcetype`.
6. `splunk-visualization.png` — Column chart output.
7. `splunk-dashboard.png` — Saved dashboard, if completed.

Example embedding after you add the file:

```markdown
![Splunk sourcetype column chart](screenshots/splunk-visualization.png)
```

## 13. Next steps

- Configure Splunk to run under a dedicated, non-root account.
- Learn SPL filtering, field extraction, aggregation, and time-series analysis.
- Ingest authorized Ubuntu authentication and system logs.
- Build dashboards for authentication activity and other security-relevant events.
- Explore alerting and detection engineering with controlled test data.
- Add documented screenshots and a brief reflection on lessons learned.

## References

- [Oracle VirtualBox documentation](https://www.virtualbox.org/manual/)
- [Ubuntu Server documentation](https://documentation.ubuntu.com/server/)
- [Splunk Enterprise documentation](https://docs.splunk.com/Documentation/Splunk)
- [Splunk Enterprise downloads](https://www.splunk.com/en_us/download/splunk-enterprise.html)

---

**Portfolio note:** This README documents a real personal lab build, with observed results distinguished from recommended settings and future work. It is not a claim of production deployment or security monitoring beyond the data actually ingested.
