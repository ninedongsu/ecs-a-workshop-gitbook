# Traefik Service 생성

## Task Definition 생성

Cloud9 IDE 터미널에서 Traefik Service를 위한 IAM Policy 및 Role 을 생성합니다.

```
cat << EOF > ~/environment/hybrid-containers/traefik-ecs-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "TraefikECSReadAccess",
            "Effect": "Allow",
            "Action": [
                "ecs:ListClusters",
                "ecs:DescribeClusters",
                "ecs:ListTasks",
                "ecs:DescribeTasks",
                "ecs:DescribeContainerInstances",
                "ecs:DescribeTaskDefinition",
                "ec2:DescribeInstances",
                "ssm:DescribeInstanceInformation"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
EOF

```

```
aws iam create-policy \
--policy-name TraefikECSPolicy \
--policy-document file://~/environment/hybrid-containers/traefik-ecs-policy.json

aws iam create-role \
--role-name ecsAnywhereTraefikRole \
--assume-role-policy-document file://~/environment/hybrid-containers/ecs-trust-policy.json

aws iam attach-role-policy \
--role-name ecsAnywhereTraefikRole \
--policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/TraefikECSPolicy

```

```
aws iam --region $AWS_REGION create-role --role-name ecsanywhereTaskExecutionRole \
    --assume-role-policy-document file://~/environment/hybrid-containers/orders-processor/task-execution-assume-role.json

aws iam --region $AWS_REGION attach-role-policy --role-name ecsanywhereTaskExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws iam --region $AWS_REGION attach-role-policy --role-name ecsanywhereTaskExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess

```



Treafik service 를 위한 Task Definition을 생성 하고 등록합니다.

```
cat << EOF > ~/environment/hybrid-containers/task-definition-TraefikLoadBalancer.json
{
    "family": "TraefikLoadBalancer",
    "cpu": "256",
    "memory": "128",
    "containerDefinitions": [
    {
        "name": "traefik",
        "image": "traefik:latest",
        "entryPoint": [],
        "portMappings": [
            {
                "hostPort": 80,
                "protocol": "tcp",
                "containerPort": 80
            },
            {
                "hostPort": 8080,
                "protocol": "tcp",
                "containerPort": 8080
            }
        ],
        "command": [
            "--api.dashboard=true",
            "--api.insecure=true",
            "--accesslog=true",
            "--providers.ecs.ecsAnywhere=true",
            "--providers.ecs.region=$AWS_REGION",
            "--providers.ecs.autoDiscoverClusters=false",
            "--providers.ecs.clusters=$CLUSTER_NAME",
            "--providers.ecs.exposedByDefault=false"
        ]
    }
    ],
    "placementConstraints": [
        {
            "type": "memberOf",
            "expression": "attribute:role == loadbalancer"
        }
    ],
    "taskRoleArn": "arn:aws:iam::$AWS_ACCOUNT_ID:role/ecsAnywhereTraefikRole",
    "executionRoleArn": "arn:aws:iam::$AWS_ACCOUNT_ID:role/ecsanywhereTaskExecutionRole",  
    "requiresCompatibilities": [
        "EC2", "EXTERNAL"
    ]
}
EOF

```

```
aws ecs register-task-definition --cli-input-json file://~/environment/hybrid-containers/task-definition-TraefikLoadBalancer.json

```



## Service 생성

위 절차에서 만든 Task Definition 기반으로 Traefik LB Service를 생성합니다.

```
aws ecs create-service \
    --cluster $CLUSTER_NAME \
    --service-name TraefikLoadBalancer \
    --task-definition TraefikLoadBalancer:1 \
    --desired-count 1 \
    --launch-type EXTERNAL

```

생성 된 Traefik Dashboard로 접근해 서비스 배포가 잘 되었는지 확인합니다.

```
instanceip=`aws cloudformation describe-stacks --stack-name on-prem --query "Stacks[0].Outputs[?OutputKey=='InstanceIP'].OutputValue" --output text` 
echo http://$instanceip:8080/dashboard

```

<figure><img src="../.gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>
