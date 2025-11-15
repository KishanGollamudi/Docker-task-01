# 🚀 **Complete CI/CD Pipeline – Jenkins, SonarQube, Nexus, Docker, DockerHub, Tomcat Deployment**

This repository demonstrates a **full end-to-end DevOps CI/CD pipeline** for a Java Web Application (WAR) deployed on **Tomcat** using Docker.
Pipeline includes:

✔ Jenkins (running inside Docker)
✔ SonarQube (static code analysis)
✔ Nexus Repository (artifact storage)
✔ Docker (image building & execution)
✔ DockerHub (image registry)
✔ Tomcat (runtime deployment)
✔ GitHub (source code)

Everything is deployed on an **AWS EC2 instance**.

---

# 🏗️ **1. EC2 Instance Setup**

### ✔ Instance Type: `t2.large`

### ✔ OS: Ubuntu 22.04

### ✔ Minimum Storage: 16GB

### ✔ Open Ports:

| Port | Usage                    |
| ---- | ------------------------ |
| 22   | SSH                      |
| 8080 | Jenkins                  |
| 8081 | Nexus                    |
| 9000 | SonarQube                |
| 8082 | Tomcat App Deployment    |
| 2375 | Docker Engine (optional) |

---

# 🐳 **2. Install Docker on EC2**

```bash
sudo apt update -y
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

Logout & login again.

---

# 🎛️ **3. Run SonarQube in Docker**

SonarQube requires good RAM (2GB+).

```bash
docker run -d --name sonar \
  -p 9000:9000 \
  sonarqube:lts
```

Access:

```
http://<EC2_PUBLIC_IP>:9000
```

---

# 🧰 **4. Run Nexus Repository**

```bash
docker run -d --name nexus \
  -p 8081:8081 \
  sonatype/nexus3
```

Access:

```
http://<EC2_PUBLIC_IP>:8081
```

Retrieve initial password:

```bash
docker exec -it nexus cat /nexus-data/admin.password
```

---

# 🛠️ **5. Build Custom Jenkins Image with Docker Installed**

### Create a directory:

```bash
mkdir jenkins-docker
cd jenkins-docker
```

### Add **Dockerfile**:

```dockerfile
FROM jenkins/jenkins:lts

USER root

RUN apt-get update && \
    apt-get install -y docker.io

RUN groupadd -g 999 docker || true
RUN usermod -aG docker jenkins

USER jenkins
```

### Build the image:

```bash
docker build -t jenkins-docker:latest .
```

---

# 🧩 **6. Run Jenkins Container**

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins-docker:latest
```

Retrieve initial password:

```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Access:

```
http://<EC2_PUBLIC_IP>:8080
```

---

# 🔧 **7. Jenkins Configuration**

### Install Plugins:

* Git plugin
* Pipeline plugin
* SonarQube Scanner
* Docker Pipeline
* Nexus Artifact Uploader (optional)

### Configure Tools:

**Manage Jenkins → Global Tool Configuration**

#### Maven

* Name: `Maven-3`
* Install automatically

---

# 🔐 **8. Add Required Jenkins Credentials**

| ID             | Type        | Usage                   |
| -------------- | ----------- | ----------------------- |
| sonar-token    | Secret Text | SonarQube token         |
| nexus          | User/Pass   | Nexus admin credentials |
| dockerhub-user | User/Pass   | DockerHub login         |

---

# 🗳️ **9. Update `pom.xml` for Nexus (distributionManagement)**

```xml
<distributionManagement>
    <repository>
        <id>nexus</id>
        <url>http://<EC2_PUBLIC_IP>:8081/repository/maven-releases/</url>
    </repository>
