# Module 02 — Standalone Installation & Setup

## Learning Objectives
- Install Jenkins on Linux, Windows, and macOS.
- Understand the layout of `JENKINS_HOME`.
- Complete the initial setup wizard and create the first admin user.
- Put Jenkins behind a reverse proxy with HTTPS.
- Manage Jenkins as a system service.

## 1. Prerequisites

### Java
Jenkins is a Java application. The Long-Term Support (LTS) line of Jenkins typically requires a specific range of Java versions — current LTS releases need **Java 17 or Java 21**. Always check the documentation for the version you are installing.

Check Java:
```bash
java -version
```

If you don't have Java, install a JDK:
```bash
# Debian/Ubuntu
sudo apt update
sudo apt install -y openjdk-17-jdk

# RHEL/CentOS/Rocky
sudo dnf install -y java-17-openjdk
```

### Hardware sizing (starting point)
| Use case | CPU | RAM | Disk |
|----------|-----|-----|------|
| Small team, light builds | 2 vCPU | 4 GB | 50 GB |
| Medium team | 4 vCPU | 8 GB | 100 GB+ |
| Large team / many plugins | 8+ vCPU | 16 GB+ | 250 GB+ SSD |

The controller itself does not need to be huge — keep heavy builds on agents.

### Network
- Default Jenkins web port: **8080**.
- Default agent inbound port (JNLP): typically **50000**.
- Outbound internet access (for plugin downloads and updates).

## 2. Installation on Linux (Debian/Ubuntu)

Add the Jenkins repository and install the LTS package:

```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
```

Start and enable the service:
```bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

Check the log:
```bash
sudo journalctl -u jenkins -f
```

## 3. Installation on RHEL / Rocky / CentOS

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo \
  https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo dnf install -y jenkins
sudo systemctl enable --now jenkins
```

## 4. Installation on Windows
1. Download the Windows installer (`.msi`) from `https://www.jenkins.io/download/`.
2. Run the installer; choose "Run as Local System" or a dedicated service account.
3. Jenkins runs as a Windows Service. Manage it via `services.msc`.
4. `JENKINS_HOME` defaults to `C:\ProgramData\Jenkins\.jenkins`.

## 5. Installation on macOS

For a development install:
```bash
brew install jenkins-lts
brew services start jenkins-lts
```

Visit `http://localhost:8080`.

## 6. Running from the WAR file (any OS)

Useful for testing or pinned versions:
```bash
java -jar jenkins.war --httpPort=8080
```

## 7. `JENKINS_HOME` Layout

`JENKINS_HOME` is the single directory that holds **everything** Jenkins knows. Back this up and you can rebuild your controller.

Default locations:
- Debian/Ubuntu/RHEL: `/var/lib/jenkins`
- Windows: `C:\ProgramData\Jenkins\.jenkins`
- macOS (Homebrew): `~/.jenkins`

Important subdirectories:
```
JENKINS_HOME/
├── config.xml              # Global controller config
├── jobs/                   # One folder per job; build history lives here
├── plugins/                # Installed plugins (.jpi/.hpi files)
├── secrets/                # Master keys, identity, credentials store
├── users/                  # Local user accounts
├── nodes/                  # Agent definitions
├── workspace/              # Workspaces for builds running on the controller
├── logs/                   # Internal Jenkins logs
└── updates/                # Plugin update center cache
```

## 8. The Initial Setup Wizard

1. Open `http://<server>:8080` in a browser.
2. Jenkins displays a "Unlock Jenkins" page asking for an initial admin password.
3. Retrieve the password from the controller:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
4. Paste it into the form.
5. Choose **Install suggested plugins** for your first install. You can refine the list later.
6. Create your first admin user — use a strong password and a real email.
7. Confirm the Jenkins URL (this becomes the base for build links and webhook callbacks).

## 9. Reverse Proxy with HTTPS

Running Jenkins on port 8080 over plain HTTP is fine for a sandbox. For anything real, put it behind Nginx or Apache with TLS.

### Nginx example
```nginx
server {
    listen 443 ssl http2;
    server_name jenkins.example.com;

    ssl_certificate     /etc/letsencrypt/live/jenkins.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jenkins.example.com/privkey.pem;

    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 90;
    }
}

server {
    listen 80;
    server_name jenkins.example.com;
    return 301 https://$host$request_uri;
}
```

### Tell Jenkins about the external URL
**Manage Jenkins → System → Jenkins URL** — set to `https://jenkins.example.com/`.

Also ensure Jenkins only binds to localhost:
```bash
# /etc/default/jenkins  (or /etc/sysconfig/jenkins)
JENKINS_LISTEN_ADDRESS=127.0.0.1
```

## 10. Managing the Service

```bash
sudo systemctl start jenkins
sudo systemctl stop jenkins
sudo systemctl restart jenkins
sudo systemctl status jenkins
```

Default user/group: `jenkins:jenkins`. The Jenkins process runs as this user; permissions on `JENKINS_HOME` should match.

### JVM options
Edit `/etc/default/jenkins` (Debian) or `/etc/sysconfig/jenkins` (RHEL):
```
JAVA_ARGS="-Xms2g -Xmx4g -Djava.awt.headless=true"
```

After changing, restart Jenkins.

## 11. Smoke Test

1. Browse to your Jenkins URL.
2. Log in with the admin user.
3. **Manage Jenkins → System Information** — verify Java version, OS, plugin counts.
4. **Manage Jenkins → System Log** — confirm there are no startup errors.
5. Create a Freestyle job called `hello-world`, add a shell step `echo "Hello from Jenkins"`, and run it. You should see a green build with that output.

## 12. Hands-On Exercise
1. Provision a Linux VM (any cloud or local VirtualBox).
2. Install Jenkins from the official package repository.
3. Open the firewall for port 8080 (temporarily).
4. Complete the setup wizard.
5. Install Nginx and a self-signed cert; reverse-proxy Jenkins on HTTPS.
6. Confirm the Jenkins URL is set correctly.
7. Run the smoke test above.

## 13. Knowledge Check
1. Where is `JENKINS_HOME` on a Debian install?
2. Which file contains the initial admin password?
3. Why should you put Jenkins behind a reverse proxy?
4. How do you change the JVM heap size for Jenkins?
5. What is the default Jenkins web port?

## What's Next
In **Module 03** you'll learn the Jenkins UI, create your first Freestyle job, and understand triggers, workspaces, and artifacts.
