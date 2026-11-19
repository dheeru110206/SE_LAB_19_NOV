# DevOps Lab Practical — Complete Command & Workflow Handbook

A practical revision guide for **Docker, Docker Compose, Dockerfiles, Kubernetes/Minikube, Nagios, AWS EC2 and Jenkins**. Commands are organised in execution order, with complete sample files, verification and cleanup.

> Replace placeholders such as `YOUR_GITHUB_URL`, `YOUR_PUBLIC_IP`, `YOUR_DOCKERHUB_USERNAME` and credentials with your actual values. PowerShell commands are for Windows; Bash commands are for Ubuntu/containers. Do not mix terminals.

## Contents
1. Exam workflow
2. Docker commands
3. Dockerfiles
4. Docker Compose
5. Kubernetes/Minikube
6. Nagios
7. AWS EC2
8. Jenkins
9. Troubleshooting and viva
10. Final checklist

---

# 1. Exam workflow

For every task:

1. Identify required image, application, ports, credentials, replica count and expected output.
2. Create files in the correct directory.
3. Validate configuration (`docker compose config`, Nagios pre-flight check, etc.).
4. Build/start/deploy.
5. Verify using status commands and a browser.
6. Capture screenshots if required.
7. Clean up only if the question asks.

**Port mapping:** `HOST_PORT:CONTAINER_PORT`. For `-p 8080:80`, browse to `http://localhost:8080`; the application listens on port 80 inside the container.

---

# 2. Docker commands

## Verify installation (PowerShell)

```powershell
docker --version
docker compose version
docker info
```

Start Docker Desktop and wait for it to be running if the daemon is unavailable.

## Images

```powershell
docker pull nginx:latest
docker images
docker image ls
docker rmi IMAGE_NAME
```

## Containers

```powershell
docker run -d --name web1 -p 8080:80 nginx:latest
docker ps
docker ps -a
docker logs web1
docker logs -f web1
docker exec -it web1 bash
docker stop web1
docker start web1
docker restart web1
docker rm web1
```

`Ctrl+C` exits `docker logs -f`; it does not stop the container.

Remove a running container:

```powershell
docker stop web1
docker rm web1
```

## Inspect and troubleshoot

```powershell
docker inspect CONTAINER_NAME
docker port CONTAINER_NAME
docker stats
docker network ls
docker volume ls
```

Find a process using a Windows port:

```powershell
netstat -ano | findstr :8080
```

## Docker Hub

```powershell
docker login
docker tag myapp:latest YOUR_DOCKERHUB_USERNAME/myapp:latest
docker push YOUR_DOCKERHUB_USERNAME/myapp:latest
docker pull YOUR_DOCKERHUB_USERNAME/myapp:latest
```

---

# 3. Dockerfile practicals

## General workflow

```powershell
mkdir docker-lab
cd docker-lab
notepad Dockerfile
docker build -t myapp:latest .
docker images
docker run -d --name myapp-container -p 8080:8080 myapp:latest
docker ps
docker logs myapp-container
```

Create a file named exactly `Dockerfile` (no `.txt` extension). The final dot in `docker build ... .` means the current directory is the build context.

## Flask + Docker

Create `app.py`:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from Flask + Docker!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Create `Dockerfile`:

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY app.py .
RUN pip install --no-cache-dir flask
EXPOSE 5000
CMD ["python", "app.py"]
```

Build, run and verify:

```powershell
docker build -t flask-demo .
docker run -d --name flask-demo-container -p 5000:5000 flask-demo
docker ps
docker logs flask-demo-container
```

Open `http://localhost:5000`.

After changing code, replace the container:

```powershell
docker stop flask-demo-container
docker rm flask-demo-container
docker build -t flask-demo .
docker run -d --name flask-demo-container -p 5000:5000 flask-demo
```

## Maven WAR + Tomcat Dockerfile

This example assumes a Maven project creates a WAR and is compatible with Java 11 and Tomcat 9. Create `Dockerfile` in the project root:

```dockerfile
FROM maven:3.8.6-eclipse-temurin-11 AS build
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

FROM tomcat:9.0-jdk11
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

Build and run:

```powershell
docker build -t maven-web-java:latest .
docker images
docker run -d --name maven-app -p 8080:8080 maven-web-java:latest
docker ps
docker logs maven-app
```

Open `http://localhost:8080`.

Stop/remove:

```powershell
docker stop maven-app
docker rm maven-app
```

Adapt Java/Tomcat versions and WAR path to the project. If no WAR appears, inspect `pom.xml` packaging and the `target` directory.

---

# 4. Docker Compose practicals

Compose defines multiple services, networks and volumes in one YAML file. YAML indentation is significant: use spaces, not tabs.

## Create project (PowerShell)