</distributionManagement>
```

---

# 🔧 **10. Jenkinsfile (CI/CD Pipeline Script)**

```groovy
pipeline {
    agent any

    environment {
        SONARQUBE_URL = 'http://sonar:9000'
        NEXUS_URL     = 'http://nexus:8081'
        DOCKER_IMAGE  = "kishangollamudi/onlinebookstore"
        VERSION       = "${env.BUILD_NUMBER}"
    }

    tools {
        maven 'Maven-3'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        mvn sonar:sonar \
                        -Dsonar.projectKey=onlinebookstore \
                        -Dsonar.host.url=${SONARQUBE_URL} \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {

                    sh '''
                    echo "<settings>
                            <servers>
                                <server>
                                    <id>nexus</id>
                                    <username>${NEXUS_USER}</username>
                                    <password>${NEXUS_PASS}</password>
                                </server>
                            </servers>
                        </settings>" > settings.xml
                    '''

                    sh 'mvn deploy -DskipTests -s settings.xml'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${VERSION}")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-user', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh """
                        echo ${DH_PASS} | docker login -u ${DH_USER} --password-stdin
                        docker push ${DOCKER_IMAGE}:${VERSION}
                        docker tag ${DOCKER_IMAGE}:${VERSION} ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

    }

    post {
        always {
            cleanWs()
        }
    }
}
```

---

# 🐳 **11. Tomcat Dockerfile (App Deployment)**

```dockerfile
FROM tomcat:9-jdk17

RUN rm -rf /usr/local/tomcat/webapps/*

COPY target/onlinebookstore.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
```

---

# 🚀 **12. Deploy Application Container**

```bash
docker run -d -p 8082:8080 --name bookstore \
  kishangollamudi/onlinebookstore:latest
```

Open in browser:

```
http://<EC2_PUBLIC_IP>:8082
```

---

# 🎉 **Final Outcome**

You now have a **complete CI/CD pipeline**:

✔ GitHub → Jenkins → SonarQube → Nexus → Docker → DockerHub → Tomcat

✔ Fully automated WAR build
✔ Static code analysis
✔ Artifact upload
✔ Docker image build
✔ Push to DockerHub
✔ Deployment in Tomcat container

This is **production-grade DevOps pipeline**.
Perfect for interviews, resume, GitHub, LinkedIn 🔥

---
# Screenshots
<img width="1920" height="1080" alt="32" src="https://github.com/user-attachments/assets/0f8f1065-26df-4cdd-b473-a538254788c2" />
<img width="1920" height="1080" alt="31" src="https://github.com/user-attachments/assets/fb056118-a32b-4423-85bd-35d38d5a9faf" />
<img width="1920" height="1080" alt="30" src="https://github.com/user-attachments/assets/5672b6c4-648a-4a86-a452-a884785c1987" />
<img width="1920" height="1080" alt="29" src="https://github.com/user-attachments/assets/21321099-3c3d-41e1-88e2-d0562f8344e0" />
<img width="1920" height="1080" alt="28" src="https://github.com/user-attachments/assets/6cf96698-8053-4a87-92c3-ce3e183dc00f" />
<img width="1920" height="1080" alt="27" src="https://github.com/user-attachments/assets/f436f2b3-e150-42b8-8341-613888afe635" />
<img width="1920" height="1080" alt="26" src="https://github.com/user-attachments/assets/f4201586-637f-46f1-bf28-aafe221f03bc" />
<img width="1920" height="1080" alt="25" src="https://github.com/user-attachments/assets/76e10fd4-779a-47e5-81b9-afbf7f5ea964" />
<img width="1920" height="1080" alt="24" src="https://github.com/user-attachments/assets/a7e6541b-ded7-4339-a8ac-e2057dd8d025" />
<img width="1920" height="1080" alt="23" src="https://github.com/user-attachments/assets/5756f5f5-9a5c-4d78-bba4-14b2d44f9ec7" />
<img width="1920" height="1080" alt="22" src="https://github.com/user-attachments/assets/c45d001a-b7a8-4832-aab8-cf59cd29639a" />
<img width="1920" height="1080" alt="21" src="https://github.com/user-attachments/assets/054adc19-ee5a-4b0c-b84c-5244b8b5cc75" />
<img width="1920" height="1080" alt="20" src="https://github.com/user-attachments/assets/7869d15c-7dee-489d-a333-f1b916b536d3" />
<img width="1920" height="1080" alt="19" src="https://github.com/user-attachments/assets/d03362cb-8d00-4985-9889-a6ab112ae7dc" />
<img width="1920" height="1080" alt="18" src="https://github.com/user-attachments/assets/00838247-f5b0-4897-8570-9991f4ea213b" />
<img width="1920" height="1080" alt="17" src="https://github.com/user-attachments/assets/cf1e77a5-b850-41c8-bd2a-52c188f2b49c" />
<img width="1920" height="1080" alt="16" src="https://github.com/user-attachments/assets/0d6fad47-33a3-43b0-8a96-74e536e12cb1" />
<img width="1920" height="1080" alt="15" src="https://github.com/user-attachments/assets/896a912f-172c-4746-bff1-e4e22b4606d0" />
<img width="1920" height="1080" alt="14" src="https://github.com/user-attachments/assets/5da174b9-aca3-4036-8879-1429710e8b9b" />
<img width="1920" height="1080" alt="13" src="https://github.com/user-attachments/assets/7d4e0482-6708-4a94-aaaf-179af3663f75" />
<img width="1920" height="1080" alt="12" src="https://github.com/user-attachments/assets/f581bf22-1e10-4042-84c6-ab234f5ded64" />
<img width="1920" height="1080" alt="11" src="https://github.com/user-attachments/assets/e8b48e3c-55b4-438f-8df1-e01534fce204" />
<img width="1920" height="1080" alt="10" src="https://github.com/user-attachments/assets/5ad6cdbf-f8f8-44b0-85e2-36a271c4e2fd" />
<img width="1920" height="1080" alt="9" src="https://github.com/user-attachments/assets/e7d816c0-2dbc-4c5c-8076-5946449f76d8" />
<img width="1920" height="1080" alt="8" src="https://github.com/user-attachments/assets/c0b31254-213f-4910-b4c9-2f7b652c7f14" />
<img width="1920" height="1080" alt="7" src="https://github.com/user-attachments/assets/413b6e37-a397-428e-a606-664adf968252" />
<img width="1920" height="1080" alt="6" src="https://github.com/user-attachments/assets/248b9c2d-be70-4d21-9136-999b036969db" />
<img width="1920" height="1080" alt="5" src="https://github.com/user-attachments/assets/16487a17-0ede-4b04-9b1c-e415740fcbf9" />
<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/922a0a38-4521-4beb-8c5d-40737e8d91a9" />
<img width="1920" height="1080" alt="3" src="https://github.com/user-attachments/assets/f1aac219-1e23-4db3-9876-90a9f1cff187" />
<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/dfd9d9fb-20d3-4013-afff-34f31821182d" />
<img width="1920" height="1080" alt="01" src="https://github.com/user-attachments/assets/a4222b86-c0d7-4a53-8bef-5b62c9b888da" />


