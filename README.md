# Deploying a Java WAR Application on Apache Tomcat (Ubuntu EC2)

This guide provides a step-by-step process for deploying a Java Web Application (WAR file) onto an Apache Tomcat Server running on an AWS EC2 Ubuntu instance.

## Prerequisites

Before you begin, ensure you have the following:

  * **AWS EC2 Ubuntu** (22.04+) instance ready.
  * **Security Group** configured with the following inbound rules:
      * Port **22** → SSH (for remote access)
      * Port **8080** → Tomcat (for application access)
      * Port **80** → Optional (if using a reverse proxy like Nginx)
  * **Java 17+** is available (we will install it).
  * Your `app.war` file ready for deployment.

-----

## Deployment Steps

### 1\. Update Server

Start by ensuring your server's package list is up-to-date and all existing packages are upgraded.

```bash
sudo apt update && sudo apt upgrade -y
```

### 2\. Install Java 17

Tomcat 10 requires Java 17 or newer. Install the OpenJDK package.

```bash
sudo apt install openjdk-17-jdk -y
```

**Verify Installation:**

```bash
java -version
```

### 3\. Download & Install Tomcat 10

Download the Tomcat 10 archive, extract it to the `/opt` directory, and rename the folder for simplicity.

```bash
cd /opt
sudo wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.49/bin/apache-tomcat-10.1.49.tar.gz
sudo tar -xvzf apache-tomcat-10.1.49.tar.gz
sudo mv apache-tomcat-10.1.49 tomcat10
sudo rm apache-tomcat-10.1.49.tar.gz
```

### 4\. Give Permissions

Set the appropriate file permissions and ownership so the `ubuntu` user can manage the Tomcat installation without needing `sudo` every time.

```bash
sudo chmod -R 755 /opt/tomcat10
sudo chown -R ubuntu:ubuntu /opt/tomcat10
```

### 5\. Start Tomcat Server

Navigate to the binary directory and start the server.

```bash
cd /opt/tomcat10/bin
./startup.sh
```

**Verify Access in Browser:**

Open your browser and navigate to:

```
http://<EC2-PUBLIC-IP>:8080
```

You should see the Apache Tomcat default page.

### 6\. Deploy the WAR File

Copy your compiled WAR file into the `webapps` directory of your Tomcat installation. Tomcat will automatically deploy it.

```bash
sudo cp /path/to/your-app.war /opt/tomcat10/webapps/
```

**Check Deployment:**

List the contents of the `webapps` directory. Tomcat automatically extracts the WAR file into a directory of the same name.

```bash
ls /opt/tomcat10/webapps
```

You should see both the extracted directory and the WAR file:

```
your-app/
your-app.war
```

### 7\. Restart Tomcat (Important)

A restart often ensures that the application is loaded correctly, especially after manual deployment.

```bash
cd /opt/tomcat10/bin
./shutdown.sh
./startup.sh
```

### 8\. Access Your Application

Access your deployed application using your EC2 Public IP, the Tomcat port (`8080`), and the name of your application directory (derived from the WAR file name).

```
http://<PUBLIC-IP>:8080/your-app/
```

**Example:**

```
http://44.210.15.172:8080/dptweb-1.0/
```

### 9\. Fix JSP Navigation Issues (If Applicable)

If you encounter navigation issues with JSPs, ensure that your links explicitly include the `.jsp` extension.

**Correct JSP Links:**

```html
<a href="login.jsp">Login</a>
<a href="register.jsp">Register</a>
```

### 10\. View Tomcat Logs

To troubleshoot issues during startup or runtime, monitor the primary Tomcat log file.

```bash
tail -f /opt/tomcat10/logs/catalina.out
```

### 11\. Tomcat Control Commands

For quick reference, here are the control commands:

| Action | Command |
| :--- | :--- |
| **Start** | `/opt/tomcat10/bin/startup.sh` |
| **Stop** | `/opt/tomcat10/bin/shutdown.sh` |

### 12\. Tomcat Folder Structure

Understanding the key directory structure can help with configuration and deployment.

