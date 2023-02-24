
# Spring Boot Kubernetes and MySQL
Sample project to test and deploy spring boot application with mysql database in kubernetes using JKube maven plugin 

## Overview
This repository contains the project code for a simple RESTful API. The objective is to design a system API that will handle Orders data. The API provides CRUD functionality to place orders.

We have an orders table that has the following fields:

``` 
    id    : integer 
    name  : string 
    qty   : integer
    price : double
```

## Technologies Used

- Docker with kubernetes enabled
- Kubernetes command-line tool(kubectl)
- JDK 8
- Maven
## Getting Started :
### 1) Setting up mysql database

Go to mysql-k8s directory and open cmd

- Create a Secret for MySQL Login Credentials :

    ``` kubectl apply -f mysql-secret.yaml ```

- Create a ConfigMap for MySQL Configuration :

    ``` kubectl apply -f mysql-config.yml ```
-  Create a MySQL Deployment :
  
    ``` kubectl apply -f mysql-deployment.yml ```

- Test MySQL Connectivity :

   ``` kubectl get pods ```

   ``` kubectl exec -it <podName> bash ```

   ``` mysql -u username -p password ```

   ``` show databaes; ```

   the above command shows the availabe databases

### 2) Setting up Spring Boot in Kubernetes using Ingress

Go to spring-boot-k8s director and open cmd

- Building the docker file :

     ``` docker build .```
- Create springboot deployment and service at once:

    ``` kubectl apply -f app-deployment.yml```
- Configuring Ingress:

    ``` kubectl apply -f Ingress.yml ```

## Test the Application

- Use this command to get list of orders :

   ```curl -X GET http://hello-world.info/orders```

## Docker 
The project includes a dockerfile to build the API's image.

Below docker file is used to build a docker image of our spring-boot application.

```
FROM openjdk:8
EXPOSE 8080
ADD target/springboot-crud-k8s.jar springboot-crud-k8s.jar
ENTRYPOINT ["java","-jar","/springboot-crud-k8s.jar"]
```
For mysql database, mysql-5.7 image is pulled from the docker hub.

## Kubernetes

### yml files for mysql :

- mysql-secret.yml:

   Storing the username and password for database in encoded format
```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-credentials
type: Opaque
data:
  username: am9zaGFu
  password: cGFzc3dvcmQ=
  database: cHJvamVjdDE=
```
- mysql-config.yml:

   configuring the host name and database name
```
apiVersion : v1
kind : ConfigMap
metadata:
  name : db-config
data:
  host : mysql
  database: Sai
```
- mysql-service.yml:

   Connecting the pods to service of type cluster Ip
```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
  - name: mysql
    port: 3306
    targetPort: 3306
```
- mysql-deployment.yml:
   
   Deploying the pods with single replication and setting username, password, host name and database name form secrets and configmaps.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-credentials
              key: password
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-credentials
              key: username
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: database
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-credentials
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-data
        emptyDir: {}
```
### yml files for Spring-boot:

- app-deployment.yml:

   Defining pod with single replication and service with type node port and setting username,password.host name, database name form secrets and configmaps.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-crud-deployment
spec:
  selector:
    matchLabels:
      app: springboot-k8s-mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: springboot-k8s-mysql
    spec:
      containers:
        - name: springboot-crud-k8s
          image: joshann/spring-project1
          ports:
            - containerPort: 8080
          env:   # Setting Enviornmental Variables
            - name: DB_HOST   # Setting Database host address from configMap
              valueFrom :
                configMapKeyRef :
                  name : db-config
                  key :  host

            - name: DB_NAME  # Setting Database name from configMap
              valueFrom :
                configMapKeyRef :
                  name : db-config
                  key :  database

            - name: DB_USERNAME  # Setting Database username from Secret
              valueFrom :
                secretKeyRef :
                  name : mysql-credentials
                  key :  username

            - name: DB_PASSWORD # Setting Database password from Secret
              valueFrom :
                secretKeyRef :
                  name : mysql-credentials
                  key :  password

---

apiVersion: v1 # Kubernetes API version
kind: Service # Kubernetes resource kind we are creating
metadata: # Metadata of the resource kind we are creating
  name: springboot-crud-svc
spec:
  selector:
    app: springboot-k8s-mysql
  ports:
    - protocol: "TCP"
      port: 443 # The port that the service is running on in the cluster
      targetPort: 8080 # The port exposed by the service
  type: NodePort # type of the service.

```
- Ingress.yml:

   Used to configure host name and set rules to reach service based opn path
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
      - host: hello-world.info
        http:
          paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: springboot-crud-svc
                  port:
                    number: 443
```
## Contact

        Dasireddy Sai Joshan (joshan.dasireddy@argano.com)
