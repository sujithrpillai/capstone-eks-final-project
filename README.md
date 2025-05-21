# Capstone EKS Final Project

AWS EKS Final Project

Pre-requisites

- AWS Account with a VPC
- AWS IAM user with permissions to create and manage resources in the AWS account
- AWS CLI installed and configured
- `eksctl` installed
- `kubectl` installed
- Helm installed
- Docker installed

## Procedure to implement the project

Following are the high level steps to implement the project:

1. Install the Kubernetes Cluster (Step 2 to 4 can be parallel to this step)
2. Create Docker Registry
3. Create Docker Images for the applications
   1. Events API
   2. Events Website
   3. Events DB Initializer
4. Set the `gp2` storage class as default
5. Create a Helm Chart deployment for the database
6. Run the Job to initialize the database
7. Deploy the API and Website and test the application

### Step 1: Create the Kubernetes Cluster

```bash
eksctl create cluster -f cluster.yaml
```

### Step 2: Create Docker Registry

```bash
aws ecr create-repository --repository-name e-api
aws ecr create-repository --repository-name e-web
aws ecr create-repository --repository-name e-jobs
```

### Step 3: Create Docker Images for the applications

```bash
## Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
## Create the API
cd events-api
docker build -t <ecr repo for e-api> .
docker push <ecr repo for e-api>

## Create the DB Initializer
cd ../events-db-initializer
docker build -t <ecr repo for e-jobs> .
docker push <ecr repo for e-jobs>

## Create the Website Version 1
cd ../events-website
docker build -t <ecr repo for e-web>:v1 .
docker push <ecr repo for e-web>:v1

## Modify the `events-website/test/views/layouts/default.hbs` file to change the title of the website to v2
docker build -t <ecr repo for e-web>:v2 .
docker push <ecr repo for e-web>:v2
## Modify the `events-website/test/views/layouts/default.hbs` file to change the title of the website to v3
docker build -t <ecr repo for e-web>:v3 .
docker push <ecr repo for e-web>:v3

```

### Step 4: Set the `gp2` storage class as default

Ensure that the Cluster is up and running by this time and the nodes are in `Ready` state. Then patch the `gp2` storage class to set it as default.

```bash
kubectl get nodes
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Step 5: Create a Helm Chart deployment for the database

```bash
helm install database-server oci://registry-1.docker.io/bitnamicharts/mariadb --set primary.persistence.storageClass=gp2
```

### Step 6: Run the Job to initialize the database

```bash
kubectl apply -f db_init_job.yaml
```

Check the status of the job:

```bash
kubectl get jobs
```
It should show as `Completed` once the job is done as shown below:

```bash
NAME             STATUS     COMPLETIONS   DURATION   AGE
db-initializer   Complete   1/1           3m11s      47m
```
### Step 7: Deploy the API and Website and test the application

```bash
kubectl apply -f events-api.yaml
kubectl apply -f events-website.yaml
kubectl apply -f events-website_v2.yaml # This will deploy v2 of the website
kubectl apply -f events-website_v3.yaml
```

Wait for the pods to be in `Running` state:

```bash
kubectl get pods
```

It should show the following:

```bash
NAME                             READY   STATUS      RESTARTS   AGE
database-server-mariadb-0        1/1     Running     0          55m
db-initializer-x42kz             0/1     Completed   0          49m
events-api-d5d78c67d-fhkkx       1/1     Running     0          38m
events-web-5fd77bc757-ksgwk      1/1     Running     0          37m
events-web-5fd77bc757-s85wx      1/1     Running     0          37m
events-web-v2-765f966cd7-s42sb   1/1     Running     0          32m
events-web-v3-765fc4974f-qvdmj   1/1     Running     0          32m
```

Find the external IP of the website:

```bash
kubectl get svc events-web-svc
```

Use the external IP to access the website. You should see the following:

(Refresh the page multiple times to see version 1, 2 and 3 of the website)

## Cleanup

Delete the cluster and all the resources created in the cluster:

```bash
eksctl delete cluster -f cluster.yaml
```