```powershell
mkdir compose-lab
cd compose-lab
notepad docker-compose.yml
```

## Nginx + PostgreSQL + Redis

`docker-compose.yml`:

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    depends_on:
      - db
      - redis
    networks:
      - app-net

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo_db
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-net

  redis:
    image: redis:alpine
    networks:
      - app-net

networks:
  app-net:

volumes:
  db-data:
```

Validate, start and verify:

```powershell
docker compose config
docker compose up -d
docker compose ps
docker compose logs
```

Open `http://localhost:8080`. Nginx should show its welcome page; PostgreSQL and Redis should be running.

`depends_on` controls startup order; it does not guarantee a database is ready to accept connections.

## Simple web + MySQL template

Use the web image and environment variable names expected by the actual application:

```yaml
services:
  web:
    image: YOUR_WEB_IMAGE
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db
      DB_NAME: demo_db
      DB_USER: demo
      DB_PASSWORD: demo_password
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: demo_db
      MYSQL_USER: demo
      MYSQL_PASSWORD: demo_password
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

Inside Compose, the database hostname is `db`, not `localhost`. Some applications expect a JDBC URL such as `jdbc:mysql://db:3306/demo_db`. Never assume generic `DB_*` variables are automatically recognised by an arbitrary web image.

## Flask + PostgreSQL Compose

Create `app.py`:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from Flask + Docker Compose!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Create `Dockerfile`:

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY app.py .
RUN pip install --no-cache-dir flask
EXPOSE 5000
CMD ["python", "app.py"]
```

Create `docker-compose.yml`:

```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo_db
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

Run and browse:

```powershell
docker compose config
docker compose up --build -d
docker compose ps
docker compose logs
```

Open `http://localhost:5000`. After changing code, run `docker compose up --build -d`.

## Compose command sheet

```powershell
docker compose config                 # Validate/resolve YAML
docker compose up -d                   # Start all services detached
docker compose up --build -d           # Build and start
docker compose up                      # Foreground with logs
docker compose ps                      # List services
docker compose logs -f                 # Follow logs
docker compose stop                    # Stop, retain containers
docker compose start                   # Start stopped containers
docker compose restart                 # Restart all services
docker compose down                    # Stop/remove containers and default network
docker compose down -v                 # Also delete named volumes (data loss)
docker compose restart web             # Restart only web
docker compose up -d --build frontend  # Rebuild/start only frontend
docker compose up -d --scale web=2     # Scale service if app supports replicas
docker compose stop backend            # Stop only backend
```

### Compose scenario answers

| Requirement | Answer |
|---|---|
| Stop services but retain containers | `docker compose stop` |
| Start same containers again | `docker compose start` |
| Remove containers and default network | `docker compose down` |
| Apply changed YAML | `docker compose up -d` |
| Stop only backend | `docker compose stop backend` |
| Rebuild only frontend | `docker compose up -d --build frontend` |
| Preserve database data | Named volume; do not use `down -v` |
| Two apps need same host port | Assign different host ports, e.g. `8081:80`, `8082:80` |

To move a Compose project to another machine, copy the entire project folder (YAML, Dockerfiles, source and environment files), install Docker/Compose there, then run `docker compose config` and `docker compose up -d`.

---

# 5. Kubernetes with Minikube

Kubernetes manages containerised applications. A Pod is the smallest deployable unit. A Deployment manages replicas and replaces failed Pods. A Service provides stable access to Pods.

## Verify tools (PowerShell)

```powershell
docker --version
kubectl version --client
minikube version
```

Start Docker Desktop first.

## Install Minikube if needed

Run PowerShell as Administrator if required:

```powershell
curl.exe -LO https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe
.\minikube-installer.exe
```

Install `kubectl` separately if missing.

## Start cluster and verify

```powershell
minikube start --driver=docker
minikube status
kubectl get nodes
kubectl get pods -A
```

Alternative kubectl invocation:

```powershell
minikube kubectl -- get pods -A
```

Optional dashboard:

```powershell
minikube dashboard
```

## Nginx deployment

```powershell
kubectl create deployment mynginx --image=nginx:latest
kubectl get deployments
kubectl get pods
kubectl describe deployment mynginx
```

Expose as NodePort:

```powershell
kubectl expose deployment mynginx --type=NodePort --port=80 --target-port=80
kubectl get services
kubectl describe service mynginx
```

Access:

```powershell
minikube service mynginx --url
```

Open the URL Minikube prints. Alternative port-forward:

```powershell
kubectl port-forward service/mynginx 8081:80
```

Keep that terminal running and open `http://localhost:8081`.

## Scale and self-healing

```powershell
kubectl scale deployment mynginx --replicas=4
kubectl get deployments
kubectl get pods -o wide
```

