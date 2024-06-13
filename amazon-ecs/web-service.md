# Web Service 생성

해당 랩에서는 각 Launch Type에 맞는 Web Task Definition을 기반으로 2개의 ECS Service 를 만들 것 입니다.

* EC2 Launch Type : webdef-on-aws (Task Definition) / svc-on-aws (Service)
* EXTERNAL Launch Type : webdef-on-prem (Task Definition) / svc-on-prem (Service)

## Web Task Definition 생성

Task Definition을 생성 하고 등록합니다.



EC2 Launch Type 용도 Task Definition을 생성합니다.

```
cat << EOF > ~/environment/hybrid-containers/task-definition-webserver-on-aws.json
{
  "executionRoleArn": "arn:aws:iam::$AWS_ACCOUNT_ID:role/ecsanywhereTaskExecutionRole",  
  "family": "Whoami-on-aws",
  "cpu": "256",
  "memory": "128",
  "containerDefinitions": [
    {
      "name": "whoami",
      "image": "traefik/whoami:latest",
      "entryPoint": [],
      "portMappings": [
        {
          "hostPort": 0,
          "protocol": "tcp",
          "containerPort": 80
        }
      ],
      "command": [],
      "logConfiguration": {
              "logDriver": "awslogs",
              "options": {
                  "awslogs-group": "whoami-on-aws",
                  "awslogs-region": "$AWS_REGION",
                  "awslogs-create-group": "true",
                  "awslogs-stream-prefix": "whoami-on-aws"
              }
      }      
    }
  ],
  "placementConstraints": [
    {
      "type": "memberOf",
      "expression": "attribute:role == webserver"
    }
  ],
  "volumes": [],
  "requiresCompatibilities": [
    "EC2"
  ]
}
EOF

```

```
aws ecs register-task-definition --cli-input-json file://~/environment/hybrid-containers/task-definition-webserver-on-aws.json

```

EXTERNAL Launch Type 용도 Task Definition을 생성합니다.

```
cat << EOF > ~/environment/hybrid-containers/task-definition-webserver-on-prem.json
{
  "executionRoleArn": "arn:aws:iam::$AWS_ACCOUNT_ID:role/ecsanywhereTaskExecutionRole",  
  "family": "Whoami-on-prem",
  "cpu": "256",
  "memory": "128",
  "containerDefinitions": [
    {
      "name": "whoami",
      "image": "traefik/whoami:latest",
      "entryPoint": [],
      "portMappings": [
        {
          "hostPort": 0,
          "protocol": "tcp",
          "containerPort": 80
        }
      ],
      "command": [],
      "dockerLabels": {
        "traefik.enable": "true",
        "traefik.http.services.whoami.loadbalancer.server.port": "80",
        "traefik.http.routers.whoami-host.rule": "Host(\`whoami.domain.com\`)",
        "traefik.http.routers.whoami-path.rule": "Path(\`/whoami\`)"
      },
      "logConfiguration": {
              "logDriver": "awslogs",
              "options": {
                  "awslogs-group": "whoami-on-prem",
                  "awslogs-region": "$AWS_REGION",
                  "awslogs-create-group": "true",
                  "awslogs-stream-prefix": "whoami-on-prem"
              }
      }      
    }
  ],
  "placementConstraints": [
    {
      "type": "memberOf",
      "expression": "attribute:role == webserver"
    }
  ],
  "volumes": [],
  "requiresCompatibilities": [
    "EXTERNAL"
  ]
}
EOF

```

```
aws ecs register-task-definition --cli-input-json file://~/environment/hybrid-containers/task-definition-webserver-on-prem.json

```

## ECS/EC2 on AWS 를 위한 ALB 생성

서비스 생성에 앞서 EC2 Launch Type 용 서비스 `svc-on-aws` 를 위해 ALB를 생성합니다.

