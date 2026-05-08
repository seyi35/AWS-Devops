# CI/CD Pipeline with Jenkins, Nexus, SonarQube & AWS

---

## Project Overview

This project demonstrates a complete CI/CD pipeline using Jenkins to automate the build, test, code quality analysis, and artifact publishing of a Java web application. It reflects a real-world DevOps workflow where every code change is automatically validated, analysed for quality, and stored as a versioned artifact — with the team notified via Slack at every stage.

> **Note:** The pipeline currently covers the CI portion of the workflow (source → build → test → quality gate → artifact storage). Docker, AWS ECR, and ECS deployment stages are planned for a future phase.

---

## Flow of Execution

```
Jenkins Setup → Nexus Setup → SonarQube Setup → Security Groups → 
Install Plugins → Integrate Nexus & SonarQube → Write Pipeline → Set Notifications
```

---

## Architecture Diagram

> *Figure 1: CI/CD Pipeline Architecture — GitHub → Jenkins → SonarQube → Nexus → Slack*

*(Insert architecture diagram here)*

---

## Tools Used

| Tool | Purpose |
|---|---|
| Jenkins | CI/CD orchestration and pipeline execution |
| SonarQube | Static code analysis and quality gate enforcement |
| Nexus Repository | Versioned artifact storage |
| Maven | Build automation and dependency management |
| Slack | Build notifications |
| AWS EC2 | Hosting for Jenkins, Nexus, and SonarQube servers |

---

## Pre-requisites

- AWS Account with EC2 permissions
- GitHub account with a repository containing your application code
- Slack workspace with admin access
- Basic familiarity with Linux CLI and SSH

---

## Infrastructure Setup

### Key Pair

Before launching any EC2 instances, create an SSH key pair in the AWS Console under **EC2 → Key Pairs**. Name it `vprofile-ci-key` and download the `.pem` file to your local machine.

> ⚠️ Store the `.pem` file securely and never commit it to version control.

---

### Security Groups

Create the following three security groups before launching any instances.

#### Jenkins Security Group — `jenkins-SG`

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | My IP |
| Custom TCP | TCP | 8080 | Anywhere (IPv4 + IPv6) |

