
#  Jenkins CI/CD Pipeline Using Docker, SonarQube, Nexus, Maven & Tomcat**

A fully automated CI/CD pipeline using **Docker containers**, **Jenkins**, **SonarQube**, **Nexus**, **Maven**, and **Tomcat**, deployed on **Ubuntu EC2**.

This pipeline automates:

1. **Clone source code**
2. **Run SonarQube Code Quality Analysis**
3. **Build using Maven**
4. **Upload WAR artifact to Nexus Repository**
5. **Build custom Tomcat Docker image**
6. **Push image to Docker Hub**
7. **Deploy container to EC2**

---

# **📌 1. Project Architecture**

```
                   +-------------------+
                   |     GitHub Repo   |
                   +---------+---------+
                             |
                             v
                   +-------------------+
                   |     Jenkins CI    |
                   | Pipeline (SCM)    |
                   +---------+---------+
                             |
     +----------+------------+--------------+------------------+
     |          |                           |                  |
     v          v                           v                  v
+---------+ +--------+           +----------------+     +--------------+
| SonarQ | | Maven  |           | Nexus Artifact |     | Docker build |
|  Qube  | | Build  |           | Repository     |     | & Docker Hub |
+---------+ +--------+           +----------------+     +--------------+
                                                             |
                                                             v
                                                     +---------------+
                                                     |   EC2 Deploy  |
                                                     +---------------+
```

---

# **📌 2. Create EC2 Instance**

### **EC2 Settings**

* OS: **Ubuntu**
* Instance Type: **t2.large**
* Key Pair: **Create new key**
* Network: Allow:

  * **SSH (22)**
  * **All traffic** (for lab/demo use)
* Storage: **16GB**

<img width="602" height="273" alt="image" src="https://github.com/user-attachments/assets/4b9f012c-b7fb-4934-8790-02f1c09c2fd5" />

<img width="602" height="277" alt="image" src="https://github.com/user-attachments/assets/172df02b-6209-4c51-b37a-2ff387756b7b" />

<img width="602" height="167" alt="image" src="https://github.com/user-attachments/assets/d732f6ad-7eb4-4ad7-a56a-47ea8e4ad66c" />

<img width="602" height="273" alt="image" src="https://github.com/user-attachments/assets/7a951184-2b7f-4985-94d3-62596f3e92a0" />

<img width="602" height="160" alt="image" src="https://github.com/user-attachments/assets/70dd1e3e-e258-476b-8dbe-6182bfe6f538" />

<img width="602" height="297" alt="image" src="https://github.com/user-attachments/assets/73874680-ce00-45c9-8620-8e6bcff67a13" />

---

# **📌 3. SSH Into the EC2 Server**

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```
<img width="602" height="204" alt="image" src="https://github.com/user-attachments/assets/c63b7347-512b-466e-9336-8ef2e5c7273a" />

---

# **📌 4. Update Server & Install Dependencies**

```bash
sudo apt update -y
sudo apt upgrade -y
```

Clone your project:

```bash
git clone https://github.com/SaiKumar7596/puzzle-game-java
cd puzzle-game-java
```
<img width="602" height="86" alt="image" src="https://github.com/user-attachments/assets/256966dd-c202-4e0c-b034-c1226958a4e4" />

---

# **📌 5. Install Docker**

```bash
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
```
<img width="602" height="38" alt="image" src="https://github.com/user-attachments/assets/6a23b390-7d37-47dd-9d21-59a20b8a2af8" />

<img width="602" height="83" alt="image" src="https://github.com/user-attachments/assets/1bbc5349-e4ae-4f08-9701-01ec9707c02c" />

---

# **📌 6. Deploy SonarQube**

### **Pull SonarQube Image**

```bash
docker pull sonarqube
```
<img width="602" height="175" alt="image" src="https://github.com/user-attachments/assets/27c17e01-b36e-4a05-abde-0a430a8d58ca" />

### **Run SonarQube Container**

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube
```
<img width="602" height="81" alt="image" src="https://github.com/user-attachments/assets/2a3584a8-b012-44ef-aa84-95aabb7e6aab" />

