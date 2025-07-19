# Jenkins-Project

Here's a **complete Jenkins pipeline** that includes **automated testing**, **code analysis**, **artifact build**, and optionally **deployment**—all within a CI/CD setup.

---

## ✅ **Use Case**

We'll assume:

* Language: **Python**
* Test framework: **pytest**
* Code quality: **SonarQube**
* Artifact: Docker image (optional)
* Deployment: Staged or manual (optional)

---

## 📄 Jenkinsfile – Full Pipeline with Testing

```groovy
pipeline {
    agent any

    environment {
        // Global environment variables
        PYTHON_VERSION = '3.10'
        VENV_DIR = '.venv'
        SONARQUBE_SERVER = 'SonarQubeServer' // Jenkins -> Manage Jenkins -> Configure Systems
        SONAR_PROJECT_KEY = 'my-python-app'
    }

    tools {
        python "${PYTHON_VERSION}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Setup Virtual Env & Install Dependencies') {
            steps {
                sh """
                    python -m venv ${VENV_DIR}
                    source ${VENV_DIR}/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                """
            }
        }

        stage('Run Unit Tests') {
            steps {
                sh """
                    source ${VENV_DIR}/bin/activate
                    pytest tests/ --junitxml=test-results/results.xml --cov=src/ --cov-report=xml
                """
            }
            post {
                always {
                    junit 'test-results/results.xml'
                }
            }
        }

        stage('SonarQube Code Analysis') {
            environment {
                // Required SonarScanner credentials setup in Jenkins
                SONAR_SCANNER_OPTS = "-Dsonar.projectKey=${SONAR_PROJECT_KEY}"
            }
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                        source ${VENV_DIR}/bin/activate
                        sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=src \
                            -Dsonar.python.coverage.reportPaths=coverage.xml
                    """
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { fileExists('Dockerfile') }
            }
            steps {
                sh """
                    docker build -t my-python-app:${BUILD_NUMBER} .
                """
            }
        }

        stage('Push to Registry') {
            when {
                expression { return params.DEPLOY_TO_REGISTRY }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred-id', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag my-python-app:${BUILD_NUMBER} myrepo/my-python-app:${BUILD_NUMBER}
                        docker push myrepo/my-python-app:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy (Optional)') {
            when {
                expression { return params.TRIGGER_DEPLOYMENT }
            }
            steps {
                echo "Triggering deployment script or job..."
                // sh ./deploy.sh OR build job: 'Deploy-Job'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Sending notifications...'
            // Slack/email notification integration if needed
        }
    }
}
```

---

## ✅ **How to Use**

### 🛠️ Prerequisites in Jenkins:

* Install plugins:

  * **Pipeline**
  * **Python**
  * **JUnit**
  * **SonarQube Scanner**
  * **Docker Pipeline** (optional)
* Configure:

  * **SonarQube server** under `Manage Jenkins > Configure System`
  * Docker registry credentials (ID: `docker-cred-id`)
  * Optional: GitHub or Bitbucket webhook trigger

---

## 📦 Folder Structure (Recommended)

```
project/
├── Jenkinsfile
├── requirements.txt
├── Dockerfile
├── src/
│   └── app.py
├── tests/
│   └── test_app.py
```

---

## 🔍 Add-ons (Optional)

* **Slack notifications**: in `post` block
* **Docker Compose testing**: before deployment stage
* **GitHub PR check integration**: Jenkins + GitHub webhook

---