Delete one Pod to demonstrate self-healing:

```powershell
kubectl get pods
kubectl delete pod POD_NAME
kubectl get pods -w
```

The Deployment controller creates a replacement to maintain the desired count.

## Rolling update and rollback

```powershell
kubectl set image deployment/mynginx nginx=nginx:alpine
kubectl rollout status deployment/mynginx
kubectl rollout history deployment/mynginx
kubectl rollout undo deployment/mynginx
kubectl rollout status deployment/mynginx
```

## Maven web image from Docker Hub

Use your pushed image:

```powershell
kubectl create deployment maven-app --image=YOUR_DOCKERHUB_USERNAME/maven-web-java:latest
kubectl get deployments
kubectl get pods
```

If needed, initially scale to two:

```powershell
kubectl scale deployment maven-app --replicas=2
```

For the Tomcat image in this guide, the container port is `8080`:

```powershell
kubectl expose deployment maven-app --type=NodePort --port=8080 --target-port=8080
kubectl get services
kubectl describe service maven-app
kubectl scale deployment maven-app --replicas=4
kubectl get deployments
kubectl get pods -o wide
kubectl get services
minikube service maven-app --url
```

Use the actual application container port; do not assume all images use 8080.

## Kubernetes YAML alternative

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: maven-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: maven-app
  template:
    metadata:
      labels:
        app: maven-app
    spec:
      containers:
        - name: maven-app
          image: YOUR_DOCKERHUB_USERNAME/maven-web-java:latest
          ports:
            - containerPort: 8080
```

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: maven-app
spec:
  type: NodePort
  selector:
    app: maven-app
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

Apply and verify:

```powershell
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get nodes
kubectl get deployments
kubectl get pods
kubectl get services
minikube service maven-app --url
```

NodePort range is normally `30000–32767`. Service selector must match Pod labels (`app: maven-app`).

## Troubleshooting Kubernetes

```powershell
kubectl get pods
kubectl get pods -o wide
kubectl describe pod POD_NAME
kubectl logs POD_NAME
kubectl logs POD_NAME --previous
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe deployment DEPLOYMENT_NAME
kubectl describe service SERVICE_NAME
kubectl get endpoints SERVICE_NAME
kubectl rollout status deployment/DEPLOYMENT_NAME
```

- `ImagePullBackOff`: wrong image/tag, private image or registry authentication.
- `CrashLoopBackOff`: application exits; inspect logs and configuration.
- `Pending`: insufficient resources, scheduling constraints or storage issue.
- Service endpoints `<none>`: selector mismatch or Pods not ready.
- Pods run but app inaccessible: check Service type, port, targetPort, selector, readiness and app bind address.

## Cleanup

```powershell
kubectl delete deployment mynginx
kubectl delete service mynginx
kubectl delete deployment maven-app
kubectl delete service maven-app
kubectl get all
minikube stop
```

Delete the cluster only if required:

```powershell
minikube delete
```

---

# 6. Nagios monitoring

Nagios monitors hosts and services. Common states: `OK`, `WARNING`, `CRITICAL`, `UNKNOWN`.

## Pull, run and open dashboard

PowerShell:

```powershell
docker pull jasonrivers/nagios:latest
docker run -d --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
docker ps
docker logs nagiosdemo
```

Open `http://localhost:8888`.

Lab image default login:

```text
Username: nagiosadmin
Password: nagios
```

Container management:

```powershell
docker ps
docker logs nagiosdemo
docker logs -f nagiosdemo
docker exec -it nagiosdemo bash
```

Stop/remove from PowerShell:

```powershell
docker stop nagiosdemo
docker rm nagiosdemo
docker rmi jasonrivers/nagios:latest
```

## Important paths inside container

```text
/opt/nagios/etc/nagios.cfg
/opt/nagios/etc/objects/localhost.cfg
/opt/nagios/etc/objects/commands.cfg
/opt/nagios/libexec/
```

Inspect:

```bash
ls /opt/nagios/etc/objects/
cat /opt/nagios/etc/objects/localhost.cfg
grep -n "check_" /opt/nagios/etc/objects/commands.cfg
ls /opt/nagios/libexec/
```

- `nagios.cfg`: main config and included object files.
- `localhost.cfg`: host/service definitions.
- `commands.cfg`: check command definitions.
- `libexec`: plugins.

## Common plugin checks (inside container)

```bash
/opt/nagios/libexec/check_disk -w 20% -c 10% -p /
/opt/nagios/libexec/check_load -w 5,4,3 -c 10,6,4
/opt/nagios/libexec/check_swap -w 20% -c 10%
/opt/nagios/libexec/check_procs -w 250 -c 400
/opt/nagios/libexec/check_ping -H 127.0.0.1 -w 100,20% -c 500,60%
/opt/nagios/libexec/check_http -H 127.0.0.1
/opt/nagios/libexec/check_mem.pl -u -w 80 -c 95
```

