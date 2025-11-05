# Jenkins-SonarQube-Artifactory-Ansible

## 🚀 CI/CD Pipeline for PHP TODO Application

**Jenkins → SonarQube → Artifactory → Ansible**

This project demonstrates an end-to-end **Continuous Integration and Continuous Deployment (CI/CD)** pipeline for a PHP-based TODO application.
The pipeline automates **build, test, code quality analysis, artifact management, and deployment** using modern DevOps tools.

---

## 🧭 Pipeline Overview

```text
Developer → GitHub → Jenkins → SonarQube (Quality Gate)
                                   ↓
                            Artifactory (Artifact Repo)
                                   ↓
                             Ansible (Deployment)
                                   ↓
                            Web/App Server (Dev/Prod)
```

---

## ⚙️ Prerequisites

Ensure you have the following installed and running:

| Component         | Purpose                                  | Example Host                                        |
| ----------------- | ---------------------------------------- | --------------------------------------------------- |
| Jenkins           | CI/CD Orchestration                      | `jenkins.example.com`                               |
| JFrog Artifactory | Artifact Repository                      | `artifactory.example.com`                           |
| SonarQube         | Code Quality & Security                  | `sonarqube.example.com`                             |
| Ansible           | Configuration Management & Deployment    | `ansible.example.com`                               |
| PHP + Composer    | Application Runtime & Dependency Manager | —                                                   |
| GitHub Repo       | Source Control                           | [php-todo](https://github.com/StegTechHub/php-todo) |

---

## 🧩 Section 1 — Jenkins Setup and Configuration

### Step 1: Install Required Packages

```bash
sudo apt update
sudo apt install -y zip libapache2-mod-php phploc composer \
php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip}
```

### Step 2: Install Jenkins Plugins

Install the following from **Manage Jenkins → Plugin Manager → Available Plugins**:

* **Plot Plugin**
* **JFrog Artifactory Plugin**
* **SonarQube Scanner for Jenkins**

---

## 🏗️ Section 2 — SonarQube Installation and Integration with Jenkins

### Step 1: Install SonarQube (Ubuntu 20.04+)

```bash
# Install Java
sudo apt update && sudo apt install openjdk-11-jdk -y

# Create a dedicated user
sudo adduser sonar
sudo su - sonar

# Download and extract SonarQube
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip
unzip sonarqube-9.9.0.65466.zip
mv sonarqube-9.9.0.65466 /opt/sonarqube
```

### Step 2: Configure SonarQube

Edit `/opt/sonarqube/conf/sonar.properties`:

```bash
sonar.jdbc.username=sonar
sonar.jdbc.password=StrongPassword
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
```

Start the service:

```bash
sudo /opt/sonarqube/bin/linux-x86-64/sonar.sh start
```

Access UI at:
👉 `http://<sonarqube-ip>:9000`
Login as `admin/admin`, change password, and create a **Project Token**.

---

### Step 3: Integrate SonarQube with Jenkins

In Jenkins:

1. **Manage Jenkins → Configure System → SonarQube Servers**
2. Add:

   * **Server name:** SonarQube
   * **Server URL:** `http://<sonarqube-ip>:9000`
   * **Token:** Use the generated project token
3. **Manage Jenkins → Global Tool Configuration → SonarQube Scanner**

   * Add SonarQube Scanner and name it `SonarScanner`.

---

## 🧱 Section 3 — Artifactory Configuration and Integration

### Step 1: Install Artifactory OSS

```bash
wget https://releases.jfrog.io/artifactory/artifactory-debs/pool/jfrog-artifactory-oss-7.x.deb
sudo dpkg -i jfrog-artifactory-oss-7.x.deb
sudo systemctl enable artifactory
sudo systemctl start artifactory
```

Access at:
👉 `http://<artifactory-ip>:8081`

Default credentials: `admin / password`

---

### Step 2: Create a Local Repository

1. Navigate to **Admin → Repositories → Local → New Local Repository**
2. Choose **Package Type:** `Generic`
3. Name it: `php-todo-releases`

This will host your build artifacts.

---

### Step 3: Generate API Key

In Artifactory UI:
**User Profile → Generate API Key / Access Token**

Test it:

```bash
curl -u admin:<API_KEY> "http://<artifactory-ip>:8081/artifactory/api/repositories"
```

---

### Step 4: Configure Jenkins → Artifactory Connection

In Jenkins:

* Go to **Manage Jenkins → Configure System → JFrog Artifactory**
* Add:

  * **Server ID:** `artifactory-server`
  * **URL:** `http://<artifactory-ip>:8081/artifactory`
  * **Credentials:** admin/API_KEY
* ✅ Click **Test Connection**

---

## ⚙️ Section 4 — Jenkins Pipeline (php-todo)

Create a Jenkinsfile in the project root:

```groovy
pipeline {
    agent any

    stages {
        stage('Initial Cleanup') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Source') {
            steps {
                git branch: 'main', url: 'https://github.com/StegTechHub/php-todo.git'
            }
        }

        stage('Prepare Dependencies') {
            steps {
                sh 'mv .env.sample .env'
                sh 'composer install'
                sh 'php artisan migrate'
                sh 'php artisan db:seed'
                sh 'php artisan key:generate'
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh './vendor/bin/phpunit'
            }
        }

        stage('Code Analysis') {
            steps {
                sh 'mkdir -p build/logs'
                sh 'phploc app/ --log-csv build/logs/phploc.csv'
            }
        }

        stage('SonarQube Quality Scan') {
            environment {
                scannerHome = tool 'SonarScanner'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=php-todo -Dsonar.sources=app'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package Artifact') {
            steps {
                sh 'zip -qr php-todo.zip *'
            }
        }

        stage('Publish Artifact to Artifactory') {
            steps {
                script {
                    def server = Artifactory.server 'artifactory-server'
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "php-todo.zip",
                                "target": "php-todo-releases/${env.BUILD_NUMBER}/"
                            }
                        ]
                    }"""
                    server.upload(uploadSpec)
                    server.publishBuildInfo()
                }
            }
        }

        stage('Deploy via Ansible') {
            steps {
                build job: 'ansible-project/main', parameters: [
                    [$class: 'StringParameterValue', name: 'env', value: 'dev']
                ], propagate: false, wait: true
            }
        }
    }
}
```

---

## 🪄 Section 5 — Ansible Deployment Setup

Your Ansible project should have a role that:

1. Downloads the latest artifact from Artifactory.
2. Extracts it into `/var/www/php-todo/`.
3. Restarts Apache or Nginx.

Example Ansible playbook:

```yaml
- hosts: web
  become: yes
  tasks:
    - name: Download latest build
      get_url:
        url: "http://<artifactory-ip>:8081/artifactory/php-todo-releases/{{ build_number }}/php-todo.zip"
        dest: /tmp/php-todo.zip
        url_username: admin
        url_password: "{{ artifactory_token }}"

    - name: Unpack application
      unarchive:
        src: /tmp/php-todo.zip
        dest: /var/www/html/php-todo
        remote_src: yes

    - name: Restart Apache
      service:
        name: apache2
        state: restarted
```

---

## 📈 Section 6 — Plot Code Coverage and Metrics

Configure Jenkins **Plot Plugin** to visualize `build/logs/phploc.csv` data:

* Go to **Job → Configure → Add Post-build Action → Plot build data**
* Add series from `build/logs/phploc.csv`
* Select **Line Graph**
* Titles: “Code Complexity”, “Lines of Code”, etc.

---

## ✅ Section 7 — End-to-End Workflow

| Stage | Tool            | Description                                             |
| ----- | --------------- | ------------------------------------------------------- |
| 1     | **Jenkins**     | Clones repo, runs tests, scans code                     |
| 2     | **SonarQube**   | Performs static code analysis & applies Quality Gate    |
| 3     | **Artifactory** | Stores versioned build artifacts                        |
| 4     | **Ansible**     | Deploys the packaged artifact to the target environment |

---

## 🧩 Section 8 — Final Pipeline Flow

```text
GitHub Repo
   ↓
Jenkins (Build, Test, Analyze)
   ↓
SonarQube (Quality Gate)
   ↓
Artifactory (Artifact Upload)
   ↓
Ansible (Deploy from Artifactory)
   ↓
Dev/Prod Server (Running PHP TODO App)
```

---

## 🎯 Key Benefits

* **Automated testing & static analysis** ensures code quality.
* **Immutable builds** stored in Artifactory guarantee reproducibility.
* **One-click deployment** through Ansible provides consistent releases.
* **Full visibility** via Jenkins dashboard and plots.

