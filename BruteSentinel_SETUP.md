# SETUP.md
BruteSentinel Setup Guide
This file provides step by step, copy/pasteable commands and configuration snippets to reproduce the SIEM lab: Ubuntu victim, Kali attacker, Splunk ingestion, rsyslog forwarding, packet capture, and dashboard import.

> Assumptions
- You have an isolated private network or host-only network between two VMs.
- Ubuntu server has network access to the Splunk instance.
- You have sudo privileges on the VMs.
- Replace `<ubuntu-ip>`, `<splunk-ip>`, `<iface>`, and other placeholders with your environment values.

TABLE OF CONTENTS
1. VM setup summary
2. Install Splunk on the Splunk host
3. Configure Splunk to receive syslog
4. Configure rsyslog on Ubuntu victim to forward auth logs
5. Produce SSH failed attempts from Kali
6. Packet capture on Ubuntu victim
7. Splunk saved search and dashboard notes
8. Helpful scripts

---

## 1. VM setup summary
- Ubuntu Server (victim)
  - Hostname: ubuntu-victim
  - Interface: use host-only or internal network interface
- Kali Linux (attacker)
  - Hostname: kali-attacker
- Splunk Server
  - Can be on the Ubuntu victim or a separate VM
  - If colocated on Ubuntu victim, use localhost for forwarding

Network plan options
- Option A: Splunk on separate VM `splunk-ip`, Ubuntu victim forwards to `splunk-ip:514`
- Option B: Splunk on Ubuntu victim `127.0.0.1:514`, rsyslog forwards locally

---

## 2. Install Splunk (Ubuntu .deb)
Note: Use the Splunk release you prefer. The commands below show the canonical flow. Update filenames to match the download you have.

```bash
# On the Splunk host (or Ubuntu victim if colocated)
# Create a directory to store the package
mkdir -p ~/downloads && cd ~/downloads

# Place the Splunk .deb file in this directory first.
# Example filename: splunk-9.0.0-xxxxxxxxxxx-amd64.deb
# Then install
sudo dpkg -i splunk-9.*.deb

# Accept license and create admin user
sudo /opt/splunk/bin/splunk start --accept-license --answer-yes
# You will be prompted to create admin credentials on first web login.
# Alternatively use --seed-credential-file or automations if needed

# Enable Splunk to start at boot
sudo /opt/splunk/bin/splunk enable boot-start
```

If dpkg fails due to dependencies, run:
```bash
sudo apt-get update
sudo apt-get -f install
```

Access Splunk web at:
```
http://<splunk-ip>:8000
```
Log in with the admin account you created.

---

## 3. Configure Splunk to receive syslog
We will configure Splunk to listen on a UDP or TCP port for syslog messages. UDP 514 is common but may require root privileges. Example uses TCP 5140 to avoid running Splunk as root.

### Create a TCP input in Splunk
Run this command on the Splunk host:
```bash
sudo /opt/splunk/bin/splunk add tcp 5140 -sourcetype syslog -index ubuntu_syslog -auth admin:changeme
```
Replace `admin:changeme` with your admin credentials, and pick a port you can manage.

Alternatively configure via Splunk web:
1. Settings -> Data Inputs -> TCP -> New -> Port 5140 -> Set sourcetype to syslog -> Set index to ubuntu_syslog -> Save.

Important: If you prefer UDP 514, be aware Splunk needs permissions for low ports. Using a higher port like 5140 avoids that.

Open firewall port if needed:
```bash
# Example for UFW
sudo ufw allow 5140/tcp
```

Create the index if not auto-created:
```bash
sudo /opt/splunk/bin/splunk add index ubuntu_syslog -auth admin:changeme
```

---

## 4. Configure rsyslog on Ubuntu victim to forward auth logs to Splunk
We will forward authentication logs only to reduce noise.

### Create a dedicated rsyslog config
On the Ubuntu victim, create `/etc/rsyslog.d/50-splunk-forward.conf` and add:

```conf
# Forward auth and authpriv logs to Splunk TCP listener
if ($programname == 'sshd' or $syslogfacility-text == 'auth' or $syslogfacility-text == 'authpriv') then {
    action(type="omfwd" Target="<splunk-ip>" Port="5140" Protocol="tcp")
    stop
}
```

If you want to forward all syslog messages, use:
```conf
*.* action(type="omfwd" Target="<splunk-ip>" Port="5140" Protocol="tcp")
```

After saving the file, restart rsyslog:
```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

Verify logs are being forwarded by using `tcpdump` on the Splunk host:
```bash
sudo tcpdump -n -i <iface> tcp port 5140 -c 20
```
You should see incoming syslog lines.

If Splunk is local and listening on 5140, you can also tail `/var/log/syslog` to confirm local forwarding.

---

## 5. Produce SSH failed attempts from Kali
Use a controlled loop to generate failed authentications for your demo.

### Simple single-command attempt
```bash
ssh invaliduser@<ubuntu-ip>
# Enter wrong password
```

### Script to produce repeated failed attempts
Create `scripts/ssh-brute.sh` on Kali:
```bash
#!/bin/bash
target="$1"
count="${2:-100}"
user="${3:-invaliduser}"

