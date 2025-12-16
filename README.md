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



## Step-by-Step Deployment Guide for Java Login App (Docker)

This guide assumes you are starting with a clean EC2 instance and have Docker installed.

-----
### Step 0: Install Docker on Ubuntu EC2

Before beginning the deployment, you must install Docker on your Ubuntu EC2 instance. Follow these steps.

### 1\. Update the Package Index

Start by updating your local package index to ensure you have access to the latest versions.

```bash
sudo apt update
```

### 2\. Install Necessary Packages

Install packages that allow `apt` to use a repository over HTTPS:

```bash
sudo apt install ca-certificates curl gnupg lsb-release
```

### 3\. Add Docker's Official GPG Key

Add the official GPG key for the Docker repository to verify the downloaded packages.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### 4\. Set Up the Docker Repository

Add the Docker repository to your list of sources.

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 5\. Install Docker Engine

Update the package index again with the new repository, and then install Docker Engine, CLI, and Containerd.

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

### 6\. Verify Docker Installation

Check that the Docker service is running correctly.

```bash
sudo systemctl status docker
```

You should see an output indicating the status is **`active (running)`**.

### 7\. Run Docker Without `sudo` (Optional but Recommended)

By default, only the root user and users in the `docker` group can run Docker commands. To run Docker without prefixing every command with `sudo`:

1.  **Add your current user to the `docker` group:**
    ```bash
    sudo usermod -aG docker $USER
    ```
2.  **Apply the new group membership** (you will need to log out and log back in, or simply run the following command to activate the group):
    ```bash
    newgrp docker
    ```
3.  **Test Docker without `sudo`:**
    ```bash
    docker run hello-world
    ```

If the `hello-world` container runs successfully and prints a message, Docker is correctly installed and configured.

-----

You are now ready to proceed with **Step 1** of the main guide (Verify Project Structure).

Do you want to proceed with **Step 1** of the original deployment guide now?
### Step 1: Verify Your Project Structure

Ensure your project files are organized correctly. The `Dockerfile` **must** be at the root of your project directory.

| Folder/File | Location | Purpose |
| :--- | :--- | :--- |
| `Java-Login-App/` | Root directory (e.g., on your EC2 instance) | Project main directory |
| `pom.xml` | Inside `Java-Login-App/` | Maven Project Object Model |
| `src/` | Inside `Java-Login-App/` | Java source code |
| `Dockerfile` | Inside `Java-Login-App/` (Project Root) | Docker build instructions |

### Step 2: The Multi-Stage Dockerfile

Create a file named `Dockerfile` in the root of your `Java-Login-App/` directory with the **exact** content below. This uses a **multi-stage build** for a smaller final image.

```dockerfile
########## 1) BUILD STAGE ##########
# Use a Maven image with JDK 17 to build the WAR file
FROM maven:3.9.6-eclipse-temurin-17 AS build
WORKDIR /app

# Copy pom.xml first to cache dependencies (faster subsequent builds)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source and build the final WAR package
COPY src ./src
RUN mvn -B clean package -DskipTests

########## 2) RUN STAGE ##########
# Use a standard Tomcat 10 with JDK 17 for the final runtime
FROM tomcat:10.1-jdk17

# Remove default Tomcat applications
RUN rm -rf /usr/local/tomcat/webapps/*

# Copy the generated WAR file from the 'build' stage 
# and rename it to ROOT.war to make it the default app
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war

# Tomcat runs on port 8080 by default
EXPOSE 8080
# Command to start Tomcat
CMD ["catalina.sh", "run"]
```

> **No Manual `mvn package`:** The WAR file is built automatically *inside* the Docker container during the build process.

### Step 3: Build the Docker Image

Navigate to the project root directory (`Java-Login-App/`) on your EC2 instance and run the build command.

```bash
docker build -t my-login-app .
```

  * `-t my-login-app`: Tags the resulting image with the name `my-login-app`.
  * `.`: Specifies the build context (current directory), where the `Dockerfile` is located.

**Verify the Image:**

