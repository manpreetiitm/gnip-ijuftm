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



## Note: This application is accessible over http port, however for better security we would need to have application accessible over https.

## C. Considerations/approaches for application access over https:

## 1. AWS API Gateway Terraform Configuration

A Terraform configuration for setting up an AWS API Gateway that routes requests to a FastAPI application hosted on an EKS LoadBalancer. The configuration includes the upload of an SSL certificate to IAM, the creation of an API Gateway, and the deployment of the API.


## Project Path : /home/manpreet/ping-identity/mtfuji-project/terraform

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


Accessing the prometheus db metrics:

kubectl port-forward svc/mysqld-exporter -n mtfuji-manpreet 9104:9104

Now, from local laptop, make a tunnel to jump host 9104 port where mariadb metrics are exposed.

ssh -L 9104:localhost:9104 manpreet@3.110.191.143 

Then on local laptop browser:

http://localhost:9104/

## Conclusion

This documentation outlines the Kubernetes resources required to deploy a FastAPI application with a MariaDB backend, including initialization scripts and monitoring capabilities. Each component is designed to work together within the specified namespace, ensuring a cohesive deployment strategy.
```