> Port 8080 must be open to all IPs because GitHub webhook IPs are not static. In production, consider restricting to [GitHub's published IP ranges](https://api.github.com/meta).

#### Nexus Security Group — `nexus-SG`

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | My IP |
| Custom TCP | TCP | 8081 | My IP, `jenkins-SG` |

#### SonarQube Security Group — `sonar-SG`

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | My IP |
| HTTP | TCP | 80 | My IP, `jenkins-SG` |

> After creating `sonar-SG`, go back to `jenkins-SG` and add an additional inbound rule: **allow port 8080 from `sonar-SG`**. This allows SonarQube to send analysis results back to Jenkins via the configured webhook.

---

### EC2 Instances

#### Jenkins Server

| Parameter | Value |
|---|---|
| Name | `jenkins-server` |
| AMI | Ubuntu 20.04 |
| Instance Type | `t2.medium` |
| Key Pair | `vprofile-ci-key` |
| Security Group | `jenkins-SG` |

> Attach the Jenkins userdata script at launch. See `scripts/jenkins-userdata.sh` in this repository.

#### Nexus Server

| Parameter | Value |
|---|---|
| Name | `nexus-server` |
| AMI | Amazon Linux 2 |
| Instance Type | `t2.medium` |
| Key Pair | `vprofile-ci-key` |
| Security Group | `nexus-SG` |

> Attach the Nexus userdata script at launch. See `scripts/nexus-userdata.sh` in this repository.

#### SonarQube Server

| Parameter | Value |
|---|---|
| Name | `sonar-server` |
| AMI | Ubuntu 22.04 |
| Instance Type | `t2.medium` |
| Key Pair | `vprofile-ci-key` |
| Security Group | `sonar-SG` |

> Attach the SonarQube userdata script at launch. See `scripts/sonar-userdata.sh` in this repository.

---

## Post-Installation Setup

### Jenkins

1. SSH into the Jenkins server:
   ```bash
   sudo -i
   systemctl status jenkins
   cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

2. Open `http://<jenkins_public_ip>:8080` in your browser. Paste the initial admin password when prompted.

3. Select **Install suggested plugins** and wait for installation to complete.

4. Create your first admin user when prompted.

5. Install the following additional plugins via **Manage Jenkins → Plugins → Available**:

   - Maven Integration
   - GitHub Integration
   - Nexus Artifact Uploader
   - SonarQube Scanner
   - Slack Notification
   - Build Timestamp
   - Pipeline: AWS Steps

6. Configure Build Timestamp under **Manage Jenkins → Configure System → Build Timestamp**. Set the pattern to:
   ```
   yy-MM-dd_HH-mm-ss
   ```
   > Use hyphens instead of colons to avoid file naming issues on some systems.

---

### Nexus

1. SSH into the Nexus server and confirm the service is running:
   ```bash
   sudo -i
   systemctl status nexus
   ```

2. Open `http://<nexus_public_ip>:8081` in your browser and click **Sign In**.

3. Retrieve the initial admin password:
   ```bash
   cat /opt/nexus/sonatype-work/nexus3/admin.password
   ```

4. Sign in with username `admin` and the password above. Set a new password and select **Disable Anonymous Access**.

5. Create the following repositories via the **gear icon → Repositories → Create repository**:

   **Hosted Repository** (stores your release artifacts):
   - Format: `maven2 (hosted)`
   - Name: `vprofile-repo`
   - Version Policy: Release

   **Proxy Repository** (caches Maven Central dependencies):
   - Format: `maven2 (proxy)`
   - Name: `vpro-maven-central`
   - Remote Storage URL: `https://repo1.maven.org/maven2/`

   **Group Repository** (single URL that serves all repos):
   - Format: `maven2 (group)`
   - Name: `vpro-maven-group`
   - Member Repositories: `vpro-maven-central`, `vprofile-repo`

6. Add Nexus credentials in Jenkins under **Manage Jenkins → Credentials → Global → Add Credentials**:
   - Kind: Username with password
   - Username: `admin`
   - Password: `<your nexus password>`
   - ID: `nexuslogin`

---

### SonarQube

1. Open `http://<sonar_public_ip>` in your browser. Log in with username `admin` and password `admin`. You will be prompted to set a new password on first login.

2. **Generate an authentication token** for Jenkins integration:
   - Click your profile icon (top right) → **My Account** → **Security**
   - Enter a token name (e.g., `jenkins-token`) and click **Generate**
   - Copy and store the token immediately — it will not be displayed again

3. **Create a Quality Gate** to define the code standards your project must meet:
   - Go to **Quality Gates → Create**. Give it a name (e.g., `vprofile-QG`)
   - Click **Add Condition** → select a metric such as *Bugs* → set the threshold (e.g., greater than 80)
   - Click **Save**
   - Navigate to your project → **Project Settings → Quality Gate** → select the gate you just created

   > Quality Gates define the minimum standards a project must meet to be considered releasable. If the pipeline produces code that exceeds the configured thresholds — for example, too many bugs or insufficient test coverage — the quality gate will fail and the Jenkins pipeline will be aborted automatically.

4. **Create a Webhook** so SonarQube can send analysis results back to Jenkins:
   - Go to **Administration → Configuration → Webhooks → Create**
   - Name: `jenkins`
   - URL: `http://<jenkins_private_ip>:8080/sonarqube-webhook/`
   - Click **Create**

   > Without this webhook, Jenkins would not know when the SonarQube analysis has finished or whether it passed the quality gate.

5. **Integrate SonarQube Scanner in Jenkins:**

   Install the scanner tool:
   - Go to **Manage Jenkins → Global Tool Configuration → SonarQube Scanner → Add SonarQube Scanner**
   - Name: `sonar6.2`
   - Enable **Install automatically**

   Add the SonarQube server:
   - Go to **Manage Jenkins → Configure System → SonarQube Servers → Add SonarQube**
   - Name: `sonarserver`
   - Server URL: `http://<sonar_private_ip>` *(use the private IP)*
   - Under **Server Authentication Token**, click **Add → Jenkins**
   - Kind: Secret Text
   - Secret: paste the token generated in step 2
   - ID: `sonartoken`
   - Select `sonartoken` from the dropdown and click **Save**

   > Confirm that the `sonar-SG` security group allows inbound traffic on port 80 from `jenkins-SG`. Without this rule, Jenkins will be unable to upload analysis results to SonarQube.

---

## Pipeline Stages

> The complete `Jenkinsfile` is available at `jenkins/Jenkinsfile` in this repository. Each stage is explained below.

---

### Stage 1 — Fetch

The `tools` block at the top of the pipeline ensures that Maven 3.9 and JDK 17 are available in the build environment before any stage runs. This stage then checks out the source code from the specified GitHub repository and branch into the Jenkins workspace.

```groovy
pipeline {
    agent any
    tools {
        maven 'MAVEN3.9'
        jdk 'JDK17'
    }

    stages {

        stage('Fetch') {
            steps {
                echo 'Fetching code...'
                git branch: 'atom', url: 'https://github.com/seyi35/proton.git'
            }
        }
```

---

### Stage 2 — Build and Archive Artifact

Maven compiles the project and packages it as a `.war` file. Unit tests are intentionally skipped at this stage since they run as a dedicated step in Stage 3. If the build succeeds, the artifact is archived in Jenkins for traceability and fingerprinting. If the build fails, the pipeline logs the failure and stops.

```groovy
        stage('Build') {
            steps {
                echo 'Building the project...'
                sh 'mvn install -DskipTests' // Tests run separately in Stage 3
            }
            post {
                success {
                    echo '✅ Build successful! Archiving artifacts...'
                    archiveArtifacts artifacts: '**/*.war', fingerprint: true
                }
                failure {
                    echo '❌ Build failed!'
                }
            }
        }
```

---

### Stage 3 — Unit Tests & Checkstyle Analysis

The **Unit Test** stage runs all JUnit tests using `mvn test` and generates test result reports under `target/surefire-reports`. These reports confirm that the application logic behaves as expected.

The **Checkstyle** stage performs static analysis to verify that the code conforms to the project's defined coding style rules. It produces a report at `target/checkstyle-result.xml`. Both reports are uploaded to SonarQube in Stage 4 to contribute to the overall code quality view.

```groovy
        stage('Unit Test') {
            steps {
                echo 'Running unit tests...'
                sh 'mvn test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                echo 'Running Checkstyle...'
                sh 'mvn checkstyle:checkstyle'
            }
        }
```

---

### Stage 4 — SonarQube Code Analysis

This stage runs a comprehensive static code analysis using the SonarQube Scanner. It uploads the source code, compiled binaries, unit test results, code coverage data (JaCoCo), and Checkstyle results to the SonarQube server. The `withSonarQubeEnv` block automatically injects the server URL and authentication token configured in Jenkins, so no credentials are hardcoded in the pipeline.

```groovy
        stage('SonarQube Code Analysis') {
            environment {
                scannerHome = tool 'sonar6.2'
            }
            steps {
                withSonarQubeEnv('sonarserver') {
                    echo 'Running SonarQube analysis...'
                    sh '''
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=Vprofile \
                        -Dsonar.projectName=Vprofile \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest \
                        -Dsonar.junit.reportPaths=target/surefire-reports \
                        -Dsonar.jacoco.reportPaths=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    '''
                }
            }
        }
```

---

### Stage 5 — Quality Gate

Jenkins pauses and waits for SonarQube to complete its analysis and return a pass/fail result via the configured webhook. If the quality gate fails — meaning the code does not meet the defined thresholds for bugs, vulnerabilities, or coverage — the pipeline is aborted. The one-hour timeout prevents the build from hanging indefinitely in the event of a connectivity issue.

```groovy
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
```

---

### Stage 6 — Upload Artifact to Nexus

This stage uploads the packaged `.war` artifact to the Nexus hosted repository (`vprofile-repo`). Each artifact is versioned using the Jenkins build ID combined with a timestamp, ensuring every build produces a unique, traceable artifact and preventing version conflicts in the repository. The `nexuslogin` credential ID must be configured in Jenkins credentials beforehand (see [Nexus Setup](#nexus) above).

```groovy
        stage('Upload Artifact') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '<NEXUS_PRIVATE_IP>:8081',
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'vprofile-repo',
                    credentialsId: 'nexuslogin',
                    artifacts: [
                        [
                            artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war'
                        ]
                    ]
                )
            }
        }

    } // end stages
} // end pipeline
```

---

## Slack Notifications

### Workspace Setup

1. Log in to Slack and navigate to your workspace. Create a channel named `#devopscicd`.
2. Go to [Slack App Directory](https://slack.com/apps) and search for **Jenkins CI**. Click **Add to Slack**, select the `#devopscicd` channel, and follow the setup instructions. Copy the **Integration Token** provided at the end.

### Configure Jenkins

Go to **Manage Jenkins → Configure System → Slack** and enter the following:

| Field | Value |
|---|---|
| Workspace | Your Slack subdomain (e.g., `myteam` from `myteam.slack.com`) |
| Default Channel | `#devopscicd` |
| Credential | `slacktoken` (see below) |

Add the Slack token to Jenkins credentials:
- Go to **Manage Jenkins → Credentials → Global → Add Credentials**
- Kind: Secret Text
- Secret: `<paste your Slack integration token>`
- ID: `slacktoken`

### Notification Logic

A `COLOR_MAP` is defined at the top of the pipeline to associate each build result with a corresponding Slack message color. This map is referenced in the `post` block so notification colors are set dynamically based on the actual pipeline outcome.

```groovy
def COLOR_MAP = [
    'SUCCESS'  : 'good',
    'FAILURE'  : 'danger',
    'UNSTABLE' : 'warning',
    'ABORTED'  : 'danger'
]

// Add this post block outside the stages block, before the final closing brace:

post {
    always {
        slackSend(
            channel: '#devopscicd',
            color: COLOR_MAP[currentBuild.currentResult] ?: 'danger',
            message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' #${env.BUILD_NUMBER}\n${env.BUILD_URL}"
        )
    }
}
```

> Using `always` with a dynamic color map sends a single, consistent notification for every build outcome — whether it passes, fails, or is aborted.

---

## Running the Pipeline

1. Go to your Jenkins job → **Configure** → under **Pipeline**, set the Definition to **Pipeline script from SCM**.
2. Select **Git** as the SCM.
3. Enter your repository URL and set the branch name.
4. Set the Script Path to `jenkins/Jenkinsfile`.
5. Click **Save**, then **Build Now** to trigger the first run.

> Using **Pipeline script from SCM** is strongly recommended over pasting the script directly into Jenkins. It keeps your pipeline version-controlled alongside your application code, making changes auditable and rollback straightforward.

---

## Repository Structure

```
├── jenkins/
│   └── Jenkinsfile
├── scripts/
│   ├── jenkins-userdata.sh
│   ├── nexus-userdata.sh
│   └── sonar-userdata.sh
├── src/
│   └── (application source code)
└── README.md
```

---

## Future Improvements

- [ ] Add Docker build and image tagging stage
- [ ] Push Docker image to AWS Elastic Container Registry (ECR)
- [ ] Deploy to AWS Elastic Container Service (ECS) via pipeline
- [ ] Parameterise environment variables (branch, Nexus URL, SonarQube key) for multi-environment support
- [ ] Add automated rollback on ECS deployment failure