```bash
docker images
```

You should see an image listed with the repository name `my-login-app`.

### Step 4: Run the Container

The Tomcat server inside the container listens on port **8080**. You must map this to a port on your EC2 host machine.

```bash
docker run -d --name login-app -p 8080:8080 my-login-app
```

  * `-d`: Runs the container in detached mode (background).
  * `--name login-app`: Assigns a name to the running container.
  * `-p 8080:8080`: Maps the host port `8080` to the container port `8080`.
  * `my-login-app`: The image to use.

**Verify the Container Status:**

```bash
docker ps
```

The output under the `PORTS` column should look like this, confirming the port mapping:
`0.0.0.0:8080->8080/tcp`

### Step 5: Open the Application

Once the container is running, access your application using the public IP of your EC2 instance.

**Primary Access URL:**

```
http://<EC2-IP>:8080/pages/login.jsp
```

#### Option A — Add a Root URL Redirect (`index.jsp` Fix)

If you want the base URL (`http://<EC2-IP>:8080/`) to automatically redirect to your login page, execute the following commands to create an `index.jsp` file inside the running container:

1.  **Enter the container:**
    ```bash
    docker exec -it login-app bash
    ```
2.  **Create the redirect file:**
    ```bash
    cat > /usr/local/tomcat/webapps/ROOT/index.jsp << 'EOF'
    <% response.sendRedirect("pages/login.jsp"); %>
    EOF
    ```
3.  **Exit the container:**
    ```bash
    exit
    ```

Now, visiting `http://<EC2-IP>:8080/` will redirect you.

### Step 6: EC2 Security Group Check (Crucial\!)

If your container is running but the page won't load from your browser, the **EC2 Security Group** is almost certainly blocking the traffic. You must explicitly open port 8080.

In your AWS Console, navigate to your EC2 instance's Security Group and add an **Inbound Rule**:

| Setting | Value |
| :--- | :--- |
| **Type** | `Custom TCP` |
| **Port range** | `8080` |
| **Source** | `0.0.0.0/0` (for public access) |
| **Description** | *Optional: Java Login App* |

### Step 7: Useful Docker Commands

Keep these commands handy for managing your container:

| Task | Command |
| :--- | :--- |
| **Stop** the running container | `docker stop login-app` |
| **Remove** the container (when stopped) | `docker rm login-app` |
| **Rebuild** the image (after code changes) | `docker build -t my-login-app .` |
| **Restart** the container | `docker run -d --name login-app -p 8080:8080 my-login-app` |
| View **Logs** (last 200 lines) | `docker logs login-app -n 200` |
| **View Running Containers** | `docker ps` |

### Step 8: 1-Minute Troubleshooting

| Symptom | Cause & Fix |
| :--- | :--- |
| **404 on `/` (Root URL)** | The application is correctly deployed at `/ROOT/` but your code is at `/pages/login.jsp`. Use the `index.jsp` fix in **Step 5** or directly open `/pages/login.jsp`. |
| **404 on any page** | The WAR file might not be deployed correctly as `ROOT`. Check the contents of the deployed directory: `docker exec -it login-app ls /usr/local/tomcat/webapps/ROOT` |
| **Container running but page not loading** | **Security Group Blockage.** Go to **Step 6** and confirm Port 8080 is open to your IP or `0.0.0.0/0`. |
| **Container repeatedly exiting** | Check the logs for Java or Tomcat errors: `docker logs login-app` |

-----



## Java Login App Orchestration with Minikube (EC2)

This is a comprehensive, step-by-step guide to orchestrating your Java Login App using **Minikube** on an Ubuntu EC2 instance, covering the full flow from building the Docker image to accessing the application via Kubernetes.

This guide assumes you have already completed **Docker installation** and have the `Java-Login-App` project structure ready with the `Dockerfile`.

### Phase 1: Docker Setup and Preparation

#### Step 1: Build Docker Image

Use the existing `Dockerfile` to build your application image and tag it with your Docker Hub username and a version.

