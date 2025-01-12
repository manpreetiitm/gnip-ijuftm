# gnip-ijuftm

# Documentation for FastAPI and MariaDB Deployment

## Overview

This documentation provides details on the Kubernetes deployment configuration for a FastAPI application and a MariaDB database, along with monitoring capabilities using Prometheus. The deployment is structured to run within a specific namespace (`mtfuji-manpreet`) and includes services for both the FastAPI application and the MariaDB database.

## Project Path:

```
/home/manpreet/ping-identity/mtfuji-project/k8s
```
## A. Kubernetes Resources For FastAPI and MariaDB

### 1. Namespace

#### `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mtfuji-manpreet
```

- **Purpose**: Creates a dedicated namespace for the FastAPI and MariaDB deployments to isolate resources.

### 2. FastAPI Deployment

#### `fastapideploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-server
  namespace: mtfuji-manpreet
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
      - name: fastapi-server
        image: 401074448412.dkr.ecr.ap-south-1.amazonaws.com/mtfuji:v1
        ports:
        - containerPort: 80
        env:
        - name: FUJI_MYSQL_USER
          value: "fuji"
        - name: FUJI_MYSQL_PASSWORD
          value: "test123"
        - name: FUJI_MYSQL_HOST
          value: "mysql-service"
        - name: FUJI_MYSQL_PORT
          value: "3306"
```

- **Purpose**: Deploys a FastAPI application with environment variables for MySQL connection.
- **Environment Variables**:
  - `FUJI_MYSQL_USER`: Username for the MySQL database.
  - `FUJI_MYSQL_PASSWORD`: Password for the MySQL database.
  - `FUJI_MYSQL_HOST`: Hostname of the MySQL service.
  - `FUJI_MYSQL_PORT`: Port for the MySQL service.

#### Service for FastAPI

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-server-service
  namespace: mtfuji-manpreet
spec:
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

- **Purpose**: Exposes the FastAPI application to the outside world via a LoadBalancer service.

### 3. MariaDB Deployment

#### `mysql-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: mtfuji-manpreet
  labels:
    app: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      initContainers:
      - name: init-script
        image: busybox
        command: ["/bin/sh", "-c"]
        args: ["cp /init/init.sql /docker-entrypoint-initdb.d/"]
        volumeMounts:
        - name: init-sql
          mountPath: /init
        - name: initdb
          mountPath: /docker-entrypoint-initdb.d
      containers:
      - name: mariadb
        image: mariadb:10.6
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "rootpassword"
        - name: FUJI_MYSQL_USER
          value: "fuji"
        - name: FUJI_MYSQL_PASSWORD
          value: "test123"
        - name: MYSQL_DATABASE
          value: "mtfuji"
        volumeMounts: 
        - name: initdb
          mountPath: /docker-entrypoint-initdb.d
        ports:
        - containerPort: 3306
      volumes:
      - name: init-sql
        configMap:
          name: mysql-init-sql 
      - name: initdb
        emptyDir: {}
```

- **Purpose**: Deploys a MariaDB instance with an initialization script to set up the database and user.
- **Environment Variables**:
  - `MYSQL_ROOT_PASSWORD`: Root password for the MariaDB instance.
  - `FUJI_MYSQL_USER`: Username for the database.
  - `FUJI_MYSQL_PASSWORD`: Password for the database.
  - `MYSQL_DATABASE`: Name of the database to create.

#### Service for MariaDB

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: mtfuji-manpreet
spec:
  selector:
    app: mariadb
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```

- **Purpose**: Exposes the MariaDB service for internal communication within the Kubernetes cluster.

### 4. Initialization SQL Script

#### `init-sql-cm.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-init-sql
  namespace: mtfuji-manpreet
data:
  init.sql: |
    CREATE USER IF NOT EXISTS 'fuji'@'%' IDENTIFIED BY 'test123';
    GRANT ALL PRIVILEGES ON *.* TO 'fuji'@'%' IDENTIFIED BY 'test123';
    CREATE DATABASE IF NOT EXISTS mtfuji;
    USE mtfuji;

    CREATE TABLE IF NOT EXISTS Photos (
        ImageID int,
        ImageName varchar(255),
        Location varchar(255)
    );

    INSERT INTO Photos (ImageID, ImageName, Location)
    VALUES (1, 'test1', 'https://via.placeholder.com/150')
    ON DUPLICATE KEY UPDATE ImageName='test1';
