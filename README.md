# TP DevOps

## TP1 : Discover Docker


### Database

Creating the postgres database with the Dockerfile.

```Dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

# Copier les scripts SQL dans le répertoire d'initialisation de PostgreSQL
COPY CreateSchema.sql /docker-entrypoint-initdb.d/
COPY InsertData.sql /docker-entrypoint-initdb.d/
```

Building the Docker image.
```sh
docker build . -t mypostgres 
```


Creation of a network to link containers.
```sh
docker network create app-network
```


Creating the mypostgres container
```sh
docker run -p5432:5432 -d --name my_mypostgrescontainer --network app-network -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd -e POSTGRES_DB=db -v D:/EPF/4A/DevOps/TP1/database:/var/lib/mypostgresql/data mypostgres 
```

Creating the adminer container to view the database on a front-end
```sh
docker run -d --name my_adminercontainer --network app-network -p 8090:8080 adminer
```

Creating two SQL files

CreateSchema.sql
```sql
CREATE TABLE public.departments
(
 id      SERIAL      PRIMARY KEY,
 name    VARCHAR(20) NOT NULL
);

CREATE TABLE public.students
(
 id              SERIAL      PRIMARY KEY,
 department_id   INT         NOT NULL REFERENCES departments (id),
 first_name      VARCHAR(20) NOT NULL,
 last_name       VARCHAR(20) NOT NULL
);
```
InsertData.sql
```sql
INSERT INTO departments (name) VALUES ('IRC');
INSERT INTO departments (name) VALUES ('ETI');
INSERT INTO departments (name) VALUES ('CGP');


INSERT INTO students (department_id, first_name, last_name) VALUES (1, 'Eli', 'Copter');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Emma', 'Carena');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Jack', 'Uzzi');
INSERT INTO students (department_id, first_name, last_name) VALUES (3, 'Aude', 'Javel');

```


### Backend

We can see the backend DockerFile

```Dockerfile
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

Question : Why do we need a multistage build? And explain each step of this dockerfile.

They allow you to use multiple FROM statements in your Docker file. Each FROM statement can use a different base, and each starts a new step in the build. This is useful for optimizing Docker images.

#### Build Stage :
```Dockerfile
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
```
This line is pulling a Maven image with Amazon Corretto 17 JDK installed. This image is used as the base image for the build stage. The AS myapp-build part is naming this stage so it can be referred to later.

```Dockerfile
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
```
These lines are setting an environment variable MYAPP_HOME and changing the working directory to the directory specified by MYAPP_HOME.

```Dockerfile
COPY pom.xml .
COPY src ./src
```
These lines are copying the pom.xml file and the src directory from the local machine to the Docker image.

```Dockerfile
RUN mvn package -DskipTests
```
This line is running mvn package -DskipTests to build the application without running tests.

#### Deployment Stage :

```Dockerfile
# Run
FROM amazoncorretto:17
```
This line is starting a new build stage with Amazon Corretto 17 JDK as the base image.

```Dockerfile
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
```
These lines are setting an environment variable MYAPP_HOME and changing the working directory to the directory specified by MYAPP_HOME.

```Dockerfile
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
```
This line is copying the built JAR file from the myapp-build stage to the current stage.

```Dockerfile
ENTRYPOINT java -jar myapp.jar
```


### HTTP Server

We're going to configure the http server as a simple reverse proxy server in front of our application. We're going to change the elements in the conf file.
First, we're going to copy the HTTP server conf file from the Docker files, and then we're going to add these elements:

```conf
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://backendcontainer:8080/
ProxyPassReverse / http://backendcontainer:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

We need to create a Dockerfile to customize the official httpd:2.4 image in order to modify the conf and place the index.html page as the home page.

### Link application

Rather than launching a Dockerfile for each part of the application, we'll create a docker-compose.yml file.

The file will look like this:

```yml
version: '3.7'

services:
    backend:
        container_name: backendcontainer
        build: ./backend/simple-api-student
        #TODO
        networks: 
          - app-network
        depends_on:
          - database
        #TODO

    database:
        container_name: my_mypostgrescontainer
        build: ./database
        
        networks: 
          - app-network
        

    httpd:
        container_name: frontendcontainer
        build: ./frontend
        #TODO
        ports: 
        - "80:80"
        #TODO
        networks: 
          - app-network
        #TODO
        depends_on: 
        - backend
        - database
        #TODO

networks:
    app-network:
```

Question : 1-3 Document docker-compose most important commands. 1-4 Document your docker-compose file.

The most important commands for docker-compose are :

```sh
docker-compose up -d
```
It's useful for starting the application.
```sh
docker-compose down
```
It's useful for stopping and removing the application.

```sh
docker-compose build
```
It's useful for re-building the images.

Let's explain the Dockerfile. There are 3 services, which are defined as 3 containers to perform the application.
The first service is :
```Dockerfile
 backend:
        container_name: backendcontainer
        build: ./backend/simple-api-student
        #TODO
        networks: 
          - app-network
        depends_on:
          - database
        #TODO
```

We'll define the container name, where the Dockerfile is located, with “build”. We'll specify the networks so that the containers are properly linked. And the dependency on the database to respect the 3-tier architecture of the application.

The second service is the database which is configured as the backend:
```Dockerfile
database:
        container_name: my_mypostgrescontainer
        build: ./database
        
        networks: 
          - app-network
```

```Dockerfile
    httpd:
        container_name: frontendcontainer
        build: ./frontend
        #TODO
        ports: 
        - "80:80"
        #TODO
        networks: 
          - app-network
        #TODO
        depends_on: 
        - backend
        - database
        #TODO
```

We have specified ports 80:80 and dependency on the backend and database as the application architecture.

We've defined the network with the lines below.

```Dockerfile
networks:
    app-network:
```

### Publish

We're going to publish our containers on docker hub.

We connect to our account with 

```sh
docker login 
```

First, we tag our containers:
```sh
docker tag tp1-backend cyrus925/tp1-backend:1.0
docker tag tp1-httpd cyrus925/tp1-httpd:1.0
docker tag tp1-database cyrus925/tp1-database:1.0
```

Then we'll push them into docker hub:
```sh
docker push cyrus925/tp1-backend:1.0
docker push cyrus925/tp1-httpd:1.0
docker push cyrus925/tp1-database:1.0
```


## TP 2 : Discover Github Action

We're going to test our backend.

```sh
mvn clean verify --files /path/to/pom.xml
```

Question : What are testcontainers?

Testcontainers is a Java library that provides lightweight, disposable instances of databases, message brokers, and other services running in Docker containers. It is designed to support automated testing by allowing you to create and manage Docker containers directly from your test code. This approach ensures that your tests have consistent and isolated environments, which can be crucial for reliable and reproducible tests.

We're going to create a .github/workflows directory and add our main.yml file in order to test the backend.

```yml
name: CI devops 2024
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
  pull_request:

jobs:
  build-and-test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - name: Checkout code
        uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution : 'temurin'
          java-version: '17'
        #TODO

     #finally build your app with the latest command
      - name: Build and test with Maven
        run: cd backend/simple-api-student && mvn clean verify
```
We're now going to duild the docker images inside the GitHub Actions pipeline in main.yml.

```Dockerfile
  build-and-push-docker-image:
    needs: build-and-test-backend
    # Run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # Steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # Relative path to the place where source code with Dockerfile is located
          context: ./backend/simple-api-student
          # Note: tags have to be all lower-case
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp1-backend:latest
          push: ${{ github.ref == 'refs/heads/main' }}
        

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp1-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}
        

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./frontend
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp1-httpd:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```

### Sonar

We're going to set up your pipeline to use SonarCloud analysis while testing. To do this, we're going to modify the main.yml file.

```yml
 - name: Build and test with Maven
        #run: cd backend/simple-api-student && mvn clean verify
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=tpdevopsdecyrus_tp-dev-ops -Dsonar.organization=tpdevopsdecyrus -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file backend/simple-api-student/pom.xml
```


## TP 3 : Discover Ansible

To connect to the VM, we're going to create this yml file in this folder /ansible/inventories/setup.yml