```bash
docker build -t harshitha/login-app:1.0 .
```

#### Step 2: Push to Docker Hub

You need to push the image to a publicly accessible repository (like Docker Hub) so that Kubernetes (Minikube) can pull it.

1.  **Log in to Docker Hub:**
    ```bash
    docker login
    ```
2.  **Push the image:**
    ```bash
    docker push harshitha/login-app:1.0
    ```

### Phase 2: Install and Start Minikube

Minikube is used here to create a local, single-node Kubernetes cluster on your EC2 instance.

#### Step 3: Install `kubectl` (Kubernetes Command-Line Tool)

`kubectl` is essential for managing the Kubernetes cluster.

1.  **Update system:**
    ```bash
    sudo apt update
    ```
2.  **Download latest stable `kubectl` binary:**
    ```bash
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    ```
3.  **Make it executable:**
    ```bash
    chmod +x kubectl
    ```
4.  **Move it to system PATH:**
    ```bash
    sudo mv kubectl /usr/local/bin/
    ```
5.  **Verify installation:**
    ```bash
    kubectl version --client
    ```

#### Step 4: Install and Start Minikube

Minikube requires Docker to be installed and running.

1.  **Install Minikube prerequisites (Docker is already installed):**
    ```bash
    sudo apt install -y docker.io
    ```
2.  **Download and install Minikube binary:**
    ```bash
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    ```
3.  **Start Minikube:** We will use the `docker` driver for simplicity on an EC2 instance.
    ```bash
    minikube start --driver=docker
    ```
    *(Wait for Minikube to initialize the cluster. This may take a few minutes.)*

### Phase 3: Kubernetes Deployment

#### Step 5: Create Deployment YAML

Create a file named `deployment.yaml`. This defines the Pods (containers) and ensures a specified number of replicas are running.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: login-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: login-app
  template:
    metadata:
      labels:
        app: login-app
    spec:
      containers:
      - name: login-app
        image: yourusername/login-app:1.0   # <-- Ensure this is your Docker Hub image tag
        ports:
        - containerPort: 8080
```

#### Step 6: Create Service YAML

Create a file named `service.yaml`. A Service provides a stable network endpoint for the Deployment. We use `NodePort` to expose the application outside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: login-app-service
spec:
  type: NodePort
  selector:
    app: login-app
  ports:
    - port: 8080       # Port the Service exposes inside the cluster
      targetPort: 8080 # Port the container is listening on (Tomcat)
      nodePort: 30080  # Port exposed on the EC2/Minikube Node (Must be between 30000-32767)
```

#### Step 7: Apply YAMLs to Kubernetes

Apply both the Deployment and the Service configurations to the Minikube cluster using `kubectl`.

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

**Check Status:**

Verify that the Pod and Service are running correctly:

```bash
kubectl get pods
kubectl get svc
```

  * `kubectl get pods` should show your Pod in a `Running` state.
  * `kubectl get svc` should show `login-app-service` with `NodePort` mapping `30080`.

### Phase 4: Access the Application

#### Step 8: Get Minikube IP

The Minikube service is running *inside* a virtual machine or container on your EC2 instance. You need its IP address to access the app.

```bash
minikube ip
```

*(Example output: `192.168.49.2`)*

#### Step 9: Open Application

Using the IP address from the previous step and the `nodePort` (`30080`) you defined in `service.yaml`, open the application in your browser:

```
http://<minikube-ip>:30080/pages/login.jsp
```

*Example:*
`http://192.168.49.2:30080/pages/login.jsp`

> **Note on EC2 Security Group:** Since Minikube runs *within* the EC2 instance, you must ensure the EC2 Security Group is open for port **30080** to allow external access.

#### Step 10: Verify Logs (Troubleshooting)

If the application does not load, check the container logs for errors:

```bash
kubectl logs deployment/login-app
```

-----

This is a comprehensive, step-by-step guide for implementing an error-free Ansible setup for a real-world DevOps project involving Jenkins and Kubernetes (Minikube).

The entire flow automates the setup of two distinct environments using Ansible Playbooks.