Plugin help:

```bash
/opt/nagios/libexec/check_mem.pl -h
/opt/nagios/libexec/check_uptime -h
```

For free-space disk checks, WARNING/CRITICAL thresholds indicate when free space falls below the specified levels. Exact threshold semantics depend on the plugin and whether it monitors free or used values.

HTTP `401 Unauthorized` means the HTTP server responded but requires authentication. It may be reported as a WARNING; it does not mean the Nagios process itself has failed.

## Add a check_dummy test service

Enter container:

```powershell
docker exec -it nagiosdemo bash
```

Append command definition to `commands.cfg`:

```bash
cat >> /opt/nagios/etc/objects/commands.cfg <<'EOF'

define command{
    command_name    check_dummy
    command_line    $USER1$/check_dummy $ARG1$
}
EOF
```

Append service to `localhost.cfg`:

```bash
cat >> /opt/nagios/etc/objects/localhost.cfg <<'EOF'

define service{
    use                     local-service
    host_name               localhost
    service_description     Test Alert
    check_command           check_dummy!1
}
EOF
```

`check_dummy` exit codes commonly mean `0=OK`, `1=WARNING`, `2=CRITICAL`, `3=UNKNOWN`.

Validate:

```bash
/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
```

Expected: `Total Errors: 0` and `Things look okay - No serious problems were detected during the pre-flight check`.

This lab container uses runit. Reload and check:

```bash
sv hup /etc/service/nagios
sv status /etc/service/nagios
```

Refresh the dashboard and look for `Test Alert`.

To change WARNING to CRITICAL:

```bash
sed -i 's/check_dummy!1/check_dummy!2/' /opt/nagios/etc/objects/localhost.cfg
/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
sv hup /etc/service/nagios
```

Restore OK by changing `check_dummy!2` to `check_dummy!0`, validating and reloading. If `nano` is unavailable, use `cat` heredocs or `sed`. Avoid appending duplicate definitions repeatedly.

## Nagios troubleshooting

```bash
/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
ps aux | grep '[n]agios'
sv status /etc/service/nagios
cat /opt/nagios/etc/objects/localhost.cfg
ls /opt/nagios/libexec/
```

From PowerShell:

```powershell
docker logs nagiosdemo
docker logs -f nagiosdemo
```

| State | Meaning |
|---|---|
| OK | Check succeeded |
| WARNING | Threshold/problem warning |
| CRITICAL | Serious failure/threshold crossed |
| UNKNOWN | Plugin could not determine status |
| Host UP/DOWN | Host check succeeded/failed |

A service can show WARNING even when the Nagios server is running correctly. Notifications may be disabled for an individual service while its status remains visible.

---

# 7. AWS EC2 deployment

## Launch through AWS Console

1. AWS Console → EC2 → Instances → Launch instances.
2. Name: e.g. `ubuntu-server`.
3. Choose the OS AMI specified in the question (Ubuntu in this guide).
4. Choose the required/eligible instance type.
5. Create/select a key pair and download `.pem`.
6. Configure storage.
7. Create/select a Security Group.
8. Launch; wait for Running and passing status checks.

### Security Group inbound rules

| Type | Port | Source/purpose |
|---|---:|---|
| SSH | 22 | Administrator IP only |
| HTTP | 80 | Public web access if needed |
| HTTPS | 443 | Public secure access if needed |
| Custom TCP | 6060 or 8080 | Application host port |

Open only the required ports. Avoid exposing SSH to everyone.

## Connect from Windows PowerShell

```powershell
cd "$HOME\AWS"
ssh -i "your-key.pem" ubuntu@YOUR_PUBLIC_IP
```

Use the actual key filename and public IP. If Windows reports key permissions are too open, restrict the file ACL or use EC2 Instance Connect if available.

## Install Docker and Git on Ubuntu

```bash
sudo apt update
sudo apt install -y git docker.io
sudo systemctl start docker
sudo systemctl enable docker
docker --version
git --version
sudo systemctl status docker
```

Use `sudo docker` unless Docker group permissions have been applied and you have logged out/reconnected.

Optional:

```bash
sudo usermod -aG docker ubuntu
```

## Clone, build and run Maven web app

```bash
git clone YOUR_GITHUB_REPOSITORY_URL
cd YOUR_PROJECT_FOLDER
ls
```

If there is no Dockerfile, create one using the Maven WAR/Tomcat example in Section 3.

```bash
sudo docker build -t maven-web-java:latest .
sudo docker images
sudo docker run -d --name app-demo -p 6060:8080 maven-web-java:latest
sudo docker ps
sudo docker logs app-demo
```

