# Cloud-Native SSH Honeypot & Global Threat Intelligence Feed 🚀

## 📝 Project Overview
This project demonstrates the deployment of a low-resource, high-utility **SSH Honeypot** built on AWS EC2 to capture, log, and analyze real-world adversarial brute-force attacks. By leveraging custom port obfuscation and network-level redirection, the system safely isolates malicious activities inside a simulated Debian environment while generating actionable threat intelligence.

---

## 🗺️ Network Architecture & Design

### 🔒 Cloud Network Security & Firewall Setup
The AWS Security Group was hardened to separate system administration from the honeypot target area:
* **Management Port:** Obfuscated to Custom High Port `42422` (Strictly restricted via Security Group to Admin IP only).
* **Attacker Target Port:** Publicly exposed standard SSH Port `22` (Open to the entire internet `0.0.0.0/0`).
* **Internal Redirection:** Port `22` traffic is transparently routed to Cowrie on Port `2222` using Linux kernel `iptables`.

![AWS Security Group Setup](images/aws_sg.png)
*(Note: Upload your AWS Security Group screenshot here)*

---

## 🛠️ Tools & Technologies Used
* **Cloud Infrastructure:** AWS EC2 (Ubuntu 24.04 LTS via Free Tier)
* **Honeypot Framework:** Cowrie SSH/Telnet Honeypot (Python Virtual Environment)
* **Network Defense/Routing:** Linux Netfilter (`iptables`), AWS Security Groups
* **Threat Hunting/Monitoring:** Linux `ss`, `tail -f`, JSON log analysis

---

## 🛑 Technical Challenges Faced & Lessons Learned (Troubleshooting Diary)

Building this project in a restricted resource environment presented several industry-style configuration blocks. Here is how they were triaged and resolved:

### 1. Windows SSH Permission Denied (Unprotected Private Key File)
* **The Issue:** When trying to connect from a Windows Host via CMD, SSH rejected the `.pem` file stating that permissions were too open (`0600` equivalent required).
* **The Resolution:** Resolved using Windows command-line Access Control Lists (`icacls`) to strip inherited permissions and restrict Read permissions solely to the current owner:
  ```cmd
  icacls "honeypot-key.pem" /grant:r "%username%":"(R)"
  icacls "honeypot-key.pem" /inheritance:r
  ```

### 2. Systemd SSH Socket Port Binding Refusal
* **The Issue:** After editing `/etc/ssh/sshd_config` to change the management port to `42422`, running `systemctl restart ssh` threw a connection refused error on the new port. Modern Ubuntu builds handle SSH via systemd sockets which locked the default port.
* **The Resolution:** Performed a forced systemd daemon reload and socket override to ensure the changes bound properly across the system kernel:
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl stop ssh.socket
  sudo systemctl restart ssh
  ```

### 3. Remote Host Identification Change Warning (MITM Defense Trigger)
* **The Issue:** Attempting to simulate an attack on Port 22 triggered a severe SSH security block: `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`. The laptop recognized that the server fingerprint changed from the authentic Ubuntu shell to the dummy honeypot shell.
* **The Resolution:** Cleared the cached out-of-date host keys from the local user directory to allow a fresh secure handshake:
  ```cmd
  ssh-keygen -R 51.21.200.112
  ```

### 4. Cowrie 3.x Source Structure & "Command Not Found" Error
* **The Issue:** Standard outdated guides look for a `bin/cowrie` script which no longer exists in the root directory in Cowrie 3.x, leading to a `-bash: cowrie: command not found` error even inside the Python virtual environment.
* **The Resolution:** Re-architected the setup by executing pip in editable mode within the environment to dynamically compile and load Cowrie binaries into the local path before firing up the server core:
  ```bash
  pip install -e .
  cowrie start
  ```

---

## 📊 Live Threat Intelligence Evidence & Validation

Using the `tail -f var/log/cowrie/cowrie.log` utility, the honeypot engine successfully went live on Port 2222 via `twistd`. 

A localized attack simulation mimicking an adversary attempting password combinations (`admin`) was successfully captured by the daemon log. The attacker was fooled into a decoy Debian GNU/Linux shell (`root@svr04:~#`) while full telemetry was saved.

![Live Attack Logs Captured](images/live_logs.png)
*(Note: Upload your final screenshot showing the successful password catch and tail log output here)*