```

- **Purpose**: Contains the SQL commands to initialize the database, create a user, and set up a sample table with data.


## B. Commands to deploy the FastAPI and Mariadb and test the connection:

```
kubectl create -f namespace.yml
kubectl create -f init-sql-cm.yaml
kubectl create -f mysql-deploy.yaml
kubectl create -f mysql-service.yaml
kubectl create -f fastapideploy.yaml

```

```
kubectl get svc -n mtfuji-manpreet|grep fastapi
```
### Output:
```
fastapi-server-service   LoadBalancer   10.100.188.113   abc42c7059c3f426eb2379b73c21dc42-1326097048.ap-south-1.elb.amazonaws.com   80:31548/TCP     20h

```

Now, take the loadbalancer dns name and access the application using curl command and/or browser like below:

```
curl -v a2b6d7d053ca14d6086003639915df9e-1315166877.ap-south-1.elb.amazonaws.com/get_image/1
```

### Output:

```
* Host a2b6d7d053ca14d6086003639915df9e-1315166877.ap-south-1.elb.amazonaws.com:80 was resolved.
* IPv6: (none)
* IPv4: 65.0.238.101, 3.109.230.192
*   Trying 65.0.238.101:80...
* Connected to a2b6d7d053ca14d6086003639915df9e-1315166877.ap-south-1.elb.amazonaws.com (65.0.238.101) port 80
> GET /get_image/1 HTTP/1.1
> Host: a2b6d7d053ca14d6086003639915df9e-1315166877.ap-south-1.elb.amazonaws.com
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< date: Sat, 11 Jan 2025 07:46:04 GMT
< server: uvicorn
< content-length: 344
< content-type: text/html; charset=utf-8
< 

        <html>
            <head>
                <title>Congrats you have successfully setup mtfuji</title>
            </head>
            <body>
                <h1>Congrats you have successfully setup mtfuji!</h1>
                <img src=https://via.placeholder.com/150 alt="kudos" class="center">
            </body>
        </html>
* Connection #0 to host a2b6d7d053ca14d6086003639915df9e-1315166877.ap-south-1.elb.amazonaws.com left intact

```

### Checking on Browser:

![image](https://github.com/user-attachments/assets/1e600421-2173-4039-b15d-82586fec9c62)






Note: This application is accessible over http port, however for better security we would need to have application accessible over https.


## C. Considerations/approaches for application access over https:


## 1. AWS API Gateway Terraform Configuration

A Terraform configuration for setting up an AWS API Gateway that routes requests to a FastAPI application hosted on an EKS LoadBalancer. The configuration includes the upload of an SSL certificate to IAM, the creation of an API Gateway, and the deployment of the API.


## Project Path : 
```
/home/manpreet/ping-identity/mtfuji-project/terraform
```

## main.tf
```
provider "aws" {
  region = "ap-south-1"  # Use your specific AWS region
}

# Upload SSL certificate to IAM
resource "aws_iam_server_certificate" "api_gateway_cert" {
  name        = "api-gateway-cert"
  private_key = file("/home/manpreet/ping-identity/mtfuji-project/k8s/private.key")      # Path to your private key
  certificate_body = file("/home/manpreet/ping-identity/mtfuji-project/k8s/certificate.crt")  # Path to your certificate
  # Optional: Chain certificate if available (e.g., from CA)
  # certificate_chain = file("certificate_chain.crt")
}

# Create the API Gateway REST API
resource "aws_api_gateway_rest_api" "fastapi_api" {
  name        = "FastAPI-API"
  description = "API Gateway to route requests to FastAPI"
}

# Create the /get_image path resource
resource "aws_api_gateway_resource" "get_image_resource" {
  rest_api_id = aws_api_gateway_rest_api.fastapi_api.id
  parent_id   = aws_api_gateway_rest_api.fastapi_api.root_resource_id
  path_part   = "get_image"
}

# Create the GET method for the /get_image resource
resource "aws_api_gateway_method" "get_method" {
  rest_api_id   = aws_api_gateway_rest_api.fastapi_api.id
  resource_id   = aws_api_gateway_resource.get_image_resource.id
  http_method   = "GET"
  authorization = "NONE"
}