Open `http://YOUR_PUBLIC_IP:6060`. Security Group must allow inbound TCP 6060. In `6060:8080`, 6060 is the EC2 host port and 8080 is the container port.

## Alternative: deploy basic Apache HTML page

```bash
sudo apt update
sudo apt install -y apache2
sudo systemctl start apache2
sudo systemctl enable apache2
echo "<h1>Welcome to AWS EC2</h1>" | sudo tee /var/www/html/index.html
sudo systemctl status apache2
```

Allow inbound TCP 80 and open `http://YOUR_PUBLIC_IP`.

## Useful EC2 commands

```bash
hostname
hostname -I
ip addr
sudo systemctl status docker
sudo systemctl status apache2
sudo systemctl restart apache2
sudo docker ps
sudo docker images
sudo docker logs app-demo
sudo docker stop app-demo
sudo docker rm app-demo
sudo ss -tulpn
sudo ufw status
```

## AWS troubleshooting

- SSH fails: check instance state, public IP, route, key pair, username (`ubuntu` for Ubuntu AMI), key permissions and inbound TCP 22 from your IP.
- Browser cannot access app: check process/container, host port mapping, Security Group, OS firewall and public IP.
- No public IPv4: configure a public IPv4/Elastic IP and correct subnet route/Internet Gateway, or use an appropriate access method.
- Need stable public address: associate an Elastic IP (check AWS pricing/conditions).
- Data must survive stop/start: use persistent EBS-backed storage and keep data on that volume.
- Restrict SSH: source should be administrator IP/CIDR.
- App works locally only: bind app to `0.0.0.0` inside the container and publish the port. `localhost` inside a container refers to that container.

---

# 8. Jenkins CI/CD

Jenkins automates checkout, build, test, package, deploy and notifications.

## Start and open Jenkins

If installed as a Windows service, PowerShell:

```powershell
Get-Service *jenkins*
Start-Service Jenkins
```

Open `http://localhost:8080`.

If running in Docker:

```powershell
docker ps
docker logs jenkins
```

Use the actual container name and host port mapping.

## Initial setup and plugins

1. Unlock Jenkins using the initial admin password when asked.
2. Install suggested plugins and create an admin user.
3. Ensure required plugins are available:
   - Git
   - Pipeline
   - Maven Integration (Maven freestyle jobs)
   - Copy Artifact (artifact-copy jobs)
   - Deploy to container (WAR-to-Tomcat plugin workflow)
   - Email Extension
   - Docker Pipeline (if using its pipeline steps)

Plugin names/availability can vary by Jenkins version.

## Configure JDK and Maven

Manage Jenkins → Tools:
- Configure JDK required by project.
- Configure Maven, e.g. installation name `MAVEN_HOME`.
- Configure/detect Git.

The Maven name selected in jobs must match the configured name. Check Java/Maven compatibility with the project.

## Freestyle Maven Java Build job

Create **New Item → `MavenJava_Build` → Freestyle project**.

**Source Code Management → Git**
- Repository URL: `YOUR_GITHUB_REPOSITORY_URL`
- Branch: `*/main` or actual branch.

**Build Steps → Invoke top-level Maven targets**
- Maven: `MAVEN_HOME`
- Goals: `clean install`

**Post-build Actions**
- Archive actual output, e.g. `target/*.jar` or `target/*.war`.
- Build other projects: `MavenJava_Test`, only if stable.

Save, Build Now, inspect Console Output.

## Freestyle Maven Java Test job

Create **New Item → `MavenJava_Test` → Freestyle project**.

Ensure source/test files are available (Git checkout or correct upstream workspace/artifacts). Add Maven goal `test`. Archive reports/artifacts as required. If using Copy Artifact, configure the upstream project and correct artifact path. A Maven `test` goal needs source and test files, not merely a packaged JAR/WAR.

Save and run. Configure the Build job to trigger this job only after a stable build.

## Freestyle Maven Web Build → Test → Deploy

Create:

```text
MavenWeb_Build
MavenWeb_Test
MavenWeb_Deploy
```

### MavenWeb_Build
- Git repository and correct branch.
- Maven goals: `clean install` or `clean package` as required.
- Archive actual output, commonly `target/*.war`.
- Post-build: trigger `MavenWeb_Test` only if stable.

### MavenWeb_Test
- Ensure source/test files are available.
- Run Maven goal `test`.
- Archive reports (e.g. `target/surefire-reports/*.xml`) and publish JUnit reports if plugin is installed.
- Post-build: trigger `MavenWeb_Deploy` only if stable.