## Ansible Implementation for DevOps Project (Jenkins + Kubernetes)

This guide assumes you are working with two separate Ubuntu EC2 servers.

### 🧠 Overall Architecture

We will set up two servers with distinct responsibilities:

| Server | Role | Components Installed | Control Flow |
| :--- | :--- | :--- | :--- |
| **Server-1 (Control Node)** | **Jenkins + Ansible Server** | Ansible, Jenkins, Java 17, Docker | Executes Ansible Playbooks to manage **Server-2** |
| **Server-2 (Managed Node)** | **Kubernetes Server** | Docker, `kubectl`, Minikube | Managed remotely by Ansible from **Server-1** |

### 🗂️ Project Structure

Your GitHub repository root (`Java-Login-App/`) should have the following file structure. The `ansible/` folder contains all the configuration files and playbooks.

```
Java-Login-App/
├── ansible/
│   ├── inventory.ini
│   ├── ansible.cfg
│   ├── jenkins-setup.yml
│   ├── k8s-setup.yml
│   └── roles/
│       ├── common/
│       │   └── tasks/main.yml
│       ├── jenkins/
│       │   └── tasks/main.yml
│       └── kubernetes/
│           └── tasks/main.yml
└── README.md
```

### STEP 1: SSH Key Setup (Crucial for Ansible)

Ansible relies on SSH for communication. This step ensures passwordless SSH access from the Jenkins/Ansible Server to the Kubernetes Server.

