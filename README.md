
# DevOps Project - CI/CD with GitHub Actions and Ansible Deployment

https://github.com/ilianbraka/Takima_TP

## Context of the Exercise
This project was carried out to explore **GitHub Actions** and **Ansible**, enabling the setup of a complete CI/CD pipeline. The objective was to automatically test a Java application, verify its code quality, deploy Docker images to a remote registry, and manage the infrastructure using Ansible.

---

## Objectives
1. Create a **GitHub Actions workflow** for CI (build and test).
2. Integrate **Testcontainers** to test the application with database containers.
3. Set up **CD** to build and publish Docker images to DockerHub.
4. Configure a **Quality Gate** with SonarCloud to analyze code quality.
5. Use **Ansible** to deploy the infrastructure and orchestrate Docker containers.

---

## 1. CI Workflow Configuration with GitHub Actions

The `main.yml` file defines the CI/CD pipeline in the `.github/workflows` directory.

### **Workflow Structure**
```yaml
name: CI/CD DevOps 2024

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout Code
        uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'corretto'
          java-version: '17'

      - name: Build and Test with Maven
        run: mvn clean verify
        working-directory: ./simple-api

  build-and-push-docker-image:
    needs: test-backend
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout Code
        uses: actions/checkout@v2.5.0

      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push Backend Image
        uses: docker/build-push-action@v3
        with:
          context: ./simple-api
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}
```

### **Steps Explanation**
1. **Checkout Code**: Fetches the source code from the repository.
2. **Set up JDK 17**: Configures the Java environment for building the application.
3. **Build and Test**: Runs `mvn clean verify` to compile the project and execute automated tests.
4. **Build and Push**: Builds a Docker image and pushes it to DockerHub.

---

## 2. Integration with Testcontainers

### **What is Testcontainers?**
Testcontainers is a Java library that enables running Docker containers for testing purposes. Here, a PostgreSQL container is used for integration tests.

### **Maven Dependencies** (excerpt from `pom.xml`)
```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>${testcontainers.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>jdbc</artifactId>
    <version>${testcontainers.version}</version>
    <scope>test</scope>
</dependency>
```

These dependencies enable automatic deployment of a PostgreSQL container during test execution.

### **Test Execution**
The following command compiles the application and runs integration tests:
```bash
mvn clean verify
```
---

## 3. Infrastructure Configuration with Ansible

To orchestrate and deploy Docker containers, **Ansible** was used with several dedicated roles.

### **Ansible Project Structure**
```
ansible/
├── inventories/
│   └── setup.yml
├── playbook.yml
└── roles/
    ├── install_docker/
    ├── create_network/
    ├── create_volume/
    ├── launch_database/
    ├── launch_app/
    └── launch_proxy/
```

setup.yml File

The setup.yml file is an inventory file for Ansible. It defines the hosts, groups, and global variables required for executing playbooks.
Structure and Explanation:

### **Inventory File: `inventories/setup.yml`**
```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: /path/to/private/key
  children:
    prod:
      hosts:
        server_ip
```

Usage:

  Global Group (all):
  - The all group applies global variables to all hosts defined in the inventory. This ensures consistent settings across the environment.

  Sub-group (prod):
  - The prod sub-group is used to organize and target production servers. Tasks can be applied specifically to this group without affecting other groups or environments.

  SSH Private Key:
  - The specified private SSH key enables Ansible to securely connect to the hosts, ensuring authentication without requiring a password.


playbook.yml File

The playbook.yml is the master file that orchestrates the execution of roles. Each role corresponds to a specific task, such as installing Docker or starting a container.
Structure and Explanation:

### **Main Playbook: `playbook.yml`**
```yaml
- hosts: all
  become: true
  roles:
    - install_docker
    - create_network
    - create_volume
    - launch_database
    - launch_app
    - launch_proxy
```

Explanation of Directives:

  hosts: all:
  - Applies the tasks to all the hosts specified in the setup.yml inventory file. This can be replaced with a specific group (e.g., prod) to target a subset of hosts.

  become: true:
  - Grants administrative privileges (e.g., via sudo) to execute tasks that require elevated permissions, such as installing packages or managing services.

  roles:
  - Defines a sequence of roles to structure and modularize tasks. Each role encapsulates related tasks, promoting reusability and clarity in automation workflows.

Purpose of setup.yml and playbook.yml:

