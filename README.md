# Keycloak ECS Deployment

This project contains AWS CloudFormation templates to deploy Keycloak on ECS with an RDS PostgreSQL database. The deployment includes an Application Load Balancer (ALB) and Route 53 DNS configuration.

## Project Structure

## CloudFormation Templates

### [cloudformation/keycloak-ecs-alb.yaml](cloudformation/keycloak-ecs-alb.yaml)

This template creates:
- An ECS cluster with a Keycloak service running on Fargate
- An Application Load Balancer (ALB) with an HTTPS listener
- A Route 53 alias record pointing to the ALB
- An Application Auto Scaling policy for the Keycloak service
- Keycloak and RDS credentials stored in AWS Secrets Manager

### [cloudformation/keycloak-vpc-rds.yaml](cloudformation/keycloak-vpc-rds.yaml)

This template creates:
- A VPC with public and private subnets
- An Internet Gateway and NAT Gateways
- Security groups for the RDS and ECS tasks
- An RDS PostgreSQL instance with Multi-AZ enabled
- Secrets in AWS Secrets Manager for the RDS and Keycloak credentials

## Docker

### [docker/Dockerfile](docker/Dockerfile)

The Dockerfile for building the Keycloak container image.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Getting Started

### Prerequisites

- AWS CLI
- AWS CloudFormation
- Docker

### Deployment

1. Deploy the VPC and RDS resources:
   ```sh
   aws cloudformation create-stack --stack-name keycloak-vpc-rds --template-body file://cloudformation/keycloak-vpc-rds.yaml --parameters ParameterKey=ProjectName,ParameterValue=keycloak

2. Deploy the ECS and ALB resources:
aws cloudformation create-stack --stack-name keycloak-ecs-alb --template-body file://cloudformation/keycloak-ecs-alb.yaml --parameters ParameterKey=VPCId,ParameterValue=<your-vpc-id> ParameterKey=PublicSubnetIds,ParameterValue=<your-public-subnet-ids> ParameterKey=ECSPrivateSubnetIds,ParameterValue=<your-private-subnet-ids> ParameterKey=ALBSecurityGroupId,ParameterValue=<your-alb-security-group-id> ParameterKey=ECSSecurityGroupId,ParameterValue=<your-ecs-security-group-id> ParameterKey=HostedZoneId,ParameterValue=<your-hosted-zone-id> ParameterKey=DomainName,ParameterValue=<your-domain-name> ParameterKey=RDSInstanceEndpoint,ParameterValue=<your-rds-endpoint> ParameterKey=DBSecretArn,ParameterValue=<your-db-secret-arn> ParameterKey=KeycloakSecretArn,ParameterValue=<your-keycloak-secret-arn> ParameterKey=CertificateArn,ParameterValue=<your-certificate-arn>