### **Access SonarQube**

```
http://<EC2_PUBLIC_IP>:9000
```

<img width="602" height="295" alt="image" src="https://github.com/user-attachments/assets/fcab32e4-9fb6-4563-85a7-b66582dd17a2" />

Default login:

* **username:** admin
* **password:** admin

Change password and create a **Sonar Token** for Jenkins.

<img width="602" height="254" alt="image" src="https://github.com/user-attachments/assets/0080573a-e34c-4329-b26b-e60fb997756c" />

---

# **📌 7. Install Java & Maven on EC2**

```bash
sudo apt install openjdk-17-jdk -y
sudo apt install maven -y
```

Verify:

```bash
java -version
mvn -version
```

<img width="602" height="43" alt="image" src="https://github.com/user-attachments/assets/032d4f2a-3285-41eb-a1af-73b216639667" />

<img width="602" height="62" alt="image" src="https://github.com/user-attachments/assets/9a16ef1b-dff7-40b4-a5d1-26543767f209" />

---

# **📌 8. Run SonarQube Scan**

```bash
mvn clean install sonar:sonar \
 -Dsonar.host.url=http://54.81.139.84:9000 \
 -Dsonar.login=squ_eb6c08ff6f7d390720b6b9dcf906d3d7eedac1b0
```
<img width="602" height="101" alt="image" src="https://github.com/user-attachments/assets/675bf0c7-7584-4be6-977d-4dbf61e49160" />

<img width="602" height="70" alt="image" src="https://github.com/user-attachments/assets/0897524d-1a17-4fca-9dd8-1bc108159710" />


<img width="602" height="286" alt="image" src="https://github.com/user-attachments/assets/9bb8da20-cdc9-4bfc-8635-6d1c93d40a63" />

---

# **📌 9. Deploy Nexus Repository**

### **Pull Nexus Image**

```bash
docker pull sonatype/nexus3
```

<img width="602" height="151" alt="image" src="https://github.com/user-attachments/assets/47cfaa95-1185-45ac-8901-5d10c22edcbe" />

### **Run Nexus Container**

```bash
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
```

<img width="602" height="64" alt="image" src="https://github.com/user-attachments/assets/fef45b3d-e628-440e-8c77-1ab1df8e7965" />

Open in browser:

```
http://<EC2_PUBLIC_IP>:8081
```

<img width="602" height="243" alt="image" src="https://github.com/user-attachments/assets/ae810662-9afc-45f7-8c3c-65f5c203f47c" />

### **Get Admin Password**

```bash
docker exec -it nexus cat /nexus-data/admin.password
```
<img width="602" height="114" alt="image" src="https://github.com/user-attachments/assets/3606aa95-3562-429b-97a1-e56bf31099a7" />

<img width="602" height="277" alt="image" src="https://github.com/user-attachments/assets/201a8149-2188-47d8-bbdd-1d649ba63253" />

<img width="602" height="201" alt="image" src="https://github.com/user-attachments/assets/bcc2a737-2f2e-4112-afa2-0eb69d84234b" />

---

# **📌 10. Configure Maven for Nexus**

### **Update pom.xml**

```xml
<distributionManagement>
  <repository>
    <id>maven-releases</id>
    <name>Maven Releases</name>
    <url>http://<NEXUS_IP>:8081/repository/maven-releases/</url>
  </repository>
</distributionManagement>
```

<img width="602" height="29" alt="image" src="https://github.com/user-attachments/assets/f7a0e22f-e9be-46a7-beea-a7e11372ea52" />

<img width="602" height="85" alt="image" src="https://github.com/user-attachments/assets/e5b71e14-19cd-4dd1-ae8a-6757d0e1cf0a" />


