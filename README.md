![image](https://github.com/chenphopp/Netflix-CI-CD/assets/82653803/3a935349-3ade-47df-b61a-fd8b9f166f16)

# Netflix Clone CI-CD with Monitoring Project 
## From Ajay Kumar Yegireddi
```
Step 1 — Launch an Ubuntu(22.04) T2 Large Instance

Step 2 — Install Jenkins, Docker and Trivy. Create a Sonarqube Container using Docker.

Step 3 — Create a TMDB API Key.

Step 4 — Install Prometheus and Grafana On the new Server.

Step 5 — Install the Prometheus Plugin and Integrate it with the Prometheus server.

Step 6 — Email Integration With Jenkins and Plugin setup.

Step 7 — Install Plugins like JDK, Sonarqube Scanner, Nodejs, and OWASP Dependency Check.

Step 8 — Create a Pipeline Project in Jenkins using a Declarative Pipeline

Step 9 — Install OWASP Dependency Check Plugins

Step 10 — Docker Image Build and Push

Step 11 — Deploy the image using Docker
```
Now, let's get started and dig deeper into each of these steps:-

## STEP1:Launch an Ubuntu(22.04) T2 Large Instance 
And allow port in SG

## Step 2 — Install Jenkins, Docker and Trivy
### 2A — To Install Jenkins
```
vi jenkins.sh #make sure run in Root (or) add at userdata while ec2 launch
```
```
#!/bin/bash
sudo apt update -y
#sudo apt upgrade -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
                  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
                  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
                              /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```
```
COPY
sudo chmod 777 jenkins.sh
./jenkins.sh    # this will installl jenkins
Once Jenkins is installed, you will need to go to your AWS EC2 Security Group and open Inbound Port 8080, since Jenkins works on Port 8080.
```
Now, grab your Public IP Address
```
<EC2 Public IP Address:8080>
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 2B — Install Docker
```
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER   #my case is ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
After the docker installation, we create a sonarqube container (Remember to add 9000 ports in the security group).
```
```
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
```
username admin
password admin
```

### 2C — Install Trivy
```
vi trivy.sh
```
```
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

## Step 3: Create a TMDB API Key
Next, we will create a TMDB API key

![image](https://github.com/chenphopp/Netflix-CI-CD/assets/82653803/50b10b11-1997-41fc-88fa-a283bb3a9ec5)

## Step 4 — Install Prometheus and Grafana On the new Server
```
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```
--system - Will create a system account.
--no-create-home - We don't need a home directory for Prometheus or any other system accounts in our case.
--shell /bin/false - It prevents logging in as a Prometheus user.
Prometheus - Will create a Prometheus user and a group with the same name.

You can use the curl or wget command to download Prometheus.
```
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
```
```
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
```

Usually, you would have a disk mounted to the data directory. For this tutorial, I will simply create a /data directory. Also, you need a folder for Prometheus configuration files.
```
sudo mkdir -p /data /etc/prometheus
```

Now, let's change the directory to Prometheus and move some files.
```
cd prometheus-2.47.1.linux-amd64/
```

First of all, let's move the Prometheus binary and a promtool to the /usr/local/bin/. promtool is used to check configuration files and Prometheus rules.
```
sudo mv prometheus promtool /usr/local/bin/
```

Optionally, we can move console libraries to the Prometheus configuration directory. Console templates allow for the creation of arbitrary consoles using the Go templating language. You don't need to worry about it if you're just getting started.
```
sudo mv consoles/ console_libraries/ /etc/prometheus/
```

Finally, let's move the example of the main Prometheus configuration file.
```
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

To avoid permission issues, you need to set the correct ownership for the /etc/prometheus/ and data directory.
```
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

You can delete the archive and a Prometheus folder when you are done.
```
cd
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
```

Verify that you can execute the Prometheus binary by running the following command:
```
prometheus --version
```

We're going to use some of these options in the service definition.

We're going to use Systemd, which is a system and service manager for Linux operating systems. For that, we need to create a Systemd unit configuration file.
```
sudo vim /etc/systemd/system/prometheus.service
```
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

Let's go over a few of the most important options related to Systemd and Prometheus. Restart - Configures whether the service shall be restarted when the service process exits, is killed, or a timeout is reached.
RestartSec - Configures the time to sleep before restarting a service.
User and Group - Are Linux user and a group to start a Prometheus process.
--config.file=/etc/prometheus/prometheus.yml - Path to the main Prometheus configuration file.
--storage.tsdb.path=/data - Location to store Prometheus data.
--web.listen-address=0.0.0.0:9090 - Configure to listen on all network interfaces. In some situations, you may have a proxy such as nginx to redirect requests to Prometheus. In that case, you would configure Prometheus to listen only on localhost.
--web.enable-lifecycle -- Allows to manage Prometheus, for example, to reload configuration without restarting the service.

To automatically start the Prometheus after reboot, run enable.
```
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Suppose you encounter any issues with Prometheus or are unable to start it. The easiest way to find the problem is to use the journalctl command and search for errors.
```
journalctl -u prometheus -f --no-pager
Now we can try to access it via the browser. I'm going to be using the IP address of the Ubuntu server. You need to append port 9090 to the IP.


COPY

COPY
<public-ip:9090>


If you go to targets, you should see only one - Prometheus target. It scrapes itself every 15 seconds by default.

Install Node Exporter on Ubuntu 22.04
Next, we're going to set up and configure Node Exporter to collect Linux system metrics like CPU load and disk I/O. Node Exporter will expose these as Prometheus-style metrics. Since the installation process is very similar, I'm not going to cover as deep as Prometheus.

First, let's create a system user for Node Exporter by running the following command:


COPY

COPY
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter


You can download Node Exporter from the same page.

Use the wget command to download the binary.


COPY

COPY
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz


Extract the node exporter from the archive.


COPY

COPY
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz


Move binary to the /usr/local/bin.


COPY

COPY
sudo mv \
  node_exporter-1.6.1.linux-amd64/node_exporter \
  /usr/local/bin/


Clean up, and delete node_exporter archive and a folder.


COPY

COPY
rm -rf node_exporter*


Verify that you can run the binary.


COPY

COPY
node_exporter --version


Node Exporter has a lot of plugins that we can enable. If you run Node Exporter help you will get all the options.


COPY

COPY
node_exporter --help
--collector.logind We're going to enable the login controller, just for the demo.

Next, create a similar systemd unit file.


COPY

COPY
sudo vim /etc/systemd/system/node_exporter.service


node_exporter.service


COPY

COPY
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind

[Install]
WantedBy=multi-user.target


Replace Prometheus user and group to node_exporter, and update the ExecStart command.

To automatically start the Node Exporter after reboot, enable the service.


COPY

COPY
sudo systemctl enable node_exporter
Then start the Node Exporter.


COPY

COPY
sudo systemctl start node_exporter


Check the status of Node Exporter with the following command:


COPY

COPY
sudo systemctl status node_exporter


If you have any issues, check logs with journalctl


COPY

COPY
journalctl -u node_exporter -f --no-pager
At this point, we have only a single target in our Prometheus. There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels. In the following tutorials, I'll give you a few examples of deploying Prometheus in a cloud-specific environment. For this tutorial, let's keep it simple and keep adding static targets. Also, I have a lesson on how to deploy and manage Prometheus in the Kubernetes cluster.

To create a static target, you need to add job_name with static_configs.


COPY

COPY
sudo vim /etc/prometheus/prometheus.yml


prometheus.yml


COPY

COPY
  - job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]


By default, Node Exporter will be exposed on port 9100.

Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.

Before, restarting check if the config is valid.


COPY

COPY
promtool check config /etc/prometheus/prometheus.yml


Then, you can use a POST request to reload the config.


COPY

COPY
curl -X POST http://localhost:9090/-/reload


Check the targets section


COPY

COPY
http://<ip>:9090/targets


Install Grafana on Ubuntu 22.04
To visualize metrics we can use Grafana. There are many different data sources that Grafana supports, one of them is Prometheus.

First, let's make sure that all the dependencies are installed.


COPY

COPY
sudo apt-get install -y apt-transport-https software-properties-common


Next, add the GPG key.


COPY

COPY
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -


Add this repository for stable releases.


COPY

COPY
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list


After you add the repository, update and install Garafana.


COPY

COPY
sudo apt-get update

COPY

COPY
sudo apt-get -y install grafana


To automatically start the Grafana after reboot, enable the service.


COPY

COPY
sudo systemctl enable grafana-server
Then start the Grafana.


COPY

COPY
sudo systemctl start grafana-server


To check the status of Grafana, run the following command:


COPY

COPY
sudo systemctl status grafana-server


Go to http://<ip>:3000 and log in to the Grafana using default credentials. The username is admin, and the password is admin as well.


COPY

COPY
username admin
password admin


When you log in for the first time, you get the option to change the password.



To visualize metrics, you need to add a data source first.



Click Add data source and select Prometheus.



For the URL, enter localhost:9090 and click Save and test. You can see Data source is working.


COPY

COPY
<public-ip:9090>


Click on Save and Test.



Let's add Dashboard for a better view



Click on Import Dashboard paste this code 1860 and click on load



Select the Datasource and click on Import



You will see this output



Step 5 — Install the Prometheus Plugin and Integrate it with the Prometheus server
Let's Monitor JENKINS SYSTEM

Need Jenkins up and running machine

Goto Manage Jenkins --> Plugins --> Available Plugins

Search for Prometheus and install it



Once that is done you will Prometheus is set to /Prometheus path in system configurations



Nothing to change click on apply and save

To create a static target, you need to add job_name with static_configs. go to Prometheus server


COPY

COPY
sudo vim /etc/prometheus/prometheus.yml


Paste below code


COPY

COPY
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<jenkins-ip>:8080']


Before, restarting check if the config is valid.


COPY

COPY
promtool check config /etc/prometheus/prometheus.yml
Then, you can use a POST request to reload the config.


COPY

COPY
curl -X POST http://localhost:9090/-/reload


Check the targets section


COPY

COPY
http://<ip>:9090/targets
You will see Jenkins is added to it



Let's add Dashboard for a better view in Grafana

Click On Dashboard --> + symbol --> Import Dashboard

Use Id 9964 and click on load



Select the data source and click on Import



Now you will see the Detailed overview of Jenkins



Step 6 — Email Integration With Jenkins and Plugin Setup
Install Email Extension Plugin in Jenkins



Go to your Gmail and click on your profile

Then click on Manage Your Google Account --> click on the security tab on the left side panel you will get this page(provide mail password).



2-step verification should be enabled.

Search for the app in the search bar you will get app passwords like the below image





Click on other and provide your name and click on Generate and copy the password



In the new update, you will get a password like this



Once the plugin is installed in Jenkins, click on manage Jenkins --> configure system there under the E-mail Notification section configure the details as shown in the below image





Click on Apply and save.

Click on Manage Jenkins--> credentials and add your mail username and generated password



This is to just verify the mail configuration

Now under the Extended E-mail Notification section configure the details as shown in the below images







Click on Apply and save.


COPY

COPY
post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'postbox.aj99@gmail.com',  #change Your mail
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
Next, we will log in to Jenkins and start to configure our Pipeline in Jenkins

Step 7 — Install Plugins like JDK, Sonarqube Scanner, NodeJs, OWASP Dependency Check
7A — Install Plugin
Goto Manage Jenkins →Plugins → Available Plugins →

Install below plugins

1 → Eclipse Temurin Installer (Install without restart)

2 → SonarQube Scanner (Install without restart)

3 → NodeJs Plugin (Install Without restart)





7B — Configure Java and Nodejs in Global Tool Configuration
Goto Manage Jenkins → Tools → Install JDK(17) and NodeJs(16)→ Click on Apply and Save





7C — Create a Job
create a job as Netflix Name, select pipeline and click on ok.

Step 8 — Configure Sonar Server in Manage Jenkins
Grab the Public IP Address of your EC2 Instance, Sonarqube works on Port 9000, so <Public IP>:9000. Goto your Sonarqube Server. Click on Administration → Security → Users → Click on Tokens and Update Token → Give it a name → and click on Generate Token



click on update Token



Create a token with a name and generate



copy Token

Goto Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text. It should look like this



You will this page once you click on create



Now, go to Dashboard → Manage Jenkins → System and Add like the below image.



Click on Apply and Save

The Configure System option is used in Jenkins to configure different server

Global Tool Configuration is used to configure different tools that we install using Plugins

We will install a sonar scanner in the tools.



In the Sonarqube Dashboard add a quality gate also

Administration--> Configuration-->Webhooks



Click on Create



Add details


COPY

COPY
#in url section of quality gate
<http://jenkins-public-ip:8080>/sonarqube-webhook/


Let's go to our Pipeline and add the script in our Pipeline Script.


COPY

COPY
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Netflix-clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'postbox.aj99@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
Click on Build now, you will see the stage view like this



To see the report, you can go to Sonarqube Server and go to Projects.



You can see the report has been generated and the status shows as passed. You can see that there are 3.2k lines it scanned. To see a detailed report, you can go to issues.

Step 9 — Install OWASP Dependency Check Plugins
GotoDashboard → Manage Jenkins → Plugins → OWASP Dependency-Check. Click on it and install it without restart.



First, we configured the Plugin and next, we had to configure the Tool

Goto Dashboard → Manage Jenkins → Tools →



Click on Apply and Save here.

Now go configure → Pipeline and add this stage to your pipeline and build.


COPY

COPY
stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
The stage view would look like this,



You will see that in status, a graph will also be generated and Vulnerabilities.



Step 10 — Docker Image Build and Push
We need to install the Docker tool in our system, Goto Dashboard → Manage Plugins → Available plugins → Search for Docker and install these plugins

Docker

Docker Commons

Docker Pipeline

Docker API

docker-build-step

and click on install without restart



Now, goto Dashboard → Manage Jenkins → Tools →



Add DockerHub Username and Password under Global Credentials



Add this stage to Pipeline Script

COPY


COPY

COPY
stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=Aj7ay86fe14eca3e76869b92 -t netflix ."
                       sh "docker tag netflix sevenajay/netflix:latest "
                       sh "docker push sevenajay/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image sevenajay/netflix:latest > trivyimage.txt" 
            }
        }
You will see the output below, with a dependency trend.



When you log in to Dockerhub, you will see a new image is created



Now Run the container to see if the game coming up or not by adding the below stage


COPY

COPY
stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 sevenajay/netflix:latest'
            }
        }
stage view



<Jenkins-public-ip:8081>

You will get this output



Step 11 — Kuberenetes Setup
Connect your machines to Putty or Mobaxtreme

Take-Two Ubuntu 20.04 instances one for k8s master and the other one for worker.

Install Kubectl on Jenkins machine also.

Kubectl is to be installed on Jenkins also

COPY

COPY
sudo apt update
sudo apt install curl
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
Part 1 ----------Master Node------------

COPY

COPY
sudo hostnamectl set-hostname K8s-Master
----------Worker Node------------

COPY

COPY
sudo hostnamectl set-hostname K8s-Worker
Part 2 ------------Both Master & Node ------------

COPY

COPY
sudo apt-get update 

sudo apt-get install -y docker.io
sudo usermod –aG docker Ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock

sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo tee /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update

sudo apt-get install -y kubelet kubeadm kubectl

sudo snap install kube-apiserver
Part 3 --------------- Master ---------------

COPY

COPY
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# in case your in root exit from it and run below commands
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
----------Worker Node------------

COPY

COPY
sudo kubeadm join <master-node-ip>:<master-node-port> --token <token> --discovery-token-ca-cert-hash <hash>
Copy the config file to Jenkins master or the local file manager and save it



copy it and save it in documents or another folder save it as secret-file.txt

Note: create a secret-file.txt in your file explorer save the config in it and use this at the kubernetes credential section.

Install Kubernetes Plugin, Once it's installed successfully



goto manage Jenkins --> manage credentials --> Click on Jenkins global --> add credentials



Install Node_exporter on both master and worker
Let's add Node_exporter on Master and Worker to monitor the metrics

First, let's create a system user for Node Exporter by running the following command:


COPY

COPY
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter


You can download Node Exporter from the same page.

Use the wget command to download the binary.


COPY

COPY
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz


Extract the node exporter from the archive.


COPY

COPY
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz


Move binary to the /usr/local/bin.


COPY

COPY
sudo mv \
  node_exporter-1.6.1.linux-amd64/node_exporter \
  /usr/local/bin/


Clean up, and delete node_exporter archive and a folder.


COPY

COPY
rm -rf node_exporter*


Verify that you can run the binary.


COPY

COPY
node_exporter --version


Node Exporter has a lot of plugins that we can enable. If you run Node Exporter help you will get all the options.


COPY

COPY
node_exporter --help
--collector.logind We're going to enable the login controller, just for the demo.

Next, create a similar systemd unit file.


COPY

COPY
sudo vim /etc/systemd/system/node_exporter.service


node_exporter.service


COPY

COPY
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind

[Install]
WantedBy=multi-user.target


Replace Prometheus user and group to node_exporter, and update the ExecStart command.

To automatically start the Node Exporter after reboot, enable the service.


COPY

COPY
sudo systemctl enable node_exporter
Then start the Node Exporter.


COPY

COPY
sudo systemctl start node_exporter


Check the status of Node Exporter with the following command:


COPY

COPY
sudo systemctl status node_exporter


If you have any issues, check logs with journalctl


COPY

COPY
journalctl -u node_exporter -f --no-pager
At this point, we have only a single target in our Prometheus. There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels. In the following tutorials, I'll give you a few examples of deploying Prometheus in a cloud-specific environment. For this tutorial, let's keep it simple and keep adding static targets. Also, I have a lesson on how to deploy and manage Prometheus in the Kubernetes cluster.

To create a static target, you need to add job_name with static_configs. Go to Prometheus server


COPY

COPY
sudo vim /etc/prometheus/prometheus.yml


prometheus.yml


COPY

COPY
  - job_name: node_export_masterk8s
    static_configs:
      - targets: ["<master-ip>:9100"]

  - job_name: node_export_workerk8s
    static_configs:
      - targets: ["<worker-ip>:9100"]
By default, Node Exporter will be exposed on port 9100.



Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.

Before, restarting check if the config is valid.


COPY

COPY
promtool check config /etc/prometheus/prometheus.yml


Then, you can use a POST request to reload the config.


COPY

COPY
curl -X POST http://localhost:9090/-/reload


Check the targets section


COPY

COPY
http://<ip>:9090/targets


final step to deploy on the Kubernetes cluster


COPY

COPY
stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }
stage view



In the Kubernetes cluster(master) give this command


COPY

COPY
kubectl get all 
kubectl get svc #use anyone


STEP 12:Access from a Web browser with
<public-ip-of-slave:service port>

output:



Monitoring







Mail



Step 13: Terminate instances.
Complete Pipeline

COPY

COPY
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Netflix-clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=AJ7AYe14eca3e76864yah319b92 -t netflix ."
                       sh "docker tag netflix sevenajay/netflix:latest "
                       sh "docker push sevenajay/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image sevenajay/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 sevenajay/netflix:latest'
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }

    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'postbox.aj99@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}

47

4



Subscribe to my newsletter
Read articles from Ajay Kumar Yegireddi directly inside your inbox. Subscribe to the newsletter, and don't miss out.

Enter your email address
SUBSCRIBE
Did you find this article valuable?
Support Ajay Kumar Yegireddi by becoming a sponsor. Any amount is appreciated!


Sponsor
Learn more about Hashnode Sponsors
Devops
netflix
Jenkins
AWS
Docker
Written by

Ajay Kumar Yegireddi
Ajay Kumar Yegireddi
Aspiring Technologist, YouTuber, DevSecOps Enthusiast

🎥 Greetings, tech enthusiasts! I'm Ajay Kumar Yegireddi , an aspiring technologist with an insatiable appetite for learning and teaching. While I may not have a long list of professional experience, I'm on a journey to share my passion for technology and DevSecOps with the world through my YouTube channel and blog.

🚀 I firmly believe that knowledge should be accessible to everyone, regardless of their background or experience level. My mission is to demystify complex tech concepts, break them down into bite-sized pieces, and make them approachable for newcomers to the field.

🛡️ In today's digitally driven world, security is paramount, and understanding how to secure software is a skill that holds immense value. Join me as we explore the exciting realm of DevSecOps, dive into containerization with Docker and Kubernetes, and uncover the intricacies of CI/CD pipelines using Jenkins.

👨‍💻 My journey is a shared one, and I invite you to learn alongside me. Together, we'll acquire the skills and insights needed to develop and secure software, even if we're starting from square one. So, let's embark on this educational adventure together, and who knows where our tech journey will take us!


Follow
 
MORE ARTICLES

Ajay Kumar Yegireddi's photo
Ajay Kumar Yegireddi
YouTube App Deployment with GitLab CI/CD
YouTube App Deployment with GitLab CI/CD
In today's software development landscape, speed and reliability are paramount. Continuous Integrati…


Ajay Kumar Yegireddi's photo
Ajay Kumar Yegireddi
GitLab CI/CD for Terraform | Plan, Apply, Destroy: Managing Infrastructure
GitLab CI/CD for Terraform | Plan, Apply, Destroy: Managing Infrastructure
YouTube video: https://youtu.be/pqW0vuEj22U Think of GitLab CI/CD as your personal robot helper for …


Ajay Kumar Yegireddi's photo
Ajay Kumar Yegireddi
Mastering Docker Security: 20 Tips
Mastering Docker Security: 20 Tips
1/20 🔒 Update Regularly: Always keep your Docker daemon and base images up-to-date to patch known v…

©2023 Ajay Kumar Yegireddi

Archive
·
Privacy policy
·
Terms
Publish with Hashnode
Powered by Hashnode - Home for tech writers and readers



Step 12 — Kubernetes master and slave setup on Ubuntu (20.04)

Step 13 — Access the Netflix app on the Browser.

Step 14 — Terminate the AWS EC2 Instances.
