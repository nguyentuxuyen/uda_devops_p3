# Coworking Space Service Extension
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

## Getting Started

### Dependencies
#### Local Environment
1. Python Environment - run Python 3.6+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster
4. `helm` - apply Helm Charts to a Kubernetes cluster

#### Remote Resources
1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

### Setup
#### 1. Create AWS ECR.


    
    aws ecr create-repository --repository-name $PROJECT_NAME --image-scanning-configuration scanOnPush=true --region us-east-1
    
#### 2. Create CodeBuild.
 Follow the lesson and below link.
https://docs.aws.amazon.com/codebuild/latest/userguide/github-webhook.html
#### 3. Create resource network.

    aws cloudformation create-stack --stack-name $PROJECT_NAME-network --template-body file://deployment/network.yaml --capabilities "CAPABILITY_IAM" "CAPABILITY_NAMED_IAM" --region us-east-1

#### 4. Create EKS.

1. Update value output from step 3 into file eks.yaml
2. Setup an AWS EKS Cluster
    ```
    eksctl create cluster -f deployment/eks.yaml 
    ```
3. Set up Postgres database
    ```
    helm repo add $PROJECT_NAME https://charts.bitnami.com/bitnami
    ```

    ```
    helm install postgresql $PROJECT_NAME/postgresql --set primary.persistence.enabled=false
    ```

4. Get the password for "postgres"
    ```
    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
    ```

5. Connecting Via Port Forwarding
    ```
    kubectl port-forward --namespace default svc/$PROJECT_NAME-postgresql 5432:5432
    ```

6. Create tables and insert data into tables
    ```
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < db/1_create_tables.sql
    ```
    ```
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < db/2_seed_users.sql
    ```
    ```
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < db/3_seed_tokens.sql
    ```
#### 5. Test Application 

1. Get external ip

    ```
    kubectl get svc
    ```

2. Access the URL and check response code

    ```
    http://<EXTERNAL-IP>:5153/health_check
    http://<EXTERNAL-IP>:5153/readiness_check
    http://<EXTERNAL-IP>:5153/api/reports/daily_usage
    http://<EXTERNAL-IP>:5153/api/reports/user_visits
    ```

#### 6. Setup Cloudwatch

* 
    ```
    ClusterName=$PROJECT_NAME
    LogRegion='us-east-1'
    FluentBitHttpPort='2020'
    FluentBitReadFromHead='Off'
    [[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
    [[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
    curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${LogRegion}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f - 
    ```