### MavenWeb_Deploy
- Copy/obtain the WAR from the upstream job.
- If using Deploy to container plugin:
  - WAR/EAR path: `target/*.war` or actual copied WAR path.
  - Context path: e.g. `webpath`.
  - Container: matching Tomcat version.
  - Tomcat URL: actual reachable Tomcat Manager URL.
  - Credentials: select credentials stored in Jenkins.
- Configure Tomcat Manager user/roles and deployment permissions.

Run Build and inspect each Console Output. Verify the deployed app at Tomcat URL/context path. `localhost` from Jenkins means the Jenkins machine/container, not necessarily a separate Tomcat host.

## Scripted Pipeline — OLTRS lab task

The lab internal specifies repository `https://github.com/archanareddyse/OLTRS.git`, Maven build/test, Docker image build and push to the student's Docker Hub repository.

Create a `Jenkinsfile` in the repository root:

```groovy
node {
    def imageName = "YOUR_DOCKERHUB_USERNAME/oltrs:${env.BUILD_NUMBER}"

    try {
        stage('Checkout') {
            git branch: 'main',
                url: 'https://github.com/archanareddyse/OLTRS.git'
        }

        stage('Build') {
            sh 'mvn clean package -DskipTests'
        }

        stage('Test') {
            sh 'mvn test'
        }

        stage('Docker Build') {
            sh "docker build -t ${imageName} ."
        }

        stage('Docker Push') {
            withCredentials([usernamePassword(
                credentialsId: 'dockerhub-creds',
                usernameVariable: 'DOCKER_USER',
                passwordVariable: 'DOCKER_PASS'
            )]) {
                sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                '''
                sh "docker push ${imageName}"
                sh 'docker logout'
            }
        }
    } catch (err) {
        currentBuild.result = 'FAILURE'
        throw err
    } finally {
        if (currentBuild.currentResult == 'FAILURE') {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed. Check ${env.BUILD_URL} for Console Output.",
                to: 'YOUR_EMAIL_ADDRESS'
            )
        }
    }
}
```

Before running:
- Replace Docker Hub username and email.
- Add Jenkins username/password credential with ID `dockerhub-creds`; use a Docker Hub access token as password.
- Agent needs Git, Java, Maven, Docker CLI and Docker daemon access.
- Ensure Dockerfile exists in repository root or adjust build context.
- Change branch if repository's branch is not `main`.
- Install/configure Email Extension Plugin for `emailext`.
- For Windows agent, replace `sh` with suitable `bat`/`powershell` steps.
- Adapt Maven goals to project. `package -DskipTests` packages without running tests; the following Test stage runs `mvn test`.

A failing stage throws an error, so later Docker stages are skipped. Do not swallow the exception.

## Docker Hub credentials in Jenkins

1. Manage Jenkins → Credentials.
2. Choose appropriate store/domain → Add Credentials.
3. Kind: Username with password.
4. Username: Docker Hub username.
5. Password: Docker Hub access token.
6. ID: `dockerhub-creds`.
7. Save.

Use `withCredentials`; never hardcode tokens/passwords in the Jenkinsfile.

## GitHub webhook with ngrok

For Jenkins running locally, PowerShell:

```powershell
ngrok http 8080
```

Keep ngrok running and copy the HTTPS forwarding URL.

In Jenkins job configuration, enable **GitHub hook trigger for GITScm polling** if using the GitHub integration setup. For Pipeline jobs, also configure Pipeline SCM/Multibranch source and the Jenkinsfile path.

In GitHub: Repository → Settings → Webhooks → Add webhook:
- Payload URL: `https://YOUR_NGROK_HOST/github-webhook/`
- Content type: `application/json`
- Events: push events.

Test:

```bash
git add .
git commit -m "Trigger Jenkins pipeline"
git push origin main
```

Check GitHub webhook delivery status and Jenkins build history.

## Email notification

1. Enable Google 2-Step Verification.
2. Create a Gmail App Password if available.
3. Install Email Extension Plugin.
4. Manage Jenkins → Configure System.
5. SMTP server `smtp.gmail.com`; port `465` with SSL or `587` with STARTTLS.
6. Configure authentication with Gmail address and App Password.
7. Send a test email.
8. Add job notification action or `emailext`.

Declarative Pipeline failure-only example:

```groovy
post {
    failure {
        emailext(
            subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "See ${env.BUILD_URL} for details.",
            to: 'YOUR_EMAIL_ADDRESS'
        )
    }
}
```

The `post` block is for a Declarative Pipeline. For Scripted Pipeline use `try/catch/finally` or currentBuild logic as above.

## Jenkins troubleshooting

