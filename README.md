# Coworking Space Service Extension
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

## Getting Started

### Dependencies
#### Local Environment
1. Python Environment - run Python 3.6+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster

#### Remote Resources
1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

### 1. Setup
#### 1. Configure a Database
1. Edit and apply db config map:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_NAME: 'proj3'
  DB_USERNAME: 'myuser'
  DB_HOST: 'postgresql-service'
  DB_PORT: '5432'
---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: c21lZQ==
```
```bash
kubectl apply -f deployment/config/configmap.yaml
```
2. Apply database deployment for PersistentVolumeClaim, PersistentVolume, Database service
```bash
kubectl apply -f deployment/db
```

3. Apply database service
```bash
kubectl apply -f service/db
```
4. Seed data to database
```bash
#forward port to local
kubectl port-forward service/<database-service-name> 5433:5432 &
```
```bash
# Seed data
psql --host 127.0.0.1 -U <database-user> -d <database-name> -p 5433 < db/1_create_tables.sql
psql --host 127.0.0.1 -U <database-user> -d <database-name> -p 5433 < db/2_seed_users.sql
psql --host 127.0.0.1 -U <database-user> -d <database-name> -p 5433 < db/3_seed_tokens.sql
```
### 2. Running the Analytics Application Locally
In the `analytics/` directory:

1. Install dependencies
```bash
pip install -r requirements.txt
```
2. Run the application (see below regarding environment variables)
```bash
<ENV_VARS> python app.py
```

There are multiple ways to set environment variables in a command. They can be set per session by running `export KEY=VAL` in the command line or they can be prepended into your command.

* `DB_USERNAME`
* `DB_PASSWORD`
* `DB_HOST` (defaults to `127.0.0.1`)
* `DB_PORT` (defaults to `5432`)
* `DB_NAME` (defaults to `postgres`)

The benefit here is that it's explicitly set. However, note that the `DB_PASSWORD` value is now recorded in the session's history in plaintext. There are several ways to work around this including setting environment variables in a file and sourcing them in a terminal session.

3. Verifying The Application
* Generate report for check-ins grouped by dates
```bash
curl <BASE_URL>/api/reports/daily_usage
```

* Generate report for check-ins grouped by users
```bash
curl <BASE_URL>/api/reports/user_visits
```

### 3. Application deployment
1. Set up for image pulling with private repository.
```bash
#Run this command to create secret for image pulling with repository
kubectl create secret docker-registry <secret-name> \
  --docker-server=${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password) \
  --namespace=health-check
```
2. Apply application deployment
```yaml
apiVersion: v1
kind: Service
metadata:
  name: coworking
spec:
  type: LoadBalancer
  selector:
    service: coworking
  ports:
  - name: "5153"
    protocol: TCP
    port: 5153
    targetPort: 5153
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coworking
  labels:
    name: coworking
spec:
  replicas: 1
  selector:
    matchLabels:
      service: coworking
  template:
    metadata:
      labels:
        service: coworking
    spec:
      containers:
      - name: coworking
        image: <Image URI>
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /health_check
            port: 5153
          initialDelaySeconds: 5
          timeoutSeconds: 2
        readinessProbe:
          httpGet:
            path: "/readiness_check"
            port: 5153
          initialDelaySeconds: 5
          timeoutSeconds: 5
        envFrom:
        - configMapRef:
            name: db-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
      restartPolicy: Always
#      Add created secret for private repository
      imagePullSecrets:
        - name: <secret-name>
```
### 4. Application CD/ID:
1. Set up ci/cd specifications in **analytics/buildspec.yaml**
```yaml
version: 0.2

phases:
  pre_build:
    commands:
#      Get ECR credentials and login
  build:
    commands:
#      Run docker build and tag image
  post_build:
    commands:
#      Push Image to ECR
```
2. Set up AWS CodeBuild for github
3. Once code is pushed to github, CI/CD will be triggered, docker image for latest code is then built and pushed to AWS ECR