1. [EC2 Load Balancers ](https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-2#LoadBalancers:sort=loadBalancerName)로 이동합니다.
2. **Create Load Balancer**를 클릭하고 **Application Load Balancer**를 선택합니다.

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

3. **Basic configuration**: Configure Load Balancer
   * Name: `demogo-alb`
   * Internet-facing
   * IPv4

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

4. **Network mapping**: 가용 영역 및 서브넷 선택
   * VPC: ECSWorkshopVPC (10.0.0.0/16)
   * 가용영역 **ap-northeast-2a**와 **ap-northeast-2b**를 모두 체크합니다.
   * Subnet: **PublicSubnet1, 2**를 선택합니다.

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

5. **Security groups**: 검색창에서 **ALB-SG** 를 찾아 선택한 뒤 **default** 보안그룹은 선택 해제합니다.

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

6. **Listeners and routing** : **Create target group** 을 클릭합니다.

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

* Name: `web`
* Target type: Instance (web 작업 정의의 네트워크를 bridge mode로 선택하였기 때문에 Instance 타입으로 선택)
* Port: 80

7. **Next**를 선택하면 **Register Targets** 창이 뜹니다. 지금은 이 단계를 건너뜁니다. **Create target group**을 선택하여 web 대상 그룹 생성을 마칩니다.

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

8. **Listeners and routing**: 다시 **Load balancers** 브라우저 탭으로 돌아와서 **HTTP:80 Listener**로 **web** 을 선택합니다.

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

9. **Summary**: 설정을 검토한 뒤 **Create load balancer** 를 클릭합니다.

<figure><img src="../.gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

## EC2 Launch Type 용 Service 생성

[ECS 클러스터 서비스 콘솔](https://ap-northeast-2.console.aws.amazon.com/ecs/v2/clusters/ECS-Workshop/services?region=ap-northeast-2)로 가서 서비스 생성 버튼을 클릭합니다.\


<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

환경 - 컴퓨팅 옵션은 **시작 유형**,  시작 유형은 **EC2** 를 선택 합니다.

<figure><img src="../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

배포 구성

* 애플리케이션 유형: 서비스
* 패밀리 : Whoami-on-aws
* 서비스 이름 : svc-on-aws
* 서비스 유형 : 복제본
* 원하는 태스크 :  2

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

로드 밸런싱

* 로드 밸런서 유형 : Application Load Balacner
* 컨테이너 : whoami 80:80
* 로드 밸런서 : demogo-alb
* 리스너 : 기존 리스너 사용 > 80:HTTP
* 대상 그룹
  * 새 대상 그룹 생성
  * 대상 그룹 이름 : svc-on-aws-tg
  * 경로 패턴 : /whoami
  * 평가 순서 : 1
  * 상태 확인 프로토콜 : HTTP
  * 상태 확인 경로 : /whoami/

<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

위 설정을 기반으로 서비스를 생성합니다.



## EXTERNAL Launch Type 용 Service 생성

[ECS 클러스터 서비스 콘솔](https://ap-northeast-2.console.aws.amazon.com/ecs/v2/clusters/ECS-Workshop/services?region=ap-northeast-2)로 가서 서비스 생성 버튼을 클릭합니다.

환경 - 컴퓨팅 옵션은 **시작 유형**,  시작 유형은 **EXTERNAL** 를 선택 합니다.

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

배포 구성

* 애플리케이션 유형: 서비스
* 패밀리 : Whoami-on-prem
* 서비스 이름 : svc-on-prem
* 서비스 유형 : 복제본
* 원하는 태스크 :  2

<figure><img src="../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

위 설정을 기반으로 서비스를 생성합니다.



## 트래픽 확인

1. svc-on-aws 서비스 트래픽 확인
   * ALB DNS 이름을 복사하여 확인

<figure><img src="../.gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

```
curl -XGET http://<DNS 이름>/whoami
```

<figure><img src="../.gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

2. svc-on-prem 서비스 트래픽 확인
   * Traefik LB 인스턴스 ip 를 복사하여 확인

<figure><img src="../.gitbook/assets/image (67).png" alt=""><figcaption></figcaption></figure>

```
curl -XGET http://<Traefik Instance's publicIP>/whoami
```

<figure><img src="../.gitbook/assets/image (68).png" alt=""><figcaption></figcaption></figure>