### **Edit Maven settings.xml**

Location:

```
vi /etc/maven/settings.xml 
```

Add:

```xml
<servers>
  <server>
    <id>maven-releases</id>
    <username>admin</username>
    <password>your_nexus_password</password>
  </server>
</servers>
```

<img width="602" height="30" alt="image" src="https://github.com/user-attachments/assets/678919d3-8cfc-409b-8562-c4c8212414bb" />

<img width="602" height="109" alt="image" src="https://github.com/user-attachments/assets/6006df67-23d2-4ed9-9589-c48f450d8227" />

---

# **📌 11. Build & Upload to Nexus**

```bash
mvn clean deploy
```


<img width="602" height="120" alt="image" src="https://github.com/user-attachments/assets/c0410bef-044e-422b-8f4a-d3167b546d65" />


<img width="602" height="156" alt="image" src="https://github.com/user-attachments/assets/d07e61a6-c2cc-4e4a-8ee3-64aa24d8c198" />

Verify artifact in Nexus:

```
maven-releases/com/example/puzzle-game-webapp/1.0/
```

<img width="602" height="248" alt="image" src="https://github.com/user-attachments/assets/2d5dd2e0-e71a-44fc-82d2-155e08e6e01e" />

---

# **📌 12. Create Custom Tomcat Docker Image**

### **Dockerfile**

```

```

### Build Image

```bash
docker build -t pz-tomcat:1.0 .
```
# Step 1: Use official Tomcat image with JDK 17
FROM tomcat:10.1.10-jdk17

# Step 2: Switch to root and install curl
USER root
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