```
/opt/tomcat10
 ├── bin/       # Startup/shutdown scripts
 ├── conf/      # Server configuration files (e.g., server.xml)
 ├── logs/      # Log files (e.g., catalina.out)
 ├── webapps/   # The deployment folder
      ├── your-app/
      └── your-app.war
```

-----

## Successfully Deployed

Your Java Web App is now running on Apache Tomcat 10 in an AWS EC2 Ubuntu instance\!






# GitHub Actions → EC2 (Tomcat) CI/CD — Full Step-by-Step Guide

This document shows a complete, error-free GitHub Actions workflow to **build a Maven Java project (WAR)** and **deploy it automatically to an EC2 instance running Apache Tomcat**. Follow each step exactly. After this, your `git push` to `main` will build the WAR, copy it to EC2, and restart Tomcat.

-----

## Prerequisites

  - GitHub repository containing your Maven Java web project (must build a `.war` in `target/`).
  - An AWS EC2 Ubuntu instance with:
      - Java 17 installed.
      - Tomcat installed at `/opt/tomcat/tomcat10` (adjust paths if needed).
      - A `tomcat` systemd service configured (recommended) or the ability to run startup/shutdown scripts.
  - You have SSH access to the EC2 instance using a key-pair.

-----

## Step 1 — Prepare EC2 (One-Time Setup)

1.  **SSH into EC2:**

    ```bash
    ssh -i /path/to/your-key.pem ubuntu@EC2_PUBLIC_IP
    ```

2.  **Install Java 17 (if not done):**

    ```bash
    sudo apt update -y
    sudo apt install -y openjdk-17-jdk
    java -version
    ```

3.  **Install/Extract Tomcat (Example Tomcat 10):**

    ```bash
    sudo mkdir -p /opt/tomcat
    cd /opt/tomcat
    sudo wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.49/bin/apache-tomcat-10.1.49.tar.gz
    sudo tar -xvzf apache-tomcat-10.1.49.tar.gz
    sudo mv apache-tomcat-10.1.49 tomcat10
    sudo rm apache-tomcat-10.1.49.tar.gz
    ```

4.  **Set Permissions (adjust user if necessary):**

    ```bash
    sudo chown -R ubuntu:ubuntu /opt/tomcat10
    sudo chmod -R 755 /opt/tomcat10
    ```

5.  **(Recommended) Create Tomcat Systemd Service:**

    ```bash
    sudo tee /etc/systemd/system/tomcat.service > /dev/null <<'EOF'
    [Unit]
    Description=Apache Tomcat Server
    After=network.target

    [Service]
    User=ubuntu
    Group=ubuntu
    Type=forking

    Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
    Environment="CATALINA_HOME=/opt/tomcat10"
    Environment="CATALINA_BASE=/opt/tomcat10"

    ExecStart=/opt/tomcat10/bin/startup.sh
    ExecStop=/opt/tomcat10/bin/shutdown.sh

    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    EOF

    sudo systemctl daemon-reload
    sudo systemctl enable --now tomcat
    sudo systemctl status tomcat
    ```

6.  **Ensure `webapps/` is Writable:**

    ```bash
    sudo chmod -R 775 /opt/tomcat10/webapps
    sudo chown -R ubuntu:ubuntu /opt/tomcat10/webapps
    ```

-----

## Step 2 — Create an SSH Key Pair for GitHub Actions

> We generate a dedicated key for GitHub Actions (avoid using your personal EC2 key).

1.  **Generate the key on your local machine:**

    ```bash
    ssh-keygen -t rsa -b 4096 -C "github-actions" -f ./gha_deploy_key -N ""
    ```

    This creates `gha_deploy_key` (private) and `gha_deploy_key.pub` (public). `-N ""` creates the key without a passphrase.

2.  **Copy the public key content:**

    ```bash
    cat gha_deploy_key.pub
    ```

    Copy the entire single-line output.

3.  **Add the public key to the EC2 user's `authorized_keys` (on EC2):**

    ```bash
    mkdir -p ~/.ssh
    echo "paste-the-public-key-line-here" >> ~/.ssh/authorized_keys
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    ```

-----

## Step 3 — Add Secrets to GitHub

1.  In your GitHub repo go to: **Settings → Secrets and variables → Actions → New repository secret**.

