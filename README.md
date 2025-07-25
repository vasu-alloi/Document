# Document for onprem deployment 
### step1: install all dependencies
#### Install cert-manager for SSL certificates
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```
#### Install NGINX ingress controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```
#### Wait for services to be ready
```
kubectl wait --for=condition=available --timeout=300s deployment -n cert-manager --all
```
```
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=ingress-nginx -n ingress-nginx --timeout=300s
```
### Step 2: Create SSL Certificate Issuer (Let's Encrypt)
This will allow Kubernetes to automatically generate SSL certificates using Let's Encrypt.
```
cat << EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@domain.com  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```
This will be used to issue certificates for your domain via cert-manager.
### Step 3: Set Up PostgreSQL Databases & Users
#### Connect to your PostgreSQL instance (via psql, DBeaver, or pgAdmin) and run the following:
If your PostgreSQL is running on a different host or port:
```
psql -h your_host -p your_port -U your_username -d your_database
```
It will prompt for a password.Put your Database password 
Then enter into your database and perform these actions by replace your DB's,Users and Passwords.
```
-- Create databases
CREATE DATABASE "alloi-embeddings";
CREATE DATABASE "alloi-jackson";
CREATE DATABASE "alloi-supertokens";
CREATE DATABASE "alloi-backend";

-- Create users
CREATE USER embeddings_user WITH PASSWORD 'secure_password_1';
CREATE USER jackson_user WITH PASSWORD 'secure_password_2';
CREATE USER supertokens_user WITH PASSWORD 'secure_password_3';
CREATE USER backend_user WITH PASSWORD 'secure_password_4';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE "alloi-embeddings" TO embeddings_user;
GRANT ALL PRIVILEGES ON DATABASE "alloi-jackson" TO jackson_user;
GRANT ALL PRIVILEGES ON DATABASE "alloi-supertokens" TO supertokens_user;
GRANT ALL PRIVILEGES ON DATABASE "alloi-backend" TO backend_user;
```
### Step 4: Clone and Prepare Alloi Helm Charts
```
# Clone the charts repo
git clone https://github.com/opshealth/alloi-public-charts.git
cd alloi-public-charts

# Update Helm chart dependencies
helm dependency update ./alloi-stack

# Create namespace for Alloi services
kubectl create namespace alloi
```
### Step 5: Configure DNS
Get the external IP of your ingress controller:
```
kubectl get svc -n ingress-nginx ingress-nginx-controller
```
<img width="1190" height="81" alt="image" src="https://github.com/user-attachments/assets/cd080ab7-3aa6-4e04-899b-a128e04c8106" />
When deploying a Kubernetes application on your local machine (e.g., using docker-desktop,Minikube or Kind), and exposing it using a nip.io domain with HTTPS, your browser might show the following error:
<img width="1560" height="680" alt="image" src="https://github.com/user-attachments/assets/2e183314-f812-4fd1-b5d3-bac4fa3a8f3f" />
#### step:6 Edit values.yaml
Edit the ./alloi-stack/values.yaml file with your environment-specific values:
```
global:
  domain: yourdomain.com
  hostname: alloi.yourdomain.com
  storage_class: "gp2"
  s3_bucket: your-s3-bucket-name
  aws_region: us-west-2
  database_host: your-rds-endpoint.region.rds.amazonaws.com
  database_port: "5432"

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: alloi.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - alloi.yourdomain.com
      secretName: alloi-tls

```
### Step 7: Edit values-secrets.yaml
Create a values-secrets.yaml file to store sensitive values:
```
global:
  backend:
    secretVariables:
      DATABASE_NAME: "alloi-backend"
      DATABASE_USER: "backend_user"
      DATABASE_PASSWORD: "secure_password_4"
      DJANGO_SECRET_KEY: "your-50-character-secret-key"
      CRYPTOGRAPHY_KEY: "your-32-byte-encryption-key"
      OPENAI_API_KEY: "sk-your-openai-api-key"
      AWS_ACCESS_KEY_ID: "your-aws-access-key"
      AWS_SECRET_ACCESS_KEY: "your-aws-secret-key"
      REDIS_PASSWORD: "redis-secure-password"
      RABBITMQ_PASSWORD: "rabbitmq-secure-password"

embeddings:
  secretVariables:
    DATABASE_NAME: "alloi-embeddings"
    DATABASE_USER: "embeddings_user"
    DATABASE_PASSWORD: "secure_password_1"
    OPENAI_API_KEY: "sk-your-openai-api-key"

supertokens:
  database:
    name: "alloi-supertokens"
    host: "your-rds-endpoint.region.rds.amazonaws.com"
    port: 5432
    user: "supertokens_user"
    password: "secure_password_3"

rabbitmq:
  auth:
    username: "alloi-rabbitmq"
    password: "rabbitmq-secure-password"
    erlangCookie: "your-erlang-cookie"

redis:
  global:
    redis:
      password: "redis-secure-password"
```
Keep this file safe! Do not commit this to GitHub.
### Step 8: Deploy Alloi Using Helm
```
helm install alloi ./alloi-stack -n alloi \
  -f ./alloi-stack/values.yaml \
  -f ./alloi-stack/values-secrets-template.yaml
  ```
This will deploy all the services in the Alloi stack to your Kubernetes cluster.
### Step 9: Verify Deployment
Check pods:
```
kubectl get pods -n alloi
```
Check services:
```
kubectl get svc -n alloi
```
Check certificate status:
```
kubectl describe certificate -n alloi
```
Access the domain:
```
https://alloi.yourdomain.com
```
Test access
```
curl -k https://alloi.yourdomain.com
```