# Integration setup to point to your EKS LoadBalancer
resource "aws_api_gateway_integration" "integration" {
  rest_api_id             = aws_api_gateway_rest_api.fastapi_api.id
  resource_id             = aws_api_gateway_resource.get_image_resource.id
  http_method             = aws_api_gateway_method.get_method.http_method
  integration_http_method = "ANY"
  type                    = "HTTP"
  uri                     = "http://a1214dbf6c6d74a8c93bfb672a4b6d69-1252033744.ap-south-1.elb.amazonaws.com/"  # URL to your FastAPI app
}

# Optional: Create a custom domain for API Gateway (if you want to use your own domain)
#resource "aws_api_gateway_domain_name" "custom_domain" {
#  domain_name = "your-custom-domain.com"  # Your custom domain, if needed
#  certificate_arn = aws_iam_server_certificate.api_gateway_cert.arn  # Use the uploaded certificate ARN
#}

# Deploy the API Gateway configuration to a stage
resource "aws_api_gateway_deployment" "fastapi_deployment" {
  rest_api_id = aws_api_gateway_rest_api.fastapi_api.id
  stage_name  = "v1"
}

# Output the API Gateway URL
output "api_gateway_url" {
  value = aws_api_gateway_deployment.fastapi_deployment.invoke_url
```

## Note Points for changes in above main.tf before apply.
1. Update the Configuration:
      Modify the paths to your private key and certificate in the aws_iam_server_certificate resource

    ```
   private_key = file("/path/to/your/private.key")
   certificate_body = file("/path/to/your/certificate.crt")
   ```
2. Update the uri in the aws_api_gateway_integration resource to point to your FastAPI application:

   ```
   uri = "http://your-loadbalancer-url/"
   ```
## Steps to initiate and apply terraform changes:

```
terraform init
terraform plan
terraform apply
```
##After a successful deployment, you can access your API using the output URL:

```
terraform output api_gateway_url
```


### 5. Monitoring with Prometheus

#### `mariadb-monitoring.yaml`

```yaml
# 1. Create a Secret for mysqld-exporter credentials
apiVersion: v1
kind: Secret
metadata:
  name: mysqld-exporter-secret
  namespace: mtfuji-manpreet
type: Opaque
data:
  DATA_SOURCE_NAME: bXlzcWw6Ly9mdWppOnRlc3QxMjNAbXlzcWwtc2VydmljZTozMzA2Lw==

---
# 2. Deploy mysqld-exporter
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysqld-exporter
  namespace: mtfuji-manpreet
  labels:
    app: mysqld-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysqld-exporter
  template:
    metadata:
      labels:
        app: mysqld-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9104"
    spec:
      containers:
      - name: mysqld-exporter
        image: prom/mysqld-exporter:v0.14.0
        ports:
        - containerPort: 9104
          name: metrics
        env:
        - name: DATA_SOURCE_NAME
          valueFrom:
            secretKeyRef:
              name: mysqld-exporter-secret
              key: DATA_SOURCE_NAME
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 128Mi

---
# 3. Service for mysqld-exporter
apiVersion: v1
kind: Service
metadata:
  name: mysqld-exporter
  namespace: mtfuji-manpreet
  labels:
    app: mysqld-exporter
spec:
  ports:
  - port: 9104
    protocol: TCP
    targetPort: 9104
    name: metrics
  selector:
    app: mysqld-exporter

---
# 4. ServiceMonitor for Prometheus Operator
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mysqld-monitor
  namespace: mtfuji-manpreet
  labels:
    release: prometheus
spec:
  endpoints:
  - port: metrics
    interval: 30s
  selector:
    matchLabels:
      app: mysqld-exporter
  namespaceSelector:
    matchNames:
    - mtfuji-manpreet
```

- **Purpose**: Sets up monitoring for the MariaDB instance using the `mysqld-exporter` to expose metrics to Prometheus.
- **Components**:
  - **Secret**: Stores the credentials for the MySQL exporter.
  - **Deployment**: Deploys the `mysqld-exporter` container.
  - **Service**: Exposes the metrics endpoint.
  - **ServiceMonitor**: Configures Prometheus to scrape metrics from the exporter.


## 6. Accessing the MySQLd exporter metrics:

```
kubectl port-forward svc/mysqld-exporter -n mtfuji-manpreet 9104:9104
```

Now, from local laptop, make a tunnel to jump host 9104 port where mariadb metrics are exposed.

ssh -L 9104:localhost:9104 manpreet@3.110.191.143 

Then on local laptop browser:

http://localhost:9104/

Output:

![image](https://github.com/user-attachments/assets/cd5b810a-f8b6-431f-9c96-f126723e71c7)


Note: But we also need to have access to prometheus UI to access these metrics and run the queries:
Hence, we need to deploy below prometheus YAMLs:

## 1. prometheus-config ConfigMap:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: mtfuji-manpreet
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'mysqld-exporter'
        static_configs:
          - targets: ['mysqld-exporter:9104']  # Adjust this if your exporter service name is different
      - job_name: 'fastapi'
        static_configs:
          - targets: ['fastapi-server-service:80']  # Adjust this if your FastAPI service name is different

```

## 2. prometheus Deployment:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: mtfuji-manpreet
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
```

## 3. Prometheus Service:

```
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: mtfuji-manpreet
spec:
  type: LoadBalancer  # Change to ClusterIP if you don't need external access
  ports:
  - port: 9090
    targetPort: 9090
  selector:
    app: prometheus
```

After, the deployment, we need to again do the port forward same as above from the jump host first and then from laptop to the jump host to access the prometheus UI and run some queries like below:

## Output

![image](https://github.com/user-attachments/assets/48d7e0e8-bf91-4081-8ddc-cf5e06b81728)



## 7. Deploying Grafana and creating sample dashboard to access mariadb metrics:

1. Grafana Deployment and Service:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: mtfuji-manpreet
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: "admin"  # Change this to a secure password
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
      volumes:
      - name: grafana-storage
        emptyDir: {}  # Use a persistent volume in production
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: mtfuji-manpreet
spec:
  type: LoadBalancer
  ports:
  - port: 3000
    targetPort: 3000
  selector:
    app: grafana
```

## Grafana Dashboards with some metrics:

![image](https://github.com/user-attachments/assets/1d4f2873-9e92-43af-9eef-f4b8e0420363)


![image](https://github.com/user-attachments/assets/ec706308-7497-4c31-9457-af8420ac1286)


## Conclusion

This documentation outlines the Kubernetes resources required to deploy a FastAPI application with a MariaDB backend, including initialization scripts and monitoring capabilities. Each component is designed to work together within the specified namespace, ensuring a cohesive deployment strategy.




## TASK 2: A. List the permissions on the EC2 node.

## Project Path: /home/manpreet/ping-identity/mtfuji-project/task2-python

1. Install Boto3 Using pip:
   You can install Boto3 using pip, which is the package installer for Python. Open your terminal or command prompt and run the following command:

   ```
   pip install boto3

   ```
2. Create a python script at project path named : list-ec2-permission.py

   ## list-ec2-permission.py

```
import boto3

def get_instance_permissions(instance_id, region):
    ec2_client = boto3.client('ec2', region_name=region)
    iam_client = boto3.client('iam')

    # Get the instance profile ARN
    try:
        response = ec2_client.describe_instances(InstanceIds=[instance_id])
        instance = response['Reservations'][0]['Instances'][0]
    except Exception as e:
        print(f"Error retrieving instance information: {e}")
        return

    # Check if the instance has an IAM role
    if 'IamInstanceProfile' in instance:
        instance_profile_arn = instance['IamInstanceProfile']['Arn']
        print(f"Instance Profile ARN: {instance_profile_arn}")

        # Get the role name from the instance profile
        instance_profile_name = instance_profile_arn.split('/')[-1]
        instance_profile = iam_client.get_instance_profile(InstanceProfileName=instance_profile_name)
        role_name = instance_profile['InstanceProfile']['Roles'][0]['RoleName']
        print(f"Role Name: {role_name}")

        # List attached policies
        attached_policies = iam_client.list_attached_role_policies(RoleName=role_name)
        print("\nAttached Policies:")
        for policy in attached_policies['AttachedPolicies']:
            print(f"- {policy['PolicyName']} (ARN: {policy['PolicyArn']})")

        # List inline policies
        inline_policies = iam_client.list_role_policies(RoleName=role_name)
        print("\nInline Policies:")
        for policy_name in inline_policies['PolicyNames']:
            print(f"- {policy_name}")

    else:
        print("No IAM role associated with this instance.")

   if __name__ == "__main__":
       # Take instance ID and region as input from the user
       instance_id = input("Please enter the EC2 instance ID: ")
       region = input("Please enter the AWS region (e.g., us-east-1): ")
       get_instance_permissions(instance_id, region)
   ```

3. Run this script in below manner:

```
python list-ec2-permission.py 

```

## Output:
```
Please enter the EC2 instance ID: i-0c7213644acc4b1a5
Please enter the AWS region (e.g., us-east-1): ap-south-1
Instance Profile ARN: arn:aws:iam::401074448412:instance-profile/eks-08c968da-2d88-0096-4c82-1ee57490a6af
Traceback (most recent call last):
  File "/home/manpreet/ping-identity/mtfuji-project/task2-python/list-ec2-permission.py", line 45, in <module>
    get_instance_permissions(instance_id, region)
  File "/home/manpreet/ping-identity/mtfuji-project/task2-python/list-ec2-permission.py", line 22, in get_instance_permissions
    instance_profile = iam_client.get_instance_profile(InstanceProfileName=instance_profile_name)
  File "/home/manpreet/.local/lib/python3.9/site-packages/botocore/client.py", line 569, in _api_call
    return self._make_api_call(operation_name, kwargs)
  File "/home/manpreet/.local/lib/python3.9/site-packages/botocore/client.py", line 1023, in _make_api_call
    raise error_class(parsed_response, operation_name)
botocore.exceptions.ClientError: An error occurred (AccessDenied) when calling the GetInstanceProfile operation: User: arn:aws:sts::401074448412:assumed-role/mtfuji-testuser2-ec2-to-access-eks/i-0864e0bce6c31c847 is not authorized to perform: iam:GetInstanceProfile on resource: instance profile eks-08c968da-2d88-0096-4c82-1ee57490a6af because no identity-based policy allows the iam:GetInstanceProfile action

```
Note: Currently as my assy=umed role "arn:aws:sts::401074448412:assumed-role/mtfuji-testuser2-ec2-to-access-eks/i-0864e0bce6c31c847", doesn't have access to list the permissions hence, above error.



## TASK 2: B. Script to list all unbound volumes on the AWS Account and publish the report to slack.

1. unbound-volumes.py

```
import boto3
import requests

# Function to list unbound volumes
def list_unbound_volumes(region):
    ec2 = boto3.client('ec2', region_name=region)
    response = ec2.describe_volumes()
    
    unbound_volumes = []
    for volume in response['Volumes']:
        if volume['State'] == 'available':  # Check if the volume is unbound
            unbound_volumes.append(volume['VolumeId'])
    
    return unbound_volumes

# Function to send report to Slack
def send_report_to_slack(unbound_volumes, slack_webhook_url):
    if unbound_volumes:
        message = f"Unbound EBS Volumes:\n" + "\n".join(unbound_volumes)
    else:
        message = "No unbound EBS volumes found."

    payload = {
        'text': message
    }
    requests.post(slack_webhook_url, json=payload)

if __name__ == "__main__":
    # Take AWS region and Slack webhook URL as input from the user
    region = input("Please enter the AWS region (e.g., us-west-2): ")
    slack_webhook_url = input("Please enter your Slack webhook URL: ")

    unbound_volumes = list_unbound_volumes(region)
    send_report_to_slack(unbound_volumes, slack_webhook_url)

```

2. Run it like below:

```
python unbound-volumes.py
```

```
Please enter the AWS region (e.g., us-west-2): ap-south-1
Please enter your Slack webhook URL: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```
## Example Output:

```
Unbound EBS Volumes:
- vol-0123456789abcdef0
- vol-0abcdef1234567890

```
3. Create requirements.txt file in project directory:

   ```
   boto3
   requests
   ```
   
4. Create a Dockerfile to run this script in container env:

Note: Since the script now requires user input, we will have to set the region and Slack webhook URL as environment variables instead of prompting for them interactively.
## Dockerfile

```
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "unbound_volumes.py"]

```

4. Build image:

```
sudo docker build -t unbound-volume:latest .
[sudo] password for manpreet: 
[+] Building 13.6s (10/10) FINISHED                                                                                                                         docker:default
 => [internal] load build definition from Dockerfile                                                                                                                  0.0s
 => => transferring dockerfile: 242B                                                                                                                                  0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                                    1.4s
 => [internal] load .dockerignore                                                                                                                                     0.0s
 => => transferring context: 2B                                                                                                                                       0.0s
 => [1/5] FROM docker.io/library/python:3.9-slim@sha256:caaf1af9e23adc6149e5d20662b267ead9505868ff07c7673dc4a7166951cfea                                              3.7s
 => => resolve docker.io/library/python:3.9-slim@sha256:caaf1af9e23adc6149e5d20662b267ead9505868ff07c7673dc4a7166951cfea                                              0.0s
 => => sha256:96df0e5e81799ba220e250fc3d2c1da017b41302df5954c705bece1407dcab03 3.32MB / 3.32MB                                                                        0.6s
 => => sha256:75a2bc32319e3d6a5bc12888b074f86338a9e3a4304d99d1dddc07155b8ba76e 14.93MB / 14.93MB                                                                      1.0s
 => => sha256:caaf1af9e23adc6149e5d20662b267ead9505868ff07c7673dc4a7166951cfea 10.41kB / 10.41kB                                                                      0.0s
 => => sha256:467c454a5863379d6dc810bfdbc963e877c1a887a11dddd2fca9702d2fdf27fa 1.75kB / 1.75kB                                                                        0.0s
 => => sha256:47e5116b5ec1de854ad39b1868d3ea37555ff2ed7edb8dccc618db2427d490ae 5.28kB / 5.28kB                                                                        0.0s
 => => sha256:fd674058ff8f8cfa7fb8a20c006fc0128541cbbad7f7f7f28df570d08f9e4d92 28.23MB / 28.23MB                                                                      0.7s
 => => sha256:d381bf0bfd6e4697fb23da0e7d50d080411236ffcd2853432e12fd0f6b372778 248B / 248B                                                                            0.8s
 => => extracting sha256:fd674058ff8f8cfa7fb8a20c006fc0128541cbbad7f7f7f28df570d08f9e4d92                                                                             1.6s
 => => extracting sha256:96df0e5e81799ba220e250fc3d2c1da017b41302df5954c705bece1407dcab03                                                                             0.2s
 => => extracting sha256:75a2bc32319e3d6a5bc12888b074f86338a9e3a4304d99d1dddc07155b8ba76e                                                                             0.9s
 => => extracting sha256:d381bf0bfd6e4697fb23da0e7d50d080411236ffcd2853432e12fd0f6b372778                                                                             0.0s
 => [internal] load build context                                                                                                                                     0.0s
 => => transferring context: 650B                                                                                                                                     0.0s
 => [2/5] WORKDIR /app                                                                                                                                                0.0s
 => [3/5] COPY requirements.txt .                                                                                                                                     0.1s
 => [4/5] RUN pip install -r requirements.txt                                                                                                                         7.1s
 => [5/5] COPY . .                                                                                                                                                    0.1s 
 => exporting to image                                                                                                                                                1.1s 
 => => exporting layers                                                                                                                                               1.1s 
 => => writing image sha256:fbd2398fd09aea7853049795795b43ee161473391c16632db8546af9ac7b4ad5                                                                          0.0s 
 => => naming to docker.io/library/unbound-volume:latest                                                                                                              0.0s 

```

5. Push the docker image to ECR repository:

```
docker push your-docker-image:latest
```

6. Create a Kubernetes CronJob:

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: unbound-volumes-report
  namespace: mtfuji-manpreet
spec:
  schedule: "0 0 * * *"  # This will run the job every day at midnight UTC
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: unbound-volumes
            image: unbound-volume:latest  # Replace with your Docker image
            env:
            - name: AWS_REGION
              value: "ap-south-1"  # Replace with your desired region or make it configurable
            - name: SLACK_WEBHOOK_URL
              value: "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"  # Replace with your Slack webhook URL
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-secret
                  key: access-key-id
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-secret
                  key: secret-access-key
          restartPolicy: OnFailure
```

7. Create a Kubernetes Secret for AWS Credentials:

```
kubectl create secret generic aws-secret --from-literal=access-key-id=YOUR_ACCESS_KEY_ID --from-literal=secret-access-key=YOUR_SECRET_ACCESS_KEY
```

8. Apply the CronJob:

```
kubectl apply -f cronjob.yaml
```