2.  **Create these secrets:**

| Secret Name | Value | Description |
| :--- | :--- | :--- |
| `EC2_SSH_KEY` | Paste the **full private key** from `gha_deploy_key` (starts with `-----BEGIN...`). | Used for authentication. |
| `EC2_HOST` | Your EC2 public IP or hostname (e.g., `44.210.15.172`). | Target server address. |
| `EC2_USER` | `ubuntu` (or the user you SSH into). | Target server username. |
| `EC2_SSH_PORT` | `22` (or your custom SSH port). | Optional. Default is 22. |

-----

## Step 4 — Add GitHub Actions Workflow File

Create the file path in your repo: `.github/workflows/deploy-to-ec2.yml`

Paste the following content (adjust Tomcat path if needed):

```yaml
name: Deploy to EC2 Tomcat

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Java 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Build WAR with Maven
        run: mvn -B clean package -DskipTests

      - name: Copy WAR to EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          port: ${{ secrets.EC2_SSH_PORT || '22' }}
          source: "target/*.war"
          target: "/opt/tomcat/tomcat10/webapps/"

      - name: Restart Tomcat
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          port: ${{ secrets.EC2_SSH_PORT || '22' }}
          script: |
            sudo systemctl restart tomcat
```

> **Note on Restart:** If you did not set up the systemd service , change the `script` in the **Restart Tomcat** step to use the binaries:
>
> ```yaml
> script: |
>   /opt/tomcat10/bin/shutdown.sh || true
>   /opt/tomcat10/bin/startup.sh
> ```

-----

## Step 5 — Common Troubleshooting & Checks

### Permission denied (publickey)

  * Ensure the public key is correctly in `~/.ssh/authorized_keys` of the EC2 user.
  * Verify the private key in the `EC2_SSH_KEY` secret is complete (including `BEGIN/END` lines).

### WAR not deploying / app 404

  * Check Tomcat logs on EC2:
    ```bash
    tail -n 200 /opt/tomcat10/logs/catalina.out
    ```
  * Verify the WAR extracted into `/opt/tomcat10/webapps/<appname>/`.

-----

## Step 6 — Test the Pipeline

1.  Commit & push the workflow file:

    ```bash
    git add .github/workflows/deploy-to-ec2.yml
    git commit -m "Add GitHub Actions deploy-to-ec2 workflow"
    git push origin main
    ```

2.  Open **GitHub → Actions** and monitor the workflow run.

3.  After success, open your browser:

    ```
    http://EC2_PUBLIC_IP:8080/<your-app-context>/
    ```

    Verify the application updated.

-----

## Done

Your GitHub Actions CI/CD pipeline will now automatically build your Maven WAR and deploy it to your EC2 Tomcat instance on every push to `main`.





# Jenkins CI/CD: Java WAR Deployment to EC2 Tomcat

This guide provides the required, step-by-step, error-free instructions to implement a Jenkins pipeline for building a Maven Java project and deploying the resulting WAR file to an Apache Tomcat server on EC2.

**(Assumption: You have two separate servers: a Jenkins Host and a Tomcat Host. If using a single server, commands are the same but run on the same machine.)**

