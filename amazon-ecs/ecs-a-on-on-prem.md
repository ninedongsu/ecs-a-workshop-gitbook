# ECS-A on On-Prem 구성

## 역할 생성

Cloud9 IDE로 이동해 ECS-A 환경 구성에 필요한 파일을 다운로드 받습니다.

```sh
curl 'https://static.us-east-1.prod.workshops.aws/public/01653461-ee6f-46df-be1a-e72ffd64b87f/assets/hybrid-containers.zip' --output hybrid-containers.zip
unzip hybrid-containers.zip

```

<figure><img src="../.gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

ECS Anywhere IAM Role 을 생성합니다.

```sh
cd ~/environment/hybrid-containers

aws iam create-role \
    --role-name ecsAnywhereRole \
    --assume-role-policy-document file://ssm-trust-policy.json

aws iam attach-role-policy \
    --role-name ecsAnywhereRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam attach-role-policy \
    --role-name ecsAnywhereRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

```

생성 된 IAM Role 을 확인합니다.

```
aws iam list-attached-role-policies --role-name ecsAnywhereRole

```

<figure><img src="../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

ECS Anywhere 인스턴스 등록을 위한 SSM Activation key를 생성 합니다.

```
cd ~/environment/hybrid-containers
aws ssm create-activation --iam-role ecsAnywhereRole --registration-limit 1 | tee ssm-activation.json

cat ssm-activation.json

```

<figure><img src="../.gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

## 온프레미스 Traefik LB 용도 인스턴스 생성

가상의 온프레미스 인스턴스들이 배포 될 Default VPC 및 서브넷을 환경 변수로 등록 합니다.

```
subnetID=$(aws ec2 describe-subnets --filters "Name=default-for-az,Values=true" "Name=availability-zone,Values=${AWS_REGION}a" --query 'Subnets[].SubnetId'| jq -r .[0])
vpcID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query Vpcs[].VpcId --output text)
echo $subnetID && echo $vpcID

```

다음 명령어를 통해 CloudFormation stack 을 생성합니다.

```
cd ~/environment/hybrid-containers
curl -o on-prem-instance.yml "https://static.us-east-1.prod.workshops.aws/public/01653461-ee6f-46df-be1a-e72ffd64b87f/assets/on-prem-instance.yml"
aws cloudformation create-stack --stack-name "on-prem" \
--template-body file://on-prem-instance.yml \
--parameters "ParameterKey=vpcIDParam,ParameterValue=$vpcID" "ParameterKey=subnetIDParam,ParameterValue=$subnetID"
aws cloudformation wait stack-create-complete --stack-name "on-prem"

```

Traefik LB 용도 인스턴스에 접속하기 위해 ssh private key를 생성합니다.

```
#Retrieve Secret Name
keyid=`aws cloudformation describe-stacks --stack-name on-prem --query "Stacks[0].Outputs[?OutputKey=='KeyID'].OutputValue" --output text`

#Retrieve and save ssh key
aws ssm get-parameter --name /ec2/keypair/$keyid --with-decryption | jq -r .Parameter.Value > ~/.ssh/private_key
chmod 400 ~/.ssh/private_key

#Deactivate IMDS access, so that it's like the VM is not in AWS
instanceid=`aws cloudformation describe-stacks --stack-name on-prem --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" --output text`
aws ec2 modify-instance-metadata-options \
--instance-id $instanceid \
--http-tokens required \
--http-endpoint disabled

```

생성 된 Trafik LB 용도 인스턴스(on-prem) 의 보안 그룹에 cloud9의 public ip를 소스로 하여 ssh 포트 접근에 대한 인바운드 규칙을 생성합니다.

<figure><img src="../.gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>

다음 화면과 같이 Cloud9 IDE에 **새로운 터미널을 띄워** Traefik LB 용도 인스턴스로 접속합니다.

<figure><img src="../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>

```
instanceip=`aws cloudformation describe-stacks --stack-name on-prem --query "Stacks[0].Outputs[?OutputKey=='InstanceIP'].OutputValue" --output text`
ssh ubuntu@$instanceip -i ~/.ssh/private_key

```

다음 화면과 같이 **기존 Cloud9 IDE 터미널**에서 ECS Anywhere Registration을 위한 환경 변수를 등록합니다.

<figure><img src="../.gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

```
export CLUSTER_NAME=ECS-Workshop
echo export CLUSTER_NAME=$CLUSTER_NAME && \
echo export ACTIVATION_ID=$(cat ~/environment/hybrid-containers/ssm-activation.json  | jq ".ActivationId" -r ) && \
echo export ACTIVATION_CODE=$(cat ~/environment/hybrid-containers/ssm-activation.json  | jq ".ActivationCode" -r)  && \
echo export AWS_REGION=$AWS_REGION

```



위 명령어로 출력 된 export로 시작하는 명령어들을 Traefik LB 용도 인스턴스에 붙여넣습니다.

<figure><img src="../.gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

ecs-anywhere-install 스크립트를 통해 Traefik LB 용도 인스턴스를 ECS Cluster에 등록합니다.

```
curl -o "ecs-anywhere-install.sh" "https://amazon-ecs-agent.s3.amazonaws.com/ecs-anywhere-install-latest.sh" \
    && sudo chmod +x ecs-anywhere-install.sh

# Configure ECS on external instance
sudo mkdir -p /etc/ecs
sudo bash -c 'cat << EOF > /etc/ecs/ecs.config
ECS_INSTANCE_ATTRIBUTES={"role": "loadbalancer"}
EOF'

#Install ECS Anywhere
sudo ./ecs-anywhere-install.sh \
    --cluster $CLUSTER_NAME \
    --activation-id $ACTIVATION_ID \
    --activation-code $ACTIVATION_CODE \
    --region $AWS_REGION \
    --docker-install-source distro

