# Dockerizing Ruby on Rails Application
## Objectives
1. Dockrizing Ruby on Rails app and deploy it using 
    1. docker-compose 
    2. kubernetes cluster
## Prerequisites for local setup
1. Installing docker and docker-compose 
2. Kubernets cluster (Can use Minikube)
## Deploying the app using docker-compose 
The dockerfile and docker-compose files are placed on the root directory of the repo 
```bash
$ cp env-example .env
```

Start the server:

```
$ docker-compose up -d --build
```
Initialize the database 
```
$ docker­-compose run drkiq rake db:reset
$ docker-compose run drkiq rake db:migrate
```
Checking the running services
```
$ docker-compose ps 
```
```
NAME                          COMMAND                  SERVICE             STATUS              PORTS
dockerizing-ruby-drkiq-1      "/bin/sh -c 'bundle …"   drkiq               running             0.0.0.0:8010->8010/tcp, :::8010->8010/tcp
dockerizing-ruby-nginx-1      "/docker-entrypoint.…"   nginx               running             0.0.0.0:8020->8020/tcp, :::8020->8020/tcp
dockerizing-ruby-postgres-1   "docker-entrypoint.s…"   postgres            running             0.0.0.0:5432->5432/tcp, :::5432->5432/tcp
dockerizing-ruby-redis-1      "docker-entrypoint.s…"   redis               running             0.0.0.0:6379->6379/tcp, :::6379->6379/tcp
dockerizing-ruby-sidekiq-1    "bundle exec sidekiq"    sidekiq             running   
```
Browse http://localhost:8020 

**Note** <br />
You can define the environment variables in compose file with different ways <br /> 
Ex.<br />
1. Using ```environment``` in docker-compose 
```
    environment:
      LISTEN_ON: 0.0.0.0:8010
      WORKER_PROCESSES: 1
```
2. Passing multiple ```environment``` variables from an external file through to a service’s containers with the ```env_file``` option (We are using this one)
```
    env_file:
      - .env

```

## Deploying the app using Kubernetes 
## Prerequisite
Build docker image based on **Dockerfile** placed in the root directory of the repo and push it into a registry (I have used my own docker registry)
## Config files 
**kubernets-filees** folder containes all kubernetes files used to deploy app <br />
It contains 

```bash 
├── app-deployment
│   ├── app-configmap.yaml
│   ├── app-ingress.yaml
│   ├── app.yaml
│   ├── migration_job.yaml
│   └── sidekiq.yaml
├── postgres-deployment
│   ├── postgres-deployment.yaml
│   ├── postgres-pv.yaml
│   └── postgres-secrets.yaml
└── redis-deployment
    ├── redis-deployment.yaml
    └── redis-pv.yaml

```

### 1. Deploying the PostgreSQL
1. Create secrets
```
$ echo -n "drkiq" | base64
$ echo -n "test_db_password" | base64
```
2. Update the postgres-secrets.yaml file with secrets and deploy it

```
$ kubectl apply -f postgres-deployment/postgres-secrets.yaml
```
2. Deploy persistent volume and persistent volume claim
``` 
$ kubectl apply -f postgres-deployment/postgres-pv.yaml
```
3. Deploy the Postgres (Service & deployment)
```
$ kubectl apply -f postgres-deployment/postgres-deployment.yaml
```
### 2. Redis
1. Deploy persistent volume and persistent volume claim 
``` 
$ kubectl apply -f redis-deployment/redis-pv.yaml
```
3. Deploy Redis (Service & deployment)
```
$ kubectl apply -f redis-deployment/redis-deployment.yaml
```
**Notes**
1. Postgresql and Redis are a statful application so don't scale the replicaion to more than 1 
2. Use ```strategy: type: Recreate ```in the Deployment configuration YAML file. This instructs Kubernetes to not use rolling updates. Rolling updates will not work, as you cannot have more than one Pod running at a time. The Recreate strategy will stop the first pod before creating a new one with the updated configuration.
3. To configure scalable statful application (Postgresql or Redis) <br />
    1. Deploying the Postgresql and Redis clusers with primary and slaves using ``` StatefulSet ``` controller 

### 3. Ruby on Rails App
1. Deploy Configmap that contains the .env variables  
```
$ kubectl apply -f app-deployment/app-configmap.yaml
```
2. Deploy the application (Service and deployment) 
```
$ kubectl apply -f app-deployment/app.yaml
```
3. Initialize the database 
```
$ kubectl apply -f app-deployment/migration_job.yaml
```
4. Deploy the sidekiq 
```
$ kubectl apply -f app-deployment/sidekiq.yaml
```
4. Configure ingress (Using this ingress instead of nginx container in the docker-compose file )
```
$ kubectl apply -f app-ingress.yaml
```
Accessing the app with 
1. Verify the IP address is set:
```
$ kubectl get ingress -n rails-app
```
```
NAME            CLASS   HOSTS   ADDRESS        PORTS   AGE
drkiq-ingress   nginx   drkiq   192.168.49.2   80      8h
```
2. Add the following line to the bottom of the /etc/hosts file on your computer 
```
192.168.49.2 drkiq
```
**Note** 
If you need to change the Domain 
1. Update the host in  ingress file ```- host: drkiq ```
2. Update the Rails app drkiq/config/environments/development.rb ```config.hosts << "drkiq"```
