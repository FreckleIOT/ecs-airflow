# Airflow on ECS

Infrastrucuture scripts to orchestrate an Apache Airflow cluster in ECS

These scripts are ported from [aws-infrastructure](https://github.com/FreckleIOT/aws-infrastructure)

## Prerequisites

### AWS CLI

The scripts rely on AWS CLI. You can configure a file:

`~/.aws/credentials`:
```
[default]
aws_access_key_id=changeme
aws_secret_access_key=changeme
region=us-west-2
```

If you prefer, AWS Security Token Service (STS) as well.

### Encrypting sensitive passwords

The bash script `kms-encrypt` can be used to create a new KMS data key
and store the cipher text in a JSON configuration file if it doesn't
already exist. The KMS plaintext key is then used to configure a
sensitive configuration value for a given parameter.

```bash
kms_key_id=changeme
./kms-encrypt --kms-key-id=${kms_key_id} \
    --param=ParameterKey --secret='changeme' \
    --config-path=path-to-config.json
```

__Note:__ This script uses a docker container from the `python:3.6-slim`
image which has OpenSSL 1.1.0. This version is the same as the one
used by `puckel/docker-airflow:1.9.0-4` which is used for our ECS
docker image. If there is a mismatch between OpenSSL versions the
containers will not be able to decrypt passwords and keys.

## Deploying Cloudformation Stack

### Build the docker image

Run the following command:
```bash
aws ecr get-login --no-include-email --region us-west-2
```
It will return a command to login to ECR. Run that command to login.

To build the image make sure to change the Account ID:
```bash
export AWS_REGION=us-west-2
aws_account_id=changeme
./build-docker-image ${aws_account_id}
```

### Postgres (RDS)

Create a JSON file `dev-airflow-postgres.json` to override non-default
parameters. Note that `SubnetIds` are in the private subnet and we have
included instructions below on how to connect to it over an SSH tunnel.
```json
{
    "Parameters": {
      "VpcId": "vpc-xxxxxxxx",
      "SubnetIds": "subnet-aaaaaaaa,subnet-bbbbbbbb",
      "DatabaseName": "airflow",
      "StorageInGb": 100,
      "StorageIops": 1000,
      "PostgresVersion": "10.3",
      "DbInstanceType": "db.t2.small",
      "AllowedCIDR": "10.0.0.0/16",
      "BackupRetentionInDays": "7",
      "MultiAZDeployment": "true",
      "PostgresMasterUsername": "airflow_user",
      "KmsKeyId": "arn:aws:kms:us-west-2:************:key/********-****-****-****-************",
      "Project": "Freckle IoT",
      "Team": "Freckle",
      "Environment": "dev",
      "Component": "Airflow"
    }
}
```

Deploy the Cloudformation template:
```bash
SENSITIVE_PARAMS='"PostgresMasterPassword=changeme"' ./deploy-stack \
    cloudformation/postgres-rds.cloudformation.yaml \
    dev-airflow-postgres ../ecs-airflow-config/dev-airflow-postgres.json
```
__Note:__ We send in the password in this manner because kms-encrypt
won't help in this case and also helps to keep these sensitive passwords
outside of source control.

SSH Tunnel to RDS:
```bash
ssh -i path-to-pem -N -L 5432:postgres-end-point:5432 ec2-user@bastion-host
```

Run the postgresql client (you might need to install the client first
for the target system):
```
psql -h 127.0.0.1 -d airflow -U airflow_user
```

Setup the schema:
```sql
CREATE SCHEMA IF NOT EXISTS airflow AUTHORIZATION airflow_user;
ALTER ROLE airflow_user SET search_path TO airflow;
GRANT USAGE ON SCHEMA airflow TO airflow_user;
GRANT CREATE ON SCHEMA airflow TO airflow_user;
```

### Redis (ElastiCache)

Create a JSON file `dev-airflow-redis.json` to override non-default
parameters. Note that the `SubnetIds` are the same as `SubnetIds` for
the RDS cluster and are private subnets.
```json
{
    "Parameters": {
      "VpcId": "vpc-xxxxxxxx",
      "SubnetIds": "subnet-aaaaaaaa,subnet-bbbbbbbb",
      "RedisCacheNodeType": "cache.t2.small",
      "RedisVersion": "4.0.10",
      "AllowedCIDR": "10.0.0.0/16",
      "Project": "Airflow Project",
      "Team": "Airflow Team",
      "Environment": "dev",
      "Component": "Airflow"
     }
}
```

Deploy the Cloudformation template:
```bash
./deploy-stack cloudformation/redis-cluster.cloudformation.yaml \
    dev-airflow-redis ../ecs-airflow-config/dev-airflow-redis.json
```

### ECS Cluster

Create a JSON configuration file `dev-airflow-ecs.json`. Note that
the `InstanceSubnetIds` are the same as `SubnetIds` for the RDS
cluster and are private subnets. The `LoadBalancerSubnetIds` are public
subnets.
```json
{
    "Parameters": {
      "VpcId": "vpc-xxxxxxxx",
      "InstanceSubnetIds": "subnet-aaaaaaaa,subnet-bbbbbbbb",
      "LoadBalancerSubnetIds": "subnet-cccccccc,subnet-dddddddd",
      "EcsInstanceType": "m5.large",
      "UseSSL": "yes",
      "BastionStack": "changeme",
      "CertificateArn": "arn:aws:acm:us-west-2:************:certificate/********-****-****-****-************",
      "LoadBalancerType": "internet-facing",
      "AllowedCidrIp1": "changeme",
      "CloudWatchLogGroup": "dev-airflow",
      "CloudWatchLogRetentionInDays": 180,
      "AirflowDagS3Bucket": "changeme",
      "KeyName": "changeme",
      "Project": "Airflow Project",
      "Team": "Airflow Team",
      "Environment": "dev",
      "Component": "Airflow"
    }
}
```

Deploy the Cloudformation template:
```bash
./deploy-stack cloudformation/ecs-cluster.cloudformation.yaml \
    dev-airflow-ecs ../ecs-airflow-config/dev-airflow-ecs.json
```

### Airflow Components

The `cloudformation/airflow-ecs-services` folder contains a nested
stack that deploys the following ECS services:
- Airflow Webserver
- Celery Flower Monitoring Tool
- Scheduler
- Multiple Workers

Create a JSON file `dev-airflow-ecs-services.json` to override non-default
parameters.
```json
{
  "Parameters": {
    "EcsStackName": "dev-airflow-ecs",
    "PostgresDbStackName": "dev-airflow-postgres",
    "RedisStackName": "dev-airflow-redis",
    "PostgresUsername": "airflow_user",
    "RedisDb": "0",
    "CloudWatchLogGroup": "dev-airflow",
    "HostedZoneId": "chamgeme",
    "HostedZoneName": "example.com.",
    "DNSPrefix": "dev-airflow",
    "AirflowUserName": "admin",
    "AirflowEmail": "admin@example.com",
    "GoogleOAuthClientId": "changeme",
    "GoogleOAuthDomain": "changeme",
    "AirflowDockerImage": "************.dkr.ecr.us-west-2.amazonaws.com/airflow:0.0.2",
    "MinWebserverTasks": 1,
    "MaxWebserverTasks": 3,
    "DesiredWebserverTasks": 1,
    "MinFlowerTasks": 1,
    "MaxFlowerTasks": 3,
    "DesiredFlowerTasks": 1,
    "MinWorkerTasks": 1,
    "MaxWorkerTasks": 4,
    "DesiredWorkerTasks": 1,
    "SMTPUser": "changeme",
    "SMTPPassword": "changeme",
    "SMTPHost": "changeme",
    "SMTPPort": "change",
    "SMTPStartTLS": "changeme",
    "SMTPSSL": "changeme",
    "Project": "Airflow Project",
    "Team": "Airflow Team",
    "Environment": "dev",
    "Component": "Airflow"
  }
}
```

Configure the passwords and keys as follows:
```bash
kms_key_id=changeme
./kms-encrypt --kms-key-id=${kms_key_id} \
    --param=PostgresPasswordEnc --secret='changeme' \
    --config-path=../ecs-airflow-config/dev-airflow-ecs-services.json

fernet_key=$(docker run puckel/docker-airflow python -c "from cryptography.fernet import Fernet; FERNET_KEY = Fernet.generate_key().decode(); print(FERNET_KEY)")
./kms-encrypt --kms-key-id=${kms_key_id} \
    --param=FernetKeyEnc --secret="${fernet_key}" \
    --config-path=../ecs-airflow-config/dev-airflow-ecs-services.json

./kms-encrypt --kms-key-id=${kms_key_id} \
    --param=AirflowPasswordEnc --secret='changeme' \
    --config-path=../ecs-airflow-config/dev-airflow-ecs-services.json

./kms-encrypt --kms-key-id=${kms_key_id} \
    --param=GoogleOAuthClientSecretEnc --secret='changeme' \
    --config-path=../ecs-airflow-config/dev-airflow-ecs-services.json

./kms-encrypt --kms-key-id=${kms_key_id} \
    --param=SMTPPasswordEnc --secret='changeme' \
    --config-path=../ecs-airflow-config/dev-airflow-ecs-services.json
```

Deploy the Cloudformation template:
```bash
./deploy-nested-stack airflow-ecs-services \
    dev-airflow-ecs-services ../ecs-airflow-config/dev-airflow-ecs-services.json
```

__NOTE__: All changes should be done via the master stack
`dev-airflow-ecs-services`. Do not update or destroy individual stacks
within the nested stack as that will make it difficult to manage and
deploy changes to the master stack running the ECS services.