1.  **On the Jenkins + Ansible server:**

      * Generate an SSH key pair (if you don't have one):
        ```bash
        ssh-keygen
        # Press Enter 3 times for default location and no passphrase
        ```
      * Copy the **public key**:
        ```bash
        cat ~/.ssh/id_rsa.pub
        ```
        Copy the entire output.

2.  **On the Kubernetes server:**

      * Edit the `authorized_keys` file for the `ubuntu` user:
        ```bash
        nano ~/.ssh/authorized_keys
        ```
      * Paste the public key copied from the Jenkins server into a new line.
      * Save and exit (`Ctrl+X`, then `Y`, then `Enter`).

3.  **Test SSH (from Jenkins server):**

    ```bash
    ssh ubuntu@<K8S_SERVER_IP>
    ```

    You must be able to log in without being prompted for a password. If this works, Ansible communication will be successful.

### STEP 2: Ansible Installation (Jenkins Server Only)

Install the Ansible software on the control node.

```bash
sudo apt update
sudo apt install ansible -y
```

**Verify:**

```bash
ansible --version
```

### STEP 3: `ansible.cfg` Configuration

Create the Ansible configuration file to define global settings.

**File:** `ansible/ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
remote_user = ubuntu
host_key_checking = False
private_key_file = ~/.ssh/id_rsa
```

  * `remote_user = ubuntu`: Specifies the default user for remote connections.
  * `private_key_file = ~/.ssh/id_rsa`: Points to the private key for SSH authentication.

### STEP 4: `inventory.ini` File

Define the target hosts (managed nodes) for Ansible.

**File:** `ansible/inventory.ini`

```ini
[jenkins]
jenkins ansible_host=127.0.0.1 ansible_connection=local

[kubernetes]
k8s ansible_host=<K8S_SERVER_PUBLIC_IP>
```

  * **`[jenkins]` Group:** Since Ansible and Jenkins run on the same server, we use `ansible_connection=local` to run tasks directly on the control node.
  * **`[kubernetes]` Group:** Replace `<K8S_SERVER_PUBLIC_IP>` with the public IP address of your Kubernetes EC2 server.

### STEP 5: COMMON Role

This role runs basic package updates and installs essential tools on both servers.

**File:** `ansible/roles/common/tasks/main.yml`

```yaml
- name: Update apt cache
  apt:
    update_cache: yes

- name: Install common packages
  apt:
    name:
      - git
      - curl
      - wget
    state: present
```

### STEP 6: JENKINS Role (Jenkins Server Setup)

This role installs Java, Docker, and Jenkins securely on the Jenkins server.

**File:** `ansible/roles/jenkins/tasks/main.yml`

```yaml
- name: Install Java 17
  ansible.builtin.apt:
    name: openjdk-17-jdk
    state: present

- name: Install Docker
  ansible.builtin.apt:
    name: docker.io
    state: present

- name: Start and enable Docker
  ansible.builtin.service:
    name: docker
    state: started
    enabled: yes

- name: Add ubuntu to docker group
  ansible.builtin.user:
    name: ubuntu
    groups: docker
    append: yes

- name: Install Jenkins dependencies
  ansible.builtin.apt:
    name:
      - ca-certificates
      - gnupg
    state: present

- name: Add Jenkins GPG key
  ansible.builtin.shell: |
    curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key |
    sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
  args:
    warn: false

- name: Add Jenkins repository
  ansible.builtin.shell: |
    echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" |
    sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
  args:
    warn: false

- name: Update apt
  ansible.builtin.apt:
    update_cache: yes

- name: Install Jenkins
  ansible.builtin.apt:
    name: jenkins
    state: present

- name: Start and enable Jenkins
  ansible.builtin.service:
    name: jenkins
    state: started
    enabled: yes
```

### STEP 7: KUBERNETES Role (Minikube Server Setup)

This role installs Docker, `kubectl`, and Minikube on the Kubernetes server.

**File:** `ansible/roles/kubernetes/tasks/main.yml`

```yaml
- name: Install Docker
  ansible.builtin.apt:
    name: docker.io
    state: present

- name: Start and enable Docker
  ansible.builtin.service:
    name: docker
    state: started
    enabled: yes

- name: Add ubuntu to docker group
  ansible.builtin.user:
    name: ubuntu
    groups: docker
    append: yes

- name: Install kubectl
  ansible.builtin.shell: |
    # Downloads stable kubectl and moves it to the PATH
    curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
    chmod +x kubectl
    mv kubectl /usr/local/bin/
  args:
    warn: false

- name: Install Minikube
  ansible.builtin.shell: |
    # Downloads latest minikube and moves it to the PATH
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    chmod +x minikube-linux-amd64
    mv minikube-linux-amd64 /usr/local/bin/minikube
  args:
    warn: false
```

### STEP 8: Playbooks

Create the two main playbooks that orchestrate the roles.

**File:** `ansible/jenkins-setup.yml`

```yaml
- name: Setup Jenkins Server
  hosts: jenkins
  become: yes
  roles:
    - common
    - jenkins
```

**File:** `ansible/k8s-setup.yml`

```yaml
- name: Setup Kubernetes Node
  hosts: kubernetes
  become: yes
  roles:
    - common
    - kubernetes
```

### STEP 9: Run Playbooks

Execute the playbooks from the **Jenkins + Ansible server** (inside the `ansible/` directory).

1.  **Setup Jenkins:**
    ```bash
    ansible-playbook jenkins-setup.yml
    ```
2.  **Setup Kubernetes:**
    ```bash
    ansible-playbook k8s-setup.yml
    ```

### FINAL VERIFICATION

#### Jenkins Server

1.  **Check Jenkins service status:**
    ```bash
    systemctl status jenkins
    ```
2.  **Access in Browser:**
    ```
    http://<JENKINS_SERVER_IP>:8080
    ```
    *(Ensure EC2 Security Group is open for port 8080)*

#### Kubernetes Server

1.  **Start Minikube (run this on the Kubernetes server after the playbook finishes):**
    ```bash
    minikube start --driver=docker
    ```
2.  **Check cluster status:**
    ```bash
    kubectl get nodes
    ```
    The node should show a status of `Ready`.

### Interview Explanation

| Question | Answer |
| :--- | :--- |
| **Why Ansible?** | Manual installations are error-prone and time-consuming. Ansible ensures highly repeatable, automated, and consistent setups across multiple environments, adhering to the principle of Infrastructure as Code (IaC). |
| **Why roles?** | Roles provide a structured, standardized way to organize tasks, variables, and handlers. This promotes **reusability** (e.g., the `common` role) and makes the playbooks clean, readable, and easier to manage—a core DevOps best practice. |

-----


Understood.
Here is the **same documentation**, rewritten **without any emojis**, in a **clean, professional, step-by-step format** suitable for **notes, GitHub README, or interview preparation**.

---

# Kubernetes Cluster Setup Using kubeadm

## End-to-End Step-by-Step Documentation (Zero to Application Access)

---

## 1. Architecture Overview

This setup creates a basic production-like Kubernetes cluster using kubeadm.

### Cluster Components

* One Master (Control Plane) node
* One Worker node
* Operating System: Ubuntu 20.04 or 22.04
* Container Runtime: containerd
* CNI Plugin: Calico
* Application: Java (Tomcat-based)
* Service Exposure: NodePort

---

## 2. Why kubeadm Instead of minikube

* minikube is intended only for local development
* kubeadm uses real Kubernetes components
* kubeadm setup closely matches managed Kubernetes services like EKS
* Suitable for DevOps learning and interviews

---

## 3. Prerequisites (Run on Both Master and Worker Nodes)

### 3.1 Disable Swap

Kubernetes requires swap to be disabled for predictable scheduling.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Verify:

```bash
free -h
```

---

### 3.2 Load Required Kernel Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Make permanent:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

---

### 3.3 Configure Kernel Networking Parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply changes:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

Expected output:

```
= 1
```

---

## 4. Install Container Runtime (containerd)

### 4.1 Install containerd

```bash
sudo apt update
sudo apt install -y containerd
```

---

### 4.2 Generate Default containerd Configuration

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Restart and enable:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 5. Install Kubernetes Components (Both Nodes)

### 5.1 Install Required Packages

```bash
sudo apt install -y apt-transport-https ca-certificates curl
```

---

### 5.2 Add Kubernetes Repository

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

---

### 5.3 Install kubeadm, kubelet, and kubectl

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 6. Master Node Setup

### 6.1 Initialize the Kubernetes Cluster

Run only on the master node.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Save the kubeadm join command for the worker node.

---

### 6.2 Configure kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```bash
kubectl cluster-info
```

---

## 7. Install Calico CNI

Calico enables pod-to-pod networking.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

Verify:

```bash
kubectl get pods -n kube-system
```

All pods should be in the Running state.

---

## 8. Worker Node Setup

### 8.1 Join the Cluster

Run only on the worker node.

```bash
sudo kubeadm join <MASTER-IP>:6443 \
--token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```

---

## 9. Verify Cluster Status

Run on the master node.

```bash
kubectl get nodes
```

Both nodes should show status Ready.

---

## 10. Deploy Java Application (Master Node Only)

### 10.1 Clone the Application Repository

```bash
git clone https://github.com/<your-repository>.git
cd Java-Login-App/k8s
```

---

### 10.2 Apply Kubernetes Manifests

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

### 10.3 Verify Deployment

```bash
kubectl get pods -o wide
kubectl get svc
```

Confirm that the service type is NodePort.

---

## 11. Expose Application Using NodePort

### 11.1 Configure EC2 Security Group

Open the NodePort on the worker node.

Inbound rule example:

* Protocol: TCP
* Port: 30080
* Source: 0.0.0.0/0

---

### 11.2 Access the Application

Java Tomcat applications do not run on the root path.

Use the correct context path.

```text
http://<WORKER_PUBLIC_IP>:30080/<context-path>
```

Example:

```text
http://13.xxx.xxx.xxx:30080/login
```

---

## 12. Common Errors and Fixes

### kubectl localhost:8080 refused

* Cause: kubectl executed on worker node
* Fix: run kubectl only on master node

---

### NodePort not accessible externally

* Cause: EC2 security group does not allow NodePort
* Fix: open NodePort or 30000–32767 range

---

### HTTP 404 Not Found

* Cause: wrong application context path
* Fix: use the correct Tomcat context path

---
