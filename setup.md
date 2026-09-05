# Setup & Installation Guide

This guide details the end-to-end process of building, configuring, and deploying the MP3 Converter microservices to a local Kubernetes cluster (Minikube).

## 🗂️ Project Structure

Before deploying, familiarize yourself with the core project structure:

```text
📂 microservices/
├── 📂 auth          # Auth Service (Flask + MySQL)
│   ├── .env
│   ├── Dockerfile
│   ├── server.py
│   └── 📂 manifests # kubernetes configurations
├── 📂 converter     # Converter Engine (MoviePy)
│   ├── Dockerfile
│   ├── consumer.py
│   └── 📂 manifests # kubernetes configurations
├── 📂 gateway       # API Gateway (Flask + GridFS)
│   ├── Dockerfile
│   ├── server.py
│   └── 📂 manifests # kubernetes configurations
├── 📂 notification  # Notification Service (SMTP)
│   ├── Dockerfile
│   ├── consumer.py
│   └── 📂 manifests # kubernetes configurations
└── 📂 rabbit        # RabbitMQ Broker manifests
    └── 📂 manifests # kubernetes configurations
```

---

## 🛠️ Prerequisites

1.  **Minikube & kubectl**: Ensure Minikube is installed and running (`minikube start`).
2.  **Docker Account**: You must have a Docker Hub account to push your built images.
3.  **App Password**: E-mail account (Gmail) with a generated 16-character App Password for the SMTP notification service.
4.  **MySQL Database**: You need a running MySQL instance accessible from the Minikube cluster (e.g., hosted locally or exposed via Minikube). Ensure you know the username and password.
5.  **MongoDB Database**: You need a running MongoDB instance accessible from the Minikube cluster.

---

## 🚀 Deployment Steps

### Step 1: Network & Infrastructure Setup

**1. Configure your local Hosts File**
You must map the Minikube ingress IP (typically localhost if using the tunnel) to the required domains. Add the following to your `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):
```text
127.0.0.1 mp3converter.com
127.0.0.1 rabbitmq-manager.com
```

**2. Start Minikube Tunnel**
Run this command and leave it running in a separate terminal window to expose the ingress resources to your localhost:
```bash
minikube tunnel
```

**3. Deploy RabbitMQ**
Before the other services can connect, the message broker must be online.
```bash
cd ./src/rabbit
kubectl apply -f ./manifests
```

**4. Configure RabbitMQ Queues**
1. Wait a few moments, then navigate to [http://rabbitmq-manager.com](http://rabbitmq-manager.com).
2. Login using the default credentials:
   - Username: `guest`
   - Password: `guest`
3. Navigate to the **"Queues"** tab.
4. Add two new queues exactly as named (ensure **Type** is set to `Classic` and **Durability** is set to `Durable`):
   - `video`
   - `mp3`

---

### Step 2: Deploy Auth Service

Navigate to the Auth service directory:
```bash
cd ./src/auth
```

**1. Environment Variables (`.env`)**
Create a `.env` file in the `auth` directory with your database and JWT configurations:
```env
MYSQL_HOST=mysql
MYSQL_USER=your_username
MYSQL_PASSWORD=your_mysql_password
MYSQL_DB=your_dbname
MYSQL_PORT=3306

JWT_SECRET_KEY=your_jwt_secret_key
JWT_ALGORITHM=HS256
JWT_EXPIRES_MINUTES=1440
```

**2. Kubernetes Secrets (`secret.yaml`)**
Update `manifests/secret.yaml` to include your credentials.

```yaml
  MYSQL_PASSWORD: your_mysql_password
  JWT_SECRET: your_jwt_secret_key
```

**3. Build & Push Docker Image**
Replace `<your_dockerhub_id>` with your actual Docker Hub username.
```bash
docker build -t <your_dockerhub_id>/auth-service .
docker push <your_dockerhub_id>/auth-service
```

**4. Deploy**
Update `manifests/auth-deploy.yaml` line containing `image:` to match the image you just pushed: `image: <your_dockerhub_id>/auth-service`.
```bash
kubectl apply -f ./manifests
```

---

### Step 3: Deploy Gateway Service

Navigate to the Gateway directory:
```bash
cd ../gateway
```

**1. Build & Push Docker Image**
```bash
docker build -t <your_dockerhub_id>/gateway-service .
docker push <your_dockerhub_id>/gateway-service
```

**2. Deploy**
Update `manifests/gateway-deploy.yaml` with your newly pushed image.
```bash
kubectl apply -f ./manifests
```

---

### Step 4: Deploy Converter Engine

Navigate to the Converter directory:
```bash
cd ../converter
```

**1. Build & Push Docker Image**
```bash
docker build -t <your_dockerhub_id>/converter-service .
docker push <your_dockerhub_id>/converter-service
```

**2. Deploy**
Update `manifests/converter-deploy.yaml` with your new image.
```bash
kubectl apply -f ./manifests
```

---

### Step 5: Deploy Notification Service

Navigate to the Notification directory:
```bash
cd ../notification
```

**1. Configuration**
Open `manifests/secret.yaml`. You must provide values for your Gmail integration it should be gmail id and 16 digit app password(You can get it in Google Account -> Security -> App Passwords (2-Step Verification must be enabled)):

```yaml
  GMAIL_ADDRESS: your-email@gmail.com
  GMAIL_PASSWORD: your-16-char-app-password
```

**2. Build & Push Docker Image**
```bash
docker build -t <your_dockerhub_id>/notification-service .
docker push <your_dockerhub_id>/notification-service
```

**3. Deploy**
Update `manifests/notification-deploy.yaml` with your new image.
```bash
kubectl apply -f ./manifests
```

---

## 🧪 Testing the Application

Once all pods are running (`kubectl get pods`), use the following `curl` commands to test the pipeline.

**Note:** Ensure you have preexisting user credentials inserted into the MySQL `users` table for the login step to succeed.

### 1. Login
Returns a JWT upon successful authentication.
```bash
curl -X POST http://mp3converter.com/login -u <email>:<password>
```
*Copy the returned token string without quotes.*

### 2. Upload Video
Submit an MP4 file. Pass the token generated from the previous step.
```bash
curl -X POST -F 'file=@<path_to_your_video.mp4>' -H 'Authorization: Bearer <token>' http://mp3converter.com/upload
```
*If successful, wait a few moments and check your email inbox. You will receive an email containing the File ID (`fid`) of the converted MP3.*

### 3. Download Audio
Use the `fid` from your email alongside your JWT to retrieve the converted `.mp3` file.
```bash
curl --output mp3_download.mp3 -X GET -H 'Authorization: Bearer <token>' "http://mp3converter.com/download?fid=<fid>"
```
*The file will be saved securely to your current directory as `mp3_download.mp3`.*

---
*Back to [README.md](./README.md)*
