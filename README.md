Building a Kubernetes-Native Control Plane with Crossplane
================================================================

This repository contains the complete setup used in the **CNCF Kuala Lumpur** live demo to showcase how **Crossplane** enables platform teams to offer **self-service infrastructure** using Kubernetes APIs.

In this demo, we build a **PostgreSQL database platform** on AWS where developers provision databases by applying a simple Kubernetes manifest — no tickets, no cloud console access.

🧩 What This Demo Covers
------------------------

*   Installing Crossplane on Kubernetes
    
*   Configuring AWS providers
    
*   Defining a **platform API** using XRDs
    
*   Implementing infrastructure logic using **Compositions**
    
*   Creating databases using developer-friendly **Claims**
    

🛠 Prerequisites
----------------

*   Kubernetes cluster (Minikube used for demo)
    
*   AWS account with programmatic access
    
*   kubectl, helm, curl installed
    
*   AWS credentials available at ~/.aws/credentials
    

1️⃣ Start Kubernetes Cluster (Minikube)
---------------------------------------
```
minikube start \   
 --cpus=2 \    
 --memory=2500 \    
 --kubernetes-version=v1.32.0 \    
 --driver=docker \    
 --force   
```

2️⃣ Install Crossplane
----------------------

### Add Crossplane Helm Repo

```
 helm repo add crossplane-stable https://charts.crossplane.io/stable  helm repo update
 ```

### Install Crossplane

> crossplane-values.yaml is already present in this repo

```
  helm install crossplane crossplane-stable/crossplane \
--version 1.20.0 \
--namespace crossplane-system \
--create-namespace \
--values crossplane-values.yaml --wait   
  ```

### Verify Installation

```
kubectl get pods -n crossplane-system   
```

3️⃣ Install Crossplane CLI (Optional but Useful)
------------------------------------------------

``` 
curl -fsSL -o /tmp/crossplane https://releases.crossplane.io/stable/v1.20.0/bin/linux_amd64/crank  
chmod +x /tmp/crossplane  
sudo mv /tmp/crossplane /usr/local/bin/crossplane   
``` 

``` 
crossplane version
``` 

4️⃣ Platform Admin Access (RBAC)
--------------------------------

Apply platform admin permissions:

```   
kubectl apply -f platform-admin-role.yaml

kubectl apply -f platform-admin-rolebinding.yaml   
```

Verify:
```   
kubectl get clusterroles | grep platform-admin  
```

5️⃣ Create Namespaces (Developer Environments)
----------------------------------------------
```   
kubectl create ns dev  --as=system:serviceaccount:platform-team:platform-admin 
kubectl create ns stage  --as=system:serviceaccount:platform-team:platform-admin 
```

6️⃣ Create AWS Credentials Secret
---------------------------------

Crossplane uses this secret to authenticate with AWS.

``` 
kubectl create secret generic aws-secret \
--namespace crossplane-system \
--from-file=credentials=$HOME/.aws/credentials \
--as=system:serviceaccount:platform-team:platform-admin
  ```

Verify:

```   
kubectl get secret aws-secret -n crossplane-system
```

7️⃣ Install AWS Providers
-------------------------

At this stage, no providers exist:

```   
kubectl get providers   
```

### Install EC2 Provider

```   
kubectl apply -f aws-ec2-provider.yaml   --as=system:serviceaccount:platform-team:platform-admin   
```

### Install RDS Provider

```   
kubectl apply -f aws-rds-provider.yaml  --as=system:serviceaccount:platform-team:platform-admin   
```

8️⃣ Configure AWS Provider Authentication
-----------------------------------------

``` 
kubectl apply -f aws-provider-config.yaml  --as=system:serviceaccount:platform-team:platform-admin   
```

Wait for providers to be healthy:
```   
kubectl wait --for=condition=Healthy provider/upbound-provider-aws-ec2 --timeout=300s  
kubectl wait --for=condition=Healthy provider/upbound-provider-aws-rds --timeout=300s   
```

Verify:
```   
kubectl get providers  
kubectl get providerconfig   
```

9️⃣ Define the Platform API (XRD)
---------------------------------

The XRD defines **what developers can request**.

```   
kubectl apply -f database-xrd.yaml \
--as=system:serviceaccount:platform-team:platform-admin   
```

Verify:

```   
kubectl get xrds  
kubectl get crds | grep -E "(xdatabase|databaseclaim)"   
```