These files enable Ansible to:

  Organize Hosts and Environments:
  - The setup.yml inventory groups hosts into environments (e.g., production) and defines global variables, simplifying environment management.

  Deploy Applications Step-by-Step:
  - The playbook.yml orchestrates the deployment process in a structured way, utilizing reusable roles to perform tasks in logical steps, ensuring maintainability and scalability.

---

### **Roles Details**

1. Role install_docker

Purpose:

This role installs Docker and its dependencies on the target server, serving as the foundation for deploying containers.

Key Tasks:
- Install Docker and Python3-Pip:

#### 1. `install_docker`
```yaml
- name: Install Docker and dependencies
  apt:
    name:
      - docker.io
      - python3-pip
    state: present
    update_cache: yes

- name: Install Docker SDK for Python
  pip:
    name: docker
```

Explanation: 
- This task uses the apt package manager to install Docker and Pip on a Debian-based system.
- Installs the Docker SDK, enabling Ansible to interact with Docker using its modules.


2. Role create_network

Purpose:

This role creates a Docker network to facilitate communication between containers.

Key Tasks:
- Create a Docker network:

#### 2. `create_network`
```yaml
- name: Create Docker network
  community.docker.docker_network:
    name: my-network
    state: present
```

Explanation: 

This network, named my-network, allows containers (database, backend application, proxy) to communicate with each other without exposing their ports directly to the host system.

3. Role create_volume

Purpose:

This role creates Docker volumes for persistent storage, ensuring data is preserved even if containers are stopped or removed.

#### 3. `create_volume`
```yaml
- name: Create Docker volume for database
  community.docker.docker_volume:
    name: db-volume
    state: present
```

Explanation: 

This volume (db-volume) stores the database data, allowing it to persist across container restarts or redeployments.

4. Role launch_database

Purpose:

This role deploys a PostgreSQL container for the application’s database.

Key Tasks:
- Launch the PostgreSQL container:

#### 4. `launch_database`
```yaml
- name: Run PostgreSQL container
  community.docker.docker_container:
    name: my-db
    image: postgres:latest
    env:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    networks:
      - name: my-network
    state: started
```

Explanation:

- Defines environment variables required by PostgreSQL (username, password, and default database).
- Connects the container to the my-network Docker network.
- Ensures the container is started and operational.

5. Role launch_app

Purpose:

This role deploys the backend application in a Docker container.

Key Tasks:
- Launch the backend application:

#### 5. `launch_app`
```yaml
- name: Run backend application
  community.docker.docker_container:
    name: my-backend
    image: simple-api:latest
    env:
      DATABASE_URL: postgres://user:password@my-db:5432/mydb
    networks:
      - name: my-network
    state: started
```

Explanation:

- Deploys a Docker image named simple-api:latest (the backend application).
- Configures database access using the DATABASE_URL environment variable.
- Connects the container to the my-network Docker network.

6. Role launch_proxy

Purpose:

This role deploys an HTTP proxy (Apache or HTTPD) to publicly expose the backend application.

Key Tasks:
- Launch the HTTPD container:

#### 6. `launch_proxy`
```yaml
- name: Run HTTP server
  community.docker.docker_container:
    name: my-proxy
    image: httpd:latest
    ports:
      - "80:80"
    networks:
      - name: my-network
    state: started
```

Explanation:

- Deploys a container using the httpd:latest image (HTTP server).
- Exposes port 80 on the host system to make the application publicly accessible.
- Connects the container to the my-network Docker network, enabling it to forward requests to the backend application.

---

## 4. Quality Gate with SonarCloud

### **Configuration**
The Quality Gate is added via the Maven command:
```yaml
      - name: Build, Test, and Sonar Analysis
        run: mvn -B verify sonar:sonar           -Dsonar.projectKey=devops-2024           -Dsonar.organization=devops-school           -Dsonar.host.url=https://sonarcloud.io           -Dsonar.login=${{ secrets.SONAR_TOKEN }}           --file ./simple-api/pom.xml
```

---

## 5. Conclusion
This project successfully implemented a complete CI/CD pipeline using **GitHub Actions**, **Docker**, and **Ansible** to:
1. Automatically test the application.
2. Publish Docker images to DockerHub.
3. Orchestrate infrastructure deployment with Ansible.
4. Ensure code quality with SonarCloud.