

2. Connect to the DB and Create schema
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.PostgreSQL.html#CHAP_GettingStarted.Creating.RDSPostgreSQL.EC2

```bash
sudo dnf update -y
sudo dnf install postgresql116
psql --host=keycloakdb-instance.cdce60mw24gj.us-east-1.rds.amazonaws.com --port=5432 --dbname=keycloakdb --username=dbadmin
```
```sql
CREATE SCHEMA keycloak;
```

1. Create IAM role for ECS task execution
```bash
aws iam create-role --role-name ECSTaskExecutionRole --assume-role-policy-document file://iam/ecs-trust-policy.json
```

- Attach the AmazonECSTaskExecutionRolePolicy managed policy

```bash
aws iam attach-role-policy --role-name ECSTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```


2. Create Dockerfile and Build the image
```bash
docker build -t dev-keycloak ./docker
```

3. Push to Amazon ECR
```bash
aws ecr create-repository --repository-name dev-keycloak --image-scanning-configuration scanOnPush=true --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 484907526888.dkr.ecr.us-east-1.amazonaws.com
docker tag dev-keycloak:latest 484907526888.dkr.ecr.us-east-1.amazonaws.com/dev-keycloak
docker push 484907526888.dkr.ecr.us-east-1.amazonaws.com/dev-keycloak
```
4. Create the log group
```bash
aws logs create-log-group --log-group-name /ecs/dev-keycloak --region us-east-1
```

5 Create the file task-definition.json and register the task definition
```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

6. Create a cluster
```bash
aws ecs create-cluster --cluster-name dev-keycloak-cluster 
aws ecs put-cluster-capacity-providers --cluster dev-keycloak-cluster --capacity-providers FARGATE FARGATE_SPOT  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1,base=1 
```



8. Launching a Task separately without a service. (Skip this step if you want to launch your task in a service with autoscaling and Load Balancer)
```bash
aws ecs run-task --cluster dev-keycloak-cluster --task-definition dev-keycloak-dev --count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-01a3edff4c96c1edc],securityGroups=[sg-06e4a9e7dc8c6b2b1],assignPublicIp=ENABLED}"
```

9. Disable SSL for DEV only!
```sql
select * from keycloak.REALM where name = 'master';
 |                 0 |             | master |          0 |                 | f                    | f           | f                      | f      | EXTERNAL     |             1

update keycloak.REALM set ssl_required='NONE' where name = 'master';
```

10. Create a target group
```bash
aws elbv2 create-target-group --name dev-keycloak-target-group --protocol HTTP --port 8080 --vpc-id vpc-020860057c525ea66 --target-type ip --health-check-protocol HTTPS --health-check-port 9000 --health-check-path /health --health-check-interval-seconds 30 --health-check-timeout-seconds 5 --healthy-threshold-count 3 --unhealthy-threshold-count 3
```

11. Create a load balancer
```bash
aws elbv2 create-load-balancer --name dev-keycloak-load-balancer --subnets subnet-01a3edff4c96c1edc subnet-0fede6aded311e83c subnet-0c2e1040a6ce7c636 --security-groups sg-06e4a9e7dc8c6b2b1
```

12. Create a listener
```bash
aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:484907526888:loadbalancer/app/dev-keycloak-load-balancer/bd1892354b1646c7 --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:484907526888:targetgroup/dev-keycloak-target-group/b568761164dacfb5
```

13. Create an ECS service
```bash
aws ecs create-service --cluster dev-keycloak-cluster --service-name dev-keycloak-service --task-definition dev-keycloak-dev --desired-count 1 --network-configuration "awsvpcConfiguration={subnets=[subnet-01a3edff4c96c1edc,subnet-0fede6aded311e83c,subnet-0c2e1040a6ce7c636],securityGroups=[sg-06e4a9e7dc8c6b2b1],assignPublicIp=ENABLED}"
```

14. Update the ECS service with the Load Balancer
```bash
aws ecs update-service --cluster dev-keycloak-cluster --service dev-keycloak-service --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:484907526888:targetgroup/dev-keycloak-target-group/b568761164dacfb5,containerName=dev-keycloak,containerPort=8080
```
Wait until the target in your target group is healthy. Then you can open Keycloak with the Load Balancer DNS name.

The nex steps are optional:
15. Register the ECS service as a scalable target
```bash
aws application-autoscaling register-scalable-target --service-namespace ecs --resource-id service/dev-keycloak-cluster/dev-keycloak-service --scalable-dimension ecs:service:DesiredCount --min-capacity 1 --max-capacity 5
```

16. Put a scaling policy
```bash
aws application-autoscaling put-scaling-policy --service-namespace ecs --resource-id service/dev-keycloak-cluster/dev-keycloak-service --scalable-dimension ecs:service:DesiredCount --policy-name dev-keycloak-cpu-scaling-policy --policy-type TargetTrackingScaling --target-tracking-scaling-policy-configuration file://cpu-target-tracking.json
```

17. Create an  alias record that will point dev domain name to the DNS name of the load balancer
```bash
aws route53 change-resource-record-sets --hosted-zone-id Z04002283DCEPU7BNR43J --change-batch '{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "keycloak.dev.s2ms-sn.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "ZQSVJUPU6J1EY",
          "DNSName": "dev-keycloak-load-balancer-1218414943.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }
  ]
}'
```

Check the status of the creation until the status is "INSYNC"
```bash
aws route53 get-change --id /change/C05685991I2ZR23GPA3PG
```