| Problem | Check |
|---|---|
| Git checkout fails | URL, branch, credentials, network |
| `mvn` not found | Maven tool configuration and agent PATH |
| Build fails | Console Output, Java version, dependencies, `pom.xml` |
| Docker command missing | Docker CLI on Jenkins agent |
| Docker daemon unavailable | Agent's Docker daemon/socket permissions |
| Docker push denied | Repository name, token, credential ID |
| Webhook not triggering | GitHub delivery, ngrok URL, plugin and job trigger |
| Tomcat deploy fails | WAR path, URL, Manager roles, credentials, network |
| Email not sent | SMTP host/port, SSL/STARTTLS, App Password, plugin |

---

# 9. Troubleshooting cheat sheet

## Docker / Compose

```powershell
docker ps -a
docker logs CONTAINER
docker inspect CONTAINER
docker compose config
docker compose ps
docker compose logs
```

- Port allocated: choose a different host-side port.
- Container exits: inspect logs.
- YAML parser error: fix indentation, colons and spaces; run `docker compose config`.
- Container-to-database connection: use service name such as `db`, not `localhost`.
- Database data lost: check named volume; avoid `docker compose down -v`.

## Kubernetes

```powershell
kubectl get all
kubectl get pods
kubectl describe pod POD_NAME
kubectl logs POD_NAME
kubectl logs POD_NAME --previous
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get endpoints SERVICE_NAME
```

- Pending: inspect Pod description/events and resources/storage.
- CrashLoopBackOff: inspect current and previous logs.
- ImagePullBackOff: verify image/tag and registry access.
- No Service endpoints: selector must match Pod labels.
- App inaccessible: check Service type, port, targetPort, Pod readiness and Minikube URL.

## AWS

```bash
sudo systemctl status docker
sudo docker ps
sudo docker logs app-demo
sudo ss -tulpn
sudo ufw status
```

Check Security Group, public IP, port mapping, service state and logs.

## Jenkins

Start with the failed job's **Console Output**. Find the first real error. Check agent tools, versions, credentials, network and plugin configuration.

## Nagios

```bash
/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
ps aux | grep '[n]agios'
sv status /etc/service/nagios
```

A valid configuration should show zero errors before reload. A service WARNING is not automatically a Nagios server failure.

---

# 10. Scenario-based viva answers

## Docker / Compose

1. **Multiple unrelated apps in one container?** Prefer one main application/process per container; run separate services and connect them using Compose/networking.
2. **Flask and Nginx on host ports 5000 and 80?** Publish `5000:5000` for Flask and `80:80` for Nginx.
3. **React + Express + MongoDB?** Define three Compose services; use service names for internal communication.
4. **Two containers want host port 8080?** Only one can bind it; use different host ports.
5. **Restart all services?** `docker compose restart`.
6. **Rebuild only frontend?** `docker compose up -d --build frontend`.
7. **Tomcat connection refused?** Check Tomcat process, port publishing, listening port, firewall and application bind address.
8. **Find container using port 3000?** `docker ps`, `docker port CONTAINER`; Windows `netstat -ano | findstr :3000`.
9. **Remove Tomcat container/image?** `docker stop`, `docker rm`, then `docker rmi`.
10. **Share image?** `docker login`, tag, `docker push`; teammate uses `docker pull`.

## Kubernetes

1. **Pod repeatedly restarts?** `kubectl describe pod`, `kubectl logs`, `kubectl logs --previous`; inspect probes, command, config and resource limits.
2. **Pod Pending?** Describe Pod and inspect events, resources, PVCs, taints/tolerations and scheduling.
3. **Deployment failed?** Describe Deployment, inspect Pods, logs and events.
4. **Health checks fail?** Inspect probe paths/ports, app logs and resources.
5. **Rollback?** `kubectl rollout undo deployment/NAME`.
6. **Debug Pod?** `kubectl describe pod`, `kubectl logs`, `kubectl exec -it POD -- sh`.
7. **Expose app in Minikube?** Create NodePort Service and run `minikube service SERVICE --url`.
8. **Scale to four?** `kubectl scale deployment NAME --replicas=4`.
9. **Service endpoints empty?** Compare Service selector with Pod labels and readiness.
10. **Delete app?** `kubectl delete deployment NAME` and `kubectl delete service NAME`.

## Nagios

1. **Check Nagios process?** `ps aux | grep '[n]agios'`.
2. **View live logs?** `docker logs -f nagiosdemo`.
3. **Validate configuration?** `/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg`.
4. **New host not visible?** Confirm object file is included in `nagios.cfg`, validate syntax and reload.
5. **Advantages?** Host/service monitoring, plugin extensibility, dashboard, alerts and history.
6. **WARNING vs CRITICAL?** WARNING is a warning threshold/state; CRITICAL indicates a more serious failure/threshold state.

## AWS EC2

1. **SSH fails?** Check instance state, public IP, route, key pair, username, key permissions and inbound TCP 22 from your IP.
2. **Port 8080 works locally but not publicly?** Allow TCP 8080 in Security Group and OS firewall; verify application bind address and public IP.
3. **No public IPv4?** Configure public IPv4/Elastic IP and subnet route/Internet Gateway, or another valid access path.
4. **Data survives stop/start?** Use persistent EBS-backed storage and keep data on it.
5. **Public HTTP, admin-only SSH?** Allow TCP 80 from required clients and TCP 22 only from administrator IP/CIDR.

## Jenkins

1. **Push does not trigger?** Check webhook delivery, Jenkins URL, plugin, job trigger, branch and ngrok.
2. **Maven fails; Docker stages must skip?** Let failure throw/stop pipeline; do not swallow exception.
3. **Docker push authentication fails?** Use Jenkins Credentials and `withCredentials`, not hardcoded secrets.
4. **Deploy fails?** Inspect failed stage/Console Output; verify WAR path, Tomcat URL, Manager roles, credentials and connectivity.
5. **Email only on failure?** Declarative `post { failure { ... } }` or Scripted Pipeline failure handling.
6. **Jenkinsfile?** Pipeline-as-code file, usually stored with source.
7. **Webhook?** HTTP event notification from GitHub to Jenkins.
8. **Archive artifacts?** Retains build outputs for later stages, downloads or deployment.

---

# 11. Final practical checklist

## Docker / Compose
- [ ] Docker daemon running.
- [ ] Correct folder and filenames.
- [ ] Dockerfile has correct base image, copy paths, port and startup command.
- [ ] Compose YAML indentation valid; `docker compose config` succeeds.
- [ ] Services running and browser output verified.
- [ ] Volumes/networks configured if required.

## Kubernetes
- [ ] Minikube started; node Ready.
- [ ] Deployment exists.
- [ ] Desired replica count matches running Pods.
- [ ] Service selector and ports correct.
- [ ] App accessed using Minikube service URL.
- [ ] Cleanup done if requested.

## Nagios
- [ ] Container running on host port 8888.
- [ ] Dashboard login works.
- [ ] Host/service status inspected.
- [ ] Plugins tested.
- [ ] Config validated with zero errors.
- [ ] Reload performed and dashboard refreshed.

## AWS
- [ ] EC2 running and key pair available.
- [ ] Security Group has only required ports.
- [ ] SSH successful.
- [ ] Docker/Git installed.
- [ ] Image built and container running.
- [ ] Public IP and host port verified.

## Jenkins
- [ ] Jenkins dashboard accessible.
- [ ] Git and Maven configured.
- [ ] Repository and branch correct.
- [ ] Build/test pass.
- [ ] Artifact archived if required.
- [ ] Deployment verified.
- [ ] Docker credentials secure.
- [ ] Webhook/email tested if required.

---

# Last-minute command sheet

```text
DOCKER
docker pull IMAGE
docker run -d --name NAME -p HOST:CONTAINER IMAGE
docker ps
docker logs NAME
docker exec -it NAME bash
docker stop NAME
docker rm NAME

DOCKER COMPOSE
docker compose config
docker compose up -d
docker compose up --build -d
docker compose ps
docker compose logs -f
docker compose stop
docker compose start
docker compose down

KUBERNETES
minikube start --driver=docker
kubectl get nodes
kubectl create deployment NAME --image=IMAGE
kubectl expose deployment NAME --type=NodePort --port=PORT --target-port=PORT
kubectl scale deployment NAME --replicas=4
kubectl get deployments
kubectl get pods
kubectl get services
minikube service NAME --url
kubectl rollout undo deployment/NAME
kubectl delete deployment NAME
kubectl delete service NAME

NAGIOS
docker pull jasonrivers/nagios:latest
docker run -d --name nagiosdemo -p 8888:80 jasonrivers/nagios:latest
docker exec -it nagiosdemo bash
/opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
sv hup /etc/service/nagios
sv status /etc/service/nagios

AWS (UBUNTU)
sudo apt update
sudo apt install -y docker.io git
sudo systemctl start docker
sudo systemctl enable docker
git clone REPOSITORY_URL
sudo docker build -t app .
sudo docker run -d --name app-demo -p 6060:8080 app
sudo docker ps
sudo docker logs app-demo

JENKINS
Maven goals: clean, test, package, install
Pipeline: Checkout -> Build -> Test -> Docker Build -> Docker Push
GitHub webhook: https://YOUR_JENKINS_HOST/github-webhook/
Local Jenkins tunnel: ngrok http 8080
```

**Exam rule:** Verify every major step. A command completing without an error is not enough—show the running container, healthy Pod, successful Jenkins stage, Nagios status or working browser page.
