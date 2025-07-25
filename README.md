# Document for onprem deployment 
### step1: install all dependencies
### Install cert-manager for SSL certificates
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
this command showing time out 
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