🔟 Define Infrastructure Logic (Composition)
--------------------------------------------

The Composition defines **how infrastructure is created**.

```   
kubectl apply -f database-composition.yaml \
--as=system:serviceaccount:platform-team:platform-admin   
```

Verify:

```   
kubectl get composition  
kubectl describe composition xdatabases.aws.platform.iamvasil.com   
```

1️⃣1️⃣ Create Database Claims (Developer Experience)
----------------------------------------------------

Developers only apply **simple YAML** — no AWS knowledge required.

```   
kubectl apply -f test-claims.yaml \
--as=system:serviceaccount:platform-team:platform-admin   
```

Check claim status:
```   
kubectl get databaseclaims -n dev  
kubectl get databaseclaims -n stage   
```

Check underlying composite resources:

```   
kubectl get xdatabases  
kubectl describe xdatabases   
```

Now use Crossplane's trace command to visualize the complete resource hierarchy from your Claims down to the AWS managed resources:
```
crossplane beta trace databaseclaim dev-app-db -n dev
```

This powerful command shows the complete relationship chain:

DatabaseClaim (what the developer created)
XDatabase (the composite resource created by Crossplane)
12 AWS managed resources (VPC, subnets, security groups, RDS instance, etc.)
The trace output reveals how your simple 4-parameter claim automatically provisions a complete AWS database infrastructure with proper networking, security, and high availability configuration.

Sample output:
```
controlplane ~ on ☁️  (us-east-1) ✦2 ➜  crossplane beta trace databaseclaim dev-app-db -n dev
NAME                                        SYNCED   READY   STATUS
DatabaseClaim/dev-app-db (dev)              True     False   Waiting: Claim is waiting for composite resource to become Ready
└─ XDatabase/dev-app-db-wlgjt               True     False   Creating: Unready resources: database-instance
   ├─ InternetGateway/testapp-igw           True     True    Available
   ├─ RouteTableAssociation/testapp-rta-1   True     True    Available
   ├─ RouteTableAssociation/testapp-rta-2   True     True    Available
   ├─ RouteTable/testapp-rt                 True     True    Available
   ├─ Route/testapp-route                   True     True    Available
   ├─ SecurityGroupRule/testapp-sg-egress   True     True    Available
   ├─ SecurityGroupRule/testapp-sg-rule     True     True    Available
   ├─ SecurityGroup/testapp-sg              True     True    Available
   ├─ Subnet/testapp-subnet-1               True     True    Available
   ├─ Subnet/testapp-subnet-2               True     True    Available
   ├─ VPC/testapp-vpc                       True     True    Available
   ├─ Instance/testapp-db                   True     False   Creating
   └─ SubnetGroup/testapp-subnet-group      True     True    Available
```
![RDS created](https://github.com/Vasil-Shaikh/crossplane-demo/blob/main/RDS.png)

Finally, the dev team retrieves the DB details from their own namespace, for e.g
```
k get secrets -n dev
```
```
k describe secrets -n dev 9d5f7d21-9120-448b-b35d-cff9f061f0c7-postgresql
```
(replace with your secret name)

```
kubectl get secret 9d5f7d21-9120-448b-b35d-cff9f061f0c7-postgresql \
  -n dev \
  -o json | jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'
```

Sample Output
```
address: terraform-20260209142436023500000003.cpcsu8620we9.us-east-1.rds.amazonaws.com
attribute.password: 6QtmP7dXXXXXXXXXXXXX
endpoint: terraform-20260209142436023500000003.cpcsu8620we9.us-east-1.rds.amazonaws.com:5432
host: terraform-20260209142436023500000003.cpcsu8620we9.us-east-1.rds.amazonaws.com
password: 6QtmP7dmXXXXXXXXXXXXX
port: 5432
username: postgres
```

🎯 What This Demonstrates
-------------------------

*   Self-service infrastructure
    
*   Environment-aware configurations
    
*   Secure credential delivery
    
*   Clear separation between **platform** and **developer** concerns
    
*   Kubernetes as a **control plane**, not just a scheduler
    

⚠️ Important Notes
------------------

> This setup is **for demo/lab purposes only**Some configurations (public access, wide CIDRs) are **NOT production-ready**

📌 Next Steps (Not Covered in Demo)
-----------------------------------

*   RBAC & multi-tenant isolation
    
*   GitOps integration (Argo / Flux)
    
*   Backstage integration for IDP
    
*   Crossplane v2 improvements