```

등록이 잘 되었다면, 아래 화면과 같이 `# ok` 출력이 확인 될 것입니다.

<figure><img src="../.gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>

기존 Cloud9 IDE 터미널로 돌아가 Traefik LB 용도 인스턴스가 ECS 에서 잘 보이는지 확인합니다.

```
aws ecs list-attributes \
  --target-type container-instance \
  --attribute-name role \
  --attribute-value loadbalancer \
  --cluster $CLUSTER_NAME

```

<figure><img src="../.gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

## 서비스 용도 인스턴스 생성

ECS Anywhere 인스턴스 등록을 위한 SSM Activation key를 생성 합니다.

```
cd ~/environment/hybrid-containers
aws ssm create-activation --iam-role ecsAnywhereRole --registration-limit 2 | tee ssm-activation-2.json

cat ssm-activation-2.json

```

CloudFormation을 통해 2대의 가상의 ECS-A 온프레미스 인스턴스를 생성합니다.

```
cd ~/environment/hybrid-containers
for x in 1 2; do
  aws cloudformation create-stack --stack-name "on-prem-$x" \
  --template-body file://on-prem-instance.yml \
  --parameters "ParameterKey=vpcIDParam,ParameterValue=$vpcID" "ParameterKey=subnetIDParam,ParameterValue=$subnetID"
done

for x in 1 2; do
  aws cloudformation wait stack-create-complete --stack-name "on-prem-$x"
done

```



&#x20;Traefik LB 인스턴스의 보안그룹에 Cloud9 SSH 접속 규칙을 추가했던 것 처럼 새로 생성 된 2개의 인스턴스의 보안그룹에 인바운드 규칙을 추가합니다.

<figure><img src="../.gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (56).png" alt=""><figcaption></figcaption></figure>

Cloud9 IDE로 돌아가 ssh 연결을 위한 설정을 진행합니다.

```
for x in 1 2; do
  key="keyid$x"
  export keyid$x=$(aws cloudformation describe-stacks --stack-name on-prem-$x --query "Stacks[0].Outputs[?OutputKey=='KeyID'].OutputValue" --output text)
  aws ssm get-parameter --name /ec2/keypair/${!key} --with-decryption | jq -r .Parameter.Value > private_key-$x
  sudo mv private_key-$x ~/.ssh/private_key-$x
  sudo chmod 400 ~/.ssh/private_key-$x

  #Deactivate IMDS access, so that it's like the VM is not in AWS
  instanceid=`aws cloudformation describe-stacks --stack-name on-prem-$x --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" --output text`
  aws ec2 modify-instance-metadata-options \
    --instance-id $instanceid \
    --http-tokens required \
    --http-endpoint disabled
done

```

```
cat << EOF > ~/.ssh/config
Host vm
  Hostname $(aws cloudformation describe-stacks --stack-name on-prem --query "Stacks[0].Outputs[?OutputKey=='InstanceIP'].OutputValue" --output text)
  User ubuntu  
  IdentityFile ~/.ssh/private_key
  
Host vm1
  Hostname $(aws cloudformation describe-stacks --stack-name on-prem-1 --query "Stacks[0].Outputs[?OutputKey=='InstanceIP'].OutputValue" --output text)
  User ubuntu  
  IdentityFile ~/.ssh/private_key-1

Host vm2
  Hostname $(aws cloudformation describe-stacks --stack-name on-prem-2 --query "Stacks[0].Outputs[?OutputKey=='InstanceIP'].OutputValue" --output text)
  User ubuntu  
  IdentityFile ~/.ssh/private_key-2
EOF
chmod 600 ~/.ssh/config 

```

이제 아래와 같이 alias 를 통해 바로 인스턴스에 ssh 접속 할 수 있습니다.

```
ssh vm # the VM already created and used for the Traefik loadbalancer
ssh vm1 # the VM for webserver application
ssh vm2 # the VM for webserver application
```

새로 생성 된 2대의 인스턴스를 ECS에 등록합니다.

<figure><img src="../.gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

```
echo export CLUSTER_NAME=$CLUSTER_NAME && \
echo export ACTIVATION_ID=$(cat ~/environment/hybrid-containers/ssm-activation-2.json  | jq ".ActivationId" -r ) && \
echo export ACTIVATION_CODE=$(cat ~/environment/hybrid-containers/ssm-activation-2.json  | jq ".ActivationCode" -r)  && \
echo export AWS_REGION=$AWS_REGION

```

```
curl -o "ecs-anywhere-install.sh" "https://amazon-ecs-agent.s3.amazonaws.com/ecs-anywhere-install-latest.sh" \
    && sudo chmod +x ecs-anywhere-install.sh

sudo mkdir -p /etc/ecs
sudo bash -c 'cat << EOF > /etc/ecs/ecs.config
ECS_INSTANCE_ATTRIBUTES={"role": "webserver"}
EOF'

sudo ./ecs-anywhere-install.sh \
    --cluster $CLUSTER_NAME \
    --activation-id $ACTIVATION_ID \
    --activation-code $ACTIVATION_CODE \
    --region $AWS_REGION \
    --docker-install-source distro


```

**on-prem2 도 동일하게 진행** (vm2 ssh 접속, 환경 변수 등록 & 스크립트 실행) 합니다.



기존 Cloud9 IDE 터미널로 돌아가 새로 등록 된 2대의 서비스 용도 인스턴스가 ECS 에서 잘 보이는지 확인합니다.

```
aws ecs list-attributes \
  --target-type container-instance \
  --attribute-name role \
  --attribute-value webserver \
  --cluster $CLUSTER_NAME

```

<figure><img src="../.gitbook/assets/image (58).png" alt=""><figcaption></figcaption></figure>