```yml
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: path/to/id_rsa 
 children:
   prod:
     hosts: cyrus.larger.takima.cloud 
```

We're testing to see if it connects properly:

```sh
ansible all -i inventories/setup.yml -m ping
```

We will request your server to get your OS distribution, thanks to the setup module.

```sh
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Earlier we installed Apache httpd server on your machine, let’s remove it:
```sh
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```

### Playbooks

#### First Playbook

Let’s create a first very simple playbook in my-project/ansible/playbook.yml:

```yml
- hosts: all
  gather_facts: false
  become: true

  tasks:
   - name: Test connection
     ping:
```

Just execute the playbook:
```sh
ansible-playbook -i inventories/setup.yml playbook.yml
```

#### Advanced Playbook

We're going to divide up the playbooks for each task using playbook.yml.

```yml
- hosts: all
  gather_facts: false
  become: true
  

  roles:
    - docker
    - network
    - database
    - app
    - proxy
```

The tasks are defined as follows:

/proxy/tasks/main.yml
```yml
---
# handlers file for roles/docker

- name: pull db front
  docker_image:
    name: cyrus925/tp1-httpd
    source: pull  
  vars:
      ansible_python_interpreter: /usr/bin/python3

- name: Run db container
  docker_container:
    name: frontendcontainer
    image: cyrus925/tp1-httpd
    ports:
      - "80:80"
    networks: 
      - name: "app-network"
  vars:
      ansible_python_interpreter: /usr/bin/python3
```
/network/tasks/main.yml

```yml
---
- name: Create application network
  docker_network:
    name: app-network
    state: present
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

/docker/tasks/main.yml

```yml


# Install Docker
  - name: Install device-mapper-persistent-data
    yum:
      name: device-mapper-persistent-data
      state: latest

  - name: Install lvm2
    yum:
      name: lvm2
      state: latest

  - name: add repo docker
    command:
      cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

  - name: Install Docker
    yum:
      name: docker-ce
      state: present

  - name: Install python3
    yum:
      name: python3
      state: present

  - name: Install docker with Python 3
    pip:
      name: docker
      executable: pip3
    vars:
      ansible_python_interpreter: /usr/bin/python3

  - name: Make sure Docker is running
    service: name=docker state=started
    tags: docker

  - name: Connect to docker 
    docker_login:
        username: cyrus925
        password: Docker123
        reauthorize: yes
    vars:
      ansible_python_interpreter: /usr/bin/python3
```



/database/tasks/main.yml
```yml
---
# handlers file for roles/docker
- name: pull db image
  docker_image:
    name: cyrus925/tp1-database
    source: pull  
  vars:
      ansible_python_interpreter: /usr/bin/python3

- name: Run db container
  docker_container:
    name: my_mypostgrescontainer
    image: cyrus925/tp1-database
    networks: 
      - name: "app-network"
  vars:
      ansible_python_interpreter: /usr/bin/python3
```



/app/tasks/main.yml

```sh
---
# handlers file for roles/docker
- name: pull api image
  docker_image:
    name: cyrus925/tp1-backend
    tag: latest
    source: pull  
  vars:
      ansible_python_interpreter: /usr/bin/python3

- name: Run api container
  docker_container:
    name: backendcontainer
    image: cyrus925/tp1-backend
    networks: 
      - name: "app-network"
    state: started
  vars:
      ansible_python_interpreter: /usr/bin/python3
```



We're going to deploy our server using ansible with this command :
```sh
ansible-playbook -i ansible/inventories/setup.yml ansible/playbook.yml
```

Unfortunately, there is an error in the proxy.

We need to modify the httpd.conf file and add this line in <VirtualHost *:80>.

```conf
ServerName cyrus.larger.takima.cloud
```

### Continuous Deployment


We add this in the .github/workglows/main.yml

```yml
run-ansible:
    needs: build-and-push-docker-image
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Install Ansible
        run: sudo apt-get update && sudo apt-get install -y ansible

      - name: Set up SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.id_rsa }}" > ~/.ssh/id_rsa
        
      - name: Run Ansible Playbook
        run: ansible-playbook -i ansible/inventories/setup.yml ansible/playbook.yml
```







