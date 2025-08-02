# CLO-835 Final Project: Flask & MySQL Kubernetes Deployment

This project demonstrates a simple Flask application backed by a MySQL database deployed on Kubernetes. The app stores employee data, with persistent storage, running in AWS EKS using container images stored in AWS ECR. A background image is hosted in an AWS S3 bucket.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Clone Repository](#clone-repository)
* [Prerequisites](#prerequisites)
* [Build and Push Docker Images](#build-and-push-docker-images)
* [Upload Background Image to S3](#upload-background-image-to-s3)
* [Configure Kubernetes Manifests](#configure-kubernetes-manifests)
* [Deploy to Kubernetes](#deploy-to-kubernetes)
* [Access the Application](#access-the-application)
* [Verify Data Persistence](#verify-data-persistence)
---
```
## Folder Structure

project-root/
├── app.py
├── Dockerfile                 # For Flask app
├── Dockerfile_mysql           # For MySQL image
├── mysql.sql                 # SQL schema or initial data
├── requirements.txt          # Python dependencies for Flask app
├── static/
│   └── background.jpg        # Background image used in the app (also uploaded to S3)
├── templates/
│   ├── about.html
│   ├── addemp.html
│   ├── addempoutput.html
│   ├── getemp.html
│   └── getempoutput.html
└── manifests/
  ├── cluster.yaml          # Namespace, StorageClass or cluster-wide resources
  ├── configmap.yaml        # ConfigMap including S3 image URL and app config
  ├── flask-deployment.yaml # Deployment manifest for Flask app
  ├── mysql-service.yaml    # Service manifest for MySQL
  ├── pvc.yaml              # PersistentVolumeClaim manifest for MySQL storage
  └── sql-deployment.yaml   # Deployment manifest for MySQL pod/image

````
---


## Clone Repository

```bash
git clone https://github.com/AmaanSaiyed/clo-835-final-project.git
cd clo-835-final-project
```

---

## Prerequisites

* Docker installed
* Kubernetes cluster (EKS recommended) with `kubectl` configured
* AWS CLI configured with appropriate permissions
* AWS ECR repository created for app and sql images
* AWS S3 bucket created (private) for background image

---

## Build and Push Docker Images

1. **Login to AWS ECR:**

```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com
```

2. **Build and push Flask app image:**

```bash
docker build -t my_app -f Dockerfile .
docker tag my_app <your_account_id>.dkr.ecr.<region>.amazonaws.com/app-repo:latest
docker push <your_account_id>.dkr.ecr.<region>.amazonaws.com/app-repo:latest
```

3. **Build and push MySQL image:**

```bash
docker build -t my_db -f Dockerfile_mysql .
docker tag my_db <your_account_id>.dkr.ecr.<region>.amazonaws.com/sql-repo:latest
docker push <your_account_id>.dkr.ecr.<region>.amazonaws.com/sql-repo:latest
```

> Replace `<ECR_REGISTRY>` with your AWS ECR registry URL, for example:
> `123456789012.dkr.ecr.us-east-1.amazonaws.com`

---

## Upload Background Image to S3

1. Upload the `background.jpg` file located in the `static/` folder to your private S3 bucket.

2. Copy the **S3 Object URL** of the uploaded image.
   It should look like:
   `https://<bucket-name>.s3.<region>.amazonaws.com/background.jpg`
---

## Configure Kubernetes Manifests

* In the `manifests/configmap.yaml` file, update the `IMAGE_URL` with the S3 URL copied above.

```yaml
data:
  IMAGE_URL: "https://<bucket-name>.s3.<region>.amazonaws.com/background.jpg"
  APP_COLOR: "blue"
  APP_HEADER_NAME: "Amaan Saiyed"
  APP_HEADER_SLOGAN: "Done by Amaan!!!"
```

* In the deployment manifests (`flask-deployment.yaml` and `sql-deployment.yaml`), ensure the container images point to your pushed ECR images:

```yaml
containers:
- name: flask-app
  image: <ECR_REGISTRY>/app-repo:latest
...
- name: mysql
  image: <ECR_REGISTRY>/sql-repo:latest
```

---

## Deploy to Kubernetes

Apply manifests in this order:

```bash
kubectl apply -f manifests/cluster.yaml
kubectl apply -f manifests/pvc.yaml
kubectl apply -f manifests/mysql-service.yaml
kubectl apply -f manifests/sql-deployment.yaml
kubectl apply -f manifests/configmap.yaml
kubectl apply -f manifests/flask-deployment.yaml
```

---

## Access the Application

1. Get the external URL of the Flask service:

```bash
kubectl get svc flask-service -n fp
```

2. The EXTERNAL-IP field contains the Load Balancer URL.
   Access your app in a browser via:

```
http://<external-ip-or-dns>:81
```

---

## Verify Data Persistence

1. Add some employees through the `/addemp` route.

2. Fetch employee data via `/getemp` or `/fetchdata`.

3. Restart the MySQL pod to simulate failure and verify data is persistent:

```bash
kubectl delete pod -l app=mysql -n fp
```

4. After MySQL pod restarts, refresh the app and fetch employee data again to confirm persistence.

---

*Project by Amaan Saiyed*
---