## Quick Checklist Before Start

  * Jenkins host is publicly reachable (or from GitHub for webhooks).
  * SSH access between **Jenkins → Tomcat** is configured (we'll set this up).
  * You have `sudo` permissions on both servers.
  * Your GitHub repository contains a Java Maven project and you will add the `Jenkinsfile`.

-----

## A. Install Prerequisites on Jenkins Server (Minimal & Required)

Run these commands as a user with `sudo` access:

```bash
# update and upgrade system
sudo apt update && sudo apt upgrade -y

# install Java 17 (required by Jenkins itself and for building Java apps)
sudo apt install -y openjdk-17-jdk

# install Git and Maven (build tools)
sudo apt install -y git maven

# verify installations
java -version
git --version
mvn -v
```

## B. Install Jenkins (Official Package)

Install Jenkins using its official repository to ensure you get the stable release.

```bash
# 1. Add repository key + source
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list

# 2. Update and install Jenkins
sudo apt update
sudo apt install -y jenkins

# 3. Enable & start the service
sudo systemctl daemon-reload
sudo systemctl enable --now jenkins

# 4. Check status
sudo systemctl status jenkins
```

### Unlock Jenkins UI

1.  Open your browser: `http://<JENKINS_IP>:8080` (ensure your Security Group/Firewall allows port 8080).
2.  Fetch the initial admin password:
    ```bash
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
    ```
3.  Paste the password into the UI and follow the setup wizard to create an admin user.

## C. Minimal Plugins to Install (Required Only)

Jenkins UI → **Manage Jenkins** → **Manage Plugins** → **Available**

Search for and install the following plugins. Jenkins may require a restart after installation.

  * `Pipeline` (workflow-aggregator)
  * `Git plugin`
  * `GitHub plugin`
  * `Credentials`
  * `SSH Agent Plugin`
  * `Maven Integration` (helpful for parsing `pom.xml`)

## D. Global Tool Configuration (Required)

Jenkins UI → **Manage Jenkins** → **Global Tool Configuration**

Configure the tools you installed in Step A:

  * **JDK**
      * Add JDK → Name: `jdk-17`
      * **Uncheck** "Install automatically" (since we installed Java system-wide).
  * **Maven**
      * Add Maven → Name: `Maven-3`
      * **Uncheck** "Install automatically".
  * **Git**
      * Usually auto-detected. Leave as is, Name: `git`.

Click **Save**.

## E. Create SSH Key for Jenkins & Enable Access to Tomcat

We generate a key pair for the Jenkins service user (`jenkins`) and authorize the public key on the Tomcat host's `ubuntu` user account.

1.  **Generate key on Jenkins server (run as root or ubuntu):**

    ```bash
    sudo -u jenkins mkdir -p /var/lib/jenkins/.ssh
    sudo -u jenkins ssh-keygen -t rsa -b 4096 -f /var/lib/jenkins/.ssh/id_rsa -N ""
    sudo chown -R jenkins:jenkins /var/lib/jenkins/.ssh
    sudo chmod 700 /var/lib/jenkins/.ssh
    sudo chmod 600 /var/lib/jenkins/.ssh/id_rsa
    ```

2.  **Copy public key to Tomcat server:**

    ```bash
    # Show the key on Jenkins; copy the entire single line of output
    sudo -u jenkins cat /var/lib/jenkins/.ssh/id_rsa.pub
    ```

3.  **Add public key to Tomcat server (SSH into Tomcat using your normal PEM):**

    ```bash
    # on Tomcat host
    mkdir -p ~/.ssh
    # Paste the key you copied above into the file
    echo "paste-copied-public-key-here" >> ~/.ssh/authorized_keys
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
    chown ubuntu:ubuntu -R ~/.ssh
    ```

4.  **Test connection from Jenkins server:**

    ```bash
    sudo -u jenkins ssh -o StrictHostKeyChecking=no ubuntu@<TOMCAT_IP> 'echo ok'
    ```

    **Expected Output:** `ok` (If successful, SSH keys are correct.)

## F. Add SSH Private Key to Jenkins Credentials

1.  Jenkins UI → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2.  **Kind:** SSH Username with private key
3.  **Username:** `ubuntu` (This is the user on the Tomcat host)
4.  **Private Key:** **Enter directly** → **paste the full content** of the private key (`id_rsa`):
    ```bash
    # On Jenkins server, copy this entire content:
    sudo cat /var/lib/jenkins/.ssh/id_rsa
    ```
5.  **ID:** `ec2-ssh` (Remember this ID, we use it in the Jenkinsfile)
6.  Click **Save**.

## G. Add `Jenkinsfile` to Your Repo (Pipeline as Code)

Create a file named `Jenkinsfile` in the root of your repository and paste the following content, replacing the placeholders (like URL and IP):

```groovy
pipeline {
  agent any
  // Use tool names defined in Step D
  tools { jdk 'jdk-17'; maven 'Maven-3' }

  environment {
    // --- CONFIGURE THESE VARIABLES ---
    GIT_BRANCH = '*/main'                      // change if branch different
    TOMCAT_HOST = '54.172.239.166'             // <--- YOUR TOMCAT IP HERE
    TOMCAT_USER = 'ubuntu'
    WEBAPPS_DIR = '/opt/tomcat10/webapps/'     // correct WAR deployment path on Tomcat
    SSH_CREDENTIALS_ID = 'ec2-ssh'             // Credential ID from Step F
    // ---------------------------------
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: env.GIT_BRANCH]],
          userRemoteConfigs: [[url: 'https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPO.git']]]) // <--- YOUR REPO URL HERE
      }
    }

    stage('Build') {
      steps {
        sh 'mvn -B clean package -DskipTests'
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
      }
    }

    stage('Deploy') {
      steps {
        // Uses the private key stored in Jenkins credentials
        sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
          sh """
            # Copy WAR to Tomcat webapps folder
            scp -o StrictHostKeyChecking=no target/*.war ${TOMCAT_USER}@${TOMCAT_HOST}:${WEBAPPS_DIR}
            # Restart Tomcat service to trigger deployment
            ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_HOST} 'sudo systemctl restart tomcat'
          """
        }
      }
    }
  }
  post {
    success { echo 'Deployment successful' }
    failure { echo 'Deployment failed' }
  }
}
```

**Commit & push this `Jenkinsfile` to your repository.**

## H. Create Jenkins Pipeline Job

1.  Jenkins UI → **New Item** → Name: `app-ci` → Choose **Pipeline** → **OK**
2.  In the job configuration:
      * Scroll down to the **Pipeline** section.
      * **Definition:** `Pipeline script from SCM`
      * **SCM:** `Git`
      * **Repository URL:** `https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPO.git`
      * **Credentials:** (Leave blank for public repo)
      * **Branch Specifier:** `*/main`
      * **Script Path:** `Jenkinsfile`
      * **Build Triggers:** Check `GitHub hook trigger for GITScm polling`
3.  Click **Save**.

## I. Configure GitHub Webhook

This step ensures a `git push` automatically triggers the Jenkins job.

1.  On your GitHub repo → **Settings** → **Webhooks** → **Add webhook**
2.  **Payload URL:** `http://<JENKINS_PUBLIC_IP>:8080/github-webhook/`
3.  **Content type:** `application/json`
4.  **Which events:** Just the `Push` event
5.  Click **Add webhook**.

> **Quick Check:** Test by pushing a minor commit. Go to GitHub → Webhooks → check status (should show 200 OK). Jenkins job should trigger automatically.

## J. Run Manual Build Once for Verification

1.  On the Jenkins job (`app-ci`) page → **Build Now**
2.  Open the Console Output of the running build and watch the stages:
      * `Checkout` success
      * `Maven build` success → WAR artifact created
      * `scp` success (no permission error)
      * `tomcat restart` success

**Common Quick Fixes:**
| Error | Fix |
| :--- | :--- |
| `Permission denied (publickey)` | Re-run the SSH test (Step E.4) and verify the private key in Jenkins Credentials (Step F). |
| `No such file or directory` | Correct the `WEBAPPS_DIR` in the `Jenkinsfile` (Step G) to match the Tomcat path. |
| `tomcat.service failed` | On Tomcat server: `sudo systemctl status tomcat` and verify systemd file (check the previous EC2 guide for the service file). |

## K. Minimal Troubleshooting Commands (Useful)

| Server | Action | Command |
| :--- | :--- | :--- |
| **Jenkins** | Test SSH access | `sudo -u jenkins ssh -o StrictHostKeyChecking=no ubuntu@<TOMCAT_IP> 'echo ok'` |
| **Jenkins** | View private key | `sudo cat /var/lib/jenkins/.ssh/id_rsa` |
| **Tomcat** | Check WAR presence | `ls -l /opt/tomcat10/webapps/` |
| **Tomcat** | Check Tomcat logs | `tail -n 200 /opt/tomcat10/logs/catalina.out` |

-----

## Final Notes (Required Best Practices)

  * **Security:** Keep private keys **only** in Jenkins Credentials, never in the repository.
  * **Pipeline as Code:** Always use the `Jenkinsfile` (Pipeline script from SCM).
  * **Access:** Limit Security Group/Firewall rules (open 8080 only to trusted IPs).

