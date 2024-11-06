# aws-keycloak-ecs
How to setup Keyclaok identity provider in an AWS ECS cluster

1. Create the infrastructure with the CloudFormation template vpc-rds.yaml

2. Connect to the DB and Create schema
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.PostgreSQL.html#CHAP_GettingStarted.Creating.RDSPostgreSQL.EC2

```bash
sudo dnf update -y
sudo dnf install postgresql16
psql --host=keycloakdb-instance.cdce60mw24gj.us-east-1.rds.amazonaws.com --port=5432 --dbname=keycloakdb --username=dbadmin
```
```sql
CREATE SCHEMA keycloak;
```
3. Create Dockerfile and Build the image
```bash
docker build -t dev-keycloak ./docker
```

4. Push to Amazon ECR
```bash
aws ecr create-repository --repository-name dev-keycloak --image-scanning-configuration scanOnPush=true --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 484907526888.dkr.ecr.us-east-1.amazonaws.com
docker tag dev-keycloak:latest 484907526888.dkr.ecr.us-east-1.amazonaws.com/dev-keycloak
docker push 484907526888.dkr.ecr.us-east-1.amazonaws.com/dev-keycloak
```

5. Create the ECS service with the CloudFormation template keycloak-ecs.yaml

6. Disable SSL for DEV only!
```sql
select * from keycloak.REALM where name = 'master';
 |                 0 |             | master |          0 |                 | f                    | f           | f                      | f      | EXTERNAL     |             1

update keycloak.REALM set ssl_required='NONE' where name = 'master';