if [ -z "$target" ]; then
  echo "Usage: $0 <target-ip> [count] [user]"
  exit 1
fi

for i in $(seq 1 $count); do
  # Try to connect with a username that likely does not exist
  ssh -o ConnectTimeout=5 -o BatchMode=no -o StrictHostKeyChecking=no "$user@$target" exit 2>/dev/null
  echo "Attempt $i"
  sleep 0.2
done
```

Make it executable and run:
```bash
chmod +x scripts/ssh-brute.sh
./scripts/ssh-brute.sh <ubuntu-ip> 200 invaliduser
```

Notes:
- The script will generate connection attempts and failed logins. Adjust `sleep` and `count` for demo speed.
- Do not run aggressive loops on production systems.

---

## 6. Packet capture on Ubuntu victim
Capture SSH packets to a PCAP for Wireshark validation.

```bash
# Capture live SSH traffic on interface
sudo tcpdump -i <iface> port 22 -w /srv/brutesentinel/pcap/ssh_attempts.pcap

# To capture for a fixed number of packets
sudo tcpdump -i <iface> port 22 -c 1000 -w /srv/brutesentinel/pcap/ssh_attempts.pcap
```

Move or copy the PCAP to your analysis workstation or open in Wireshark:
```bash
scp ubuntu@<ubuntu-ip>:/srv/brutesentinel/pcap/ssh_attempts.pcap ./pcap/
wireshark ./pcap/ssh_attempts.pcap
```

Wireshark filters to inspect:
- `ssh`
- `tcp.flags.syn==1 && tcp.flags.ack==0` for SYN scans
- `tcp.port==22 && tcp.analysis.retransmission` for retransmissions

---

## 7. Splunk saved search and dashboard
Below is the sample search to use in a Splunk Saved Search or Dashboard panel.

### Saved search
Name: `failed_ssh_attempts_by_ip`
Search string:
```spl
index=ubuntu_syslog sourcetype=syslog "sshd" "Failed password"
| rex field=_raw "Failed password for (invalid user )?(?<user>\S+) from (?<src_ip>\S+)"
| bin _time span=1m
| stats count by _time, src_ip, user
| where count >= 3
| sort - _time count
```
This will show source IPs with 3 or more failed attempts in a 1 minute bin.

### Dashboard panels
Create panels for:
- Timechart of failed attempts per minute
- Top source IPs by count
- Top usernames targeted
- Link to PCAPs or evidence folder

You can export the dashboard JSON via Splunk web for `splunk/` folder. I can generate a simple dashboard JSON if you want.

---

## 8. Helpful scripts and automation
Place these example scripts in `scripts/` in the repo.

### scripts/ssh-brute.sh
(See section 5 above)

### scripts/capture-ssh.sh
```bash
#!/bin/bash
outdir="/srv/brutesentinel/pcap"
mkdir -p "$outdir"
outfile="$outdir/ssh_attempts_$(date +%Y%m%d_%H%M%S).pcap"
sudo tcpdump -i <iface> port 22 -w "$outfile"
echo "Capture writing to $outfile"
```

### scripts/import-dashboard.sh
If you have dashboard JSON, use the Splunk REST API to import it. Replace credentials and host:
```bash
curl -k -u admin:changeme https://<splunk-ip>:8089/servicesNS/admin/search/data/ui/views -d name="BruteSentinel_Dashboard" -d "eai:data=@dashboard.json" -H "Content-Type:application/json"
```
Adjust REST endpoint and payload as needed. You can also import via Splunk web UI.

---

## Troubleshooting
- No logs arriving in Splunk:
  - Confirm rsyslog is running and config file loaded: `sudo systemctl status rsyslog`
  - Confirm Splunk input is listening: `sudo /opt/splunk/bin/splunk list tcp`
  - Use `tcpdump` on Splunk host to validate network traffic
- Splunk shows raw lines but regex not extracting fields:
  - Inspect `_raw` text in Splunk, adjust `rex` pattern if your distro formats differ
- Permission issues for low ports:
  - Use a higher port like 5140 and adjust rsyslog action to forward there

---

## Security and ethics reminder
Run this lab only on systems you own or where you have explicit written authorization. Do not perform scans or brute forcing on third-party systems. Treat all data responsibly. If you publish findings containing external IPs, follow responsible disclosure procedures.

---

If you want, I will:
- Generate `splunk/BruteSentinel_dashboard.json` export for you to import directly into Splunk.
- Create the `scripts/ssh-brute.sh` and `scripts/capture-ssh.sh` files in the repo and make them downloadable.
- Draft `DEMO.md` with a transcript template for your recorded walkthrough.

Tell me which to produce next and I will create the files for download.