# Step 3: Remove default Tomcat webapps
RUN rm -rf /usr/local/tomcat/webapps/*

# Step 4: Set environment variables for Nexus
ENV NEXUS_USER=admin
ENV NEXUS_PASS=admin123

# These are correct values based on your Nexus URL
ENV WAR_URL=http://54.81.139.84:8081/repository/maven-releases/com/example/puzzle-game-webapp/1.0/puzzle-game-webapp-1.0.war
ENV WAR_NAME=puzzle-game-webapp.war

# Step 5: Set working directory for Tomcat
WORKDIR /usr/local/tomcat/webapps

# Step 6: Download WAR (with fail if error)
RUN echo "Downloading WAR from Nexus..." && \
    curl -u "$NEXUS_USER:$NEXUS_PASS" -L -f -o "$WAR_NAME" "$WAR_URL" && \
    echo "WAR downloaded successfully:" && ls -lh $WAR_NAME

# Step 7: Expose Tomcat port
EXPOSE 8080

# Step 8: Start Tomcat
CMD ["catalina.sh", "run"]

---


<img width="602" height="25" alt="image" src="https://github.com/user-attachments/assets/f26e76d3-46cd-4b2c-8d95-74b27091d6fb" />


<img width="602" height="77" alt="image" src="https://github.com/user-attachments/assets/4fed7d87-c75a-47ed-b01e-398b44bba040" />

# Now build the tomcat custom image:

```
docker build -t pz-tomcat:1.0 .
```

<img width="602" height="162" alt="image" src="https://github.com/user-attachments/assets/40e5aa49-16ce-4d4d-adab-4767cfb1a877" />


<img width="602" height="91" alt="image" src="https://github.com/user-attachments/assets/1e2978d0-54fb-4135-86b4-7ffe8e824e01" />

# Go to docker hub and create repo:


<img width="602" height="246" alt="image" src="https://github.com/user-attachments/assets/4dd7bbda-6d34-4dd1-8023-318819f1aa0e" />

# **📌 13. Push Image to Docker Hub**

```bash
docker login
docker tag pz-tomcat:1.0 saikumar7596/pz-tomcat:1.0
docker push saikumar7596/pz-tomcat:1.0
```


<img width="602" height="151" alt="image" src="https://github.com/user-attachments/assets/7d7609b7-be46-44bd-81a9-1fe435095e0c" />


<img width="602" height="289" alt="image" src="https://github.com/user-attachments/assets/add391f0-b27b-4d87-8cba-efc233f34c83" />


<img width="602" height="169" alt="image" src="https://github.com/user-attachments/assets/00495ef1-d5e2-4caa-9141-7bfcdc69456c" />


<img width="602" height="241" alt="image" src="https://github.com/user-attachments/assets/4f0ca421-e824-4592-8c26-91c5a5668cdc" />


# verify it

<img width="602" height="184" alt="image" src="https://github.com/user-attachments/assets/0b802664-dc83-4cb4-837d-031605407cf7" />


---

# **📌 14. Test Tomcat Container**

```bash
docker run -d --name tomcat -p 8080:8080 pz-tomcat:1.0
```


<img width="602" height="47" alt="image" src="https://github.com/user-attachments/assets/fec3acc7-d7ea-4bed-824e-433c694a67b7" />

Check in browser:

```
http://<EC2_PUBLIC_IP>:8080/puzzle-game-webapp/
```


<img width="602" height="152" alt="image" src="https://github.com/user-attachments/assets/e7866577-70c7-4190-b7f3-0abc157d8b3f" />

---

# **📌 15. Install Jenkins**

### Pull Jenkins Image

```bash
docker pull jenkins/jenkins:lts
```

<img width="602" height="267" alt="image" src="https://github.com/user-attachments/assets/d8d19882-8b66-4622-93ee-64696aec8b3d" />

### Create storage folder

```bash
sudo mkdir /jenkins-data
sudo chmod 777 /jenkins-data
```

### Run Jenkins

```bash
docker run -d --name jenkins -p 8090:8080 -v /jenkins-data:/var/jenkins_home jenkins/jenkins:lts
```

<img width="602" height="113" alt="image" src="https://github.com/user-attachments/assets/d049c356-01eb-486a-82d3-4cb4343f45c2" />


<img width="602" height="53" alt="image" src="https://github.com/user-attachments/assets/064a2003-736d-4709-ac42-3cafe5581b0a" />

---

# **📌 16. Unlock Jenkins**

```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

<img width="602" height="296" alt="image" src="https://github.com/user-attachments/assets/a78a16aa-98f7-46a8-a999-4f54809a4e15" />


<img width="602" height="38" alt="image" src="https://github.com/user-attachments/assets/11e01aa0-28eb-46cb-87f2-1fb9f380fef3" />


<img width="602" height="301" alt="image" src="https://github.com/user-attachments/assets/7fedb407-9aa0-4fbd-95ac-3512c57db88e" />


<img width="602" height="200" alt="image" src="https://github.com/user-attachments/assets/4bc27c62-089e-4c36-b6a6-8928d731e14b" />


Install **Suggested Plugins**.

---

# **📌 17. Install Required Jenkins Plugins**

✔ Git
✔ Pipeline
✔ Docker Pipeline
✔ SSH Agent
✔ SonarQube Scanner
✔ Nexus Artifact Uploader
✔ Credentials Binding



<img width="602" height="294" alt="image" src="https://github.com/user-attachments/assets/e94a666f-9f5f-45e8-a522-2167190434ee" />


<img width="602" height="333" alt="image" src="https://github.com/user-attachments/assets/ce5645a4-c49c-42c0-b900-356605b099c5" />


<img width="602" height="309" alt="image" src="https://github.com/user-attachments/assets/2bfd3409-327d-4b16-af37-989e72cf9a61" />

---

# **📌 18. Install Java, Maven & Docker *inside Jenkins container***

### Enter as root

```bash
docker exec -u 0 -it jenkins bash
```

Install Java:

```bash
apt update
apt install -y openjdk-17-jdk
```

Install Maven:

```bash
apt install -y maven
```

<img width="602" height="71" alt="image" src="https://github.com/user-attachments/assets/8fade785-8ac3-4673-ad98-a9224bd16563" />

Install Docker:

```bash
apt install -y docker.io
```

Add Jenkins to docker group:

```bash
groupadd docker || true
usermod -aG docker jenkins
```

<img width="602" height="59" alt="image" src="https://github.com/user-attachments/assets/0ab42550-5150-4a97-bcfe-0fe3fed1b5d8" />

Exit & restart:

```bash
exit
docker restart jenkins
```

<img width="602" height="167" alt="image" src="https://github.com/user-attachments/assets/3f786d90-4c80-47c9-9ff1-cec0a13bb85e" />

---

# **📌 19. Jenkins Tool Configuration**

| Tool          | Name          | Path                               |
| ------------- | ------------- | ---------------------------------- |
| JDK           | JDK17         | /usr/lib/jvm/java-17-openjdk-amd64 |
| Maven         | Maven         | /usr/bin/mvn                       |
| Sonar Scanner | sonar-scanner | /opt/sonar-scanner                 |
| Docker        | docker        | /usr/bin                           |



<img width="602" height="121" alt="image" src="https://github.com/user-attachments/assets/96828db1-2c20-4530-8fd0-5a75129fc454" />


<img width="602" height="185" alt="image" src="https://github.com/user-attachments/assets/99dd3649-2baf-4ef4-a4d5-95d215728a0b" />


<img width="602" height="117" alt="image" src="https://github.com/user-attachments/assets/7042c5b2-498c-49aa-9726-57489b598dfd" />


<img width="602" height="161" alt="image" src="https://github.com/user-attachments/assets/e04b5550-7f80-4887-bb88-6c3a8f0314fa" />


<img width="602" height="131" alt="image" src="https://github.com/user-attachments/assets/bfee68df-26fa-4aa3-91cb-5e9d4459cebb" />

---

# **📌 20. Configure Jenkins Credentials**

### **Nexus**

```
ID: nexus
Username: admin
Password: admin123
```

<img width="602" height="296" alt="image" src="https://github.com/user-attachments/assets/df6a5801-e2f3-4815-bacd-e5b65f34f1b5" />


<img width="602" height="168" alt="image" src="https://github.com/user-attachments/assets/7a3b22e7-d338-4e83-833b-8e92c2e97c63" />


### **DockerHub**

```
ID: dockerhub-creds
```


<img width="602" height="261" alt="image" src="https://github.com/user-attachments/assets/6c275a81-b0e0-48f6-975d-e1e3f14000e6" />


<img width="602" height="153" alt="image" src="https://github.com/user-attachments/assets/50e11429-77e6-442b-b7f2-c6affa5c0fc3" />


### **SSH Private Key for EC2 Deployment**

```
ID: docker-server
Username: ubuntu
Private Key: paste .pem file
```

---

# **📌 21. Configure SonarQube in Jenkins**

Path:
**Manage Jenkins → Configure System → SonarQube**

```
Name: sonar-server
URL: http://<SONAR_IP>:9000
Token: <your-token>
```

<img width="602" height="293" alt="image" src="https://github.com/user-attachments/assets/8205f2e8-5ac1-49cc-a2b2-d705bf1c2887" />


<img width="602" height="252" alt="image" src="https://github.com/user-attachments/assets/919b8850-d57e-4012-b26c-6b4b3eb0f876" />

---

# Create Pipeline Job in Jenkins
Go to New Item → Pipeline
Name:
puzzle-game-ci-cd
Scroll to Pipeline → Definition = Pipeline Script from SCM
SCM: Git
Repo URL:
https://github.com/SaiKumar7596/puzzle-game-java
Script Path:
Jenkinsfile
Save.

<img width="602" height="292" alt="image" src="https://github.com/user-attachments/assets/16dac26a-474d-43bf-8ca6-37e798011679" />


<img width="602" height="180" alt="image" src="https://github.com/user-attachments/assets/e9111c15-7022-4661-b05d-d2f63a39d603" />

# **📌 22. Configure GitHub Webhook**

GitHub → Repo → Settings → Webhooks → Add Webhook

```
http://<JENKINS_IP>:8080/github-webhook/
Content type: application/json
Event: Push
```

<img width="602" height="262" alt="image" src="https://github.com/user-attachments/assets/b1e6f84a-12a0-4b91-a00b-bd16001bf110" />


<img width="602" height="202" alt="image" src="https://github.com/user-attachments/assets/817f5b38-2604-4a8f-b4f5-4cadb7f208ac" />

---

# **📌 23. Configure SonarQube Webhook**

SonarQube UI → Administration → Configuration → Webhooks

```
http://<JENKINS_IP>:8080/sonarqube-webhook/
```

<img width="602" height="242" alt="image" src="https://github.com/user-attachments/assets/2a548b24-eef4-4067-9e21-461f4d40cf6b" />

---
✅ Before Running the Jenkins Pipeline — Create Dockerfile & Jenkinsfile in Your GitHub Repo

Before triggering your CI/CD pipeline, you must prepare your application repository with two important files:

1. Dockerfile (For Tomcat Deployment Image)

This Dockerfile builds your custom Tomcat image that will deploy the .war file downloaded from Nexus.

2. Jenkinsfile (Pipeline Script)

Your Jenkinsfile defines the entire CI/CD workflow:

Clone the source code

Run SonarQube scan

Build using Maven

Deploy .war to Nexus

Build custom Docker image

Push to Docker Hub

Deploy container



# **📌 24. Jenkinsfile (Final Production Version)**

```
```

---

# **📌 25. Run Jenkins Pipeline**

After pushing new code → the pipeline runs automatically.

---

<img width="602" height="303" alt="image" src="https://github.com/user-attachments/assets/37dab965-f9b7-4e74-9f48-05c76070c392" />


<img width="602" height="303" alt="image" src="https://github.com/user-attachments/assets/e95270b4-9884-4f75-8903-b245061f4ca0" />


<img width="602" height="274" alt="image" src="https://github.com/user-attachments/assets/e9882303-9a27-4c58-b24c-ef3f4f4bf44c" />


<img width="602" height="241" alt="image" src="https://github.com/user-attachments/assets/f41ad30f-9b19-4e1c-917f-5720b0923402" />


# **📌 26. Verification**

### Check containers:

```bash
docker ps
```

<img width="602" height="53" alt="image" src="https://github.com/user-attachments/assets/419c93d5-0422-49dd-9c09-f98060c5736c" />

### Logs:

```bash
docker logs tomcat
```

You should see:

```
Deployment of web application archive [...] has finished
```

### Application URL:

```
http://<EC2_IP>:8080/
```

<img width="602" height="130" alt="image" src="https://github.com/user-attachments/assets/baaaf91c-462d-4862-8ad3-af69b53fded6" />


---

# **📌 27. Troubleshooting**

### ❌ SonarQube not reachable

→ Restart container

```bash
docker restart sonarqube
```

### ❌ Nexus upload failing

→ Check credentials in settings.xml
→ Check repository URL

### ❌ Jenkins cannot run Docker

→ Add user to docker group
→ Restart Jenkins container

### ❌ Webhook not triggering

→ Ensure URL ends with `/`

### ❌ Tomcat WAR not deployed

→ Ensure Dockerfile WAR_URL is correct

---

# **📌 28. Conclusion**

Your CI/CD pipeline:

✔ Automatically analyzes code
✔ Uploads artifacts
✔ Builds Docker images
✔ Deploys to EC2
✔ Fully automated using Jenkins

---
🎯 Your CI/CD Now Runs Automatically
Whenever you push to main, the full pipeline runs from GitHub Actions.

