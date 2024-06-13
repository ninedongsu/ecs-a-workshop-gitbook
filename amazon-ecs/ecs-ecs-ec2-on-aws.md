# ECS 클러스터 생성 및 ECS/EC2 on AWS 구성

## Application Load Balancer용 보안그룹 생성

1. [EC2 ](https://console.aws.amazon.com/ec2) 콘솔솔로 이동합니다.
2.  좌측 탭에서 Security Group 을 선택하고 **Create Security Group** 을 클릭합니다.



    <figure><img src="https://static.us-east-1.prod.workshops.aws/public/ff1c970f-c952-441b-a93c-737f3753d40a/static/images/ecs/cluster/sg/sg.png" alt=""><figcaption></figcaption></figure>
3.  아래와 같이 입력하여 추후 생성할 Application Load Balancer 용 보안그룹을 생성합니다.

    * Security Group Name : `ALB-SG`
    * Description: `ALB-SG`
    * VPC : **ECSWorkshopVPC**
    * Inbound Rule
      * Type : **HTTP**
      * Source : **Anywhere-IPv4 선택** (0.0.0.0/0)
    * Outbound Rule은 수정하지 않습니다.

    <figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>


4. Create Security Group 버튼을 눌러 Application Load Balancer용 보안그룹을 생성을 완료합니다.

## ECS/EC2 on AWS 용 보안그룹 생성

1.  이번엔 AWS VPC에 위치한 ECS EC2용 보안그룹을 생성합니다. 동일하게 [EC2 ](https://console.aws.amazon.com/ec2) 페이지의 좌측 탭에서 Security Group 을 선택하고 **Create Security Group** 을 클릭합니다.\


    <figure><img src="../.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>
2. 아래와 같이 입력하여 ECS Cluster용 보안그룹을 생성합니다. Application Load Balancer에서 오는 트래픽만 ECS Cluster에서 허용하도록 Inbound Rule의 Source를 앞에서 만든 **ALB-SG**으로 선택해야 합니다.
   * Security Group Name : `ECS-ASG-SG`
   * Description: `ECS-ASG-SG`
   * VPC : **ECSWorkshopVPC**
   * Inbound Rule
     * Type : **All TCP**
     * Source : **ALB-SG 검색 후 선택**
   * Outbound Rule은 수정하지 않습니다.

## ECS 클러스터 생성

1.  [Amazon ECS ](https://console.aws.amazon.com/ecs)로 이동합니다. Amazon ECS 콘솔 좌측 네비게이터에서 Clusters로 이동합니다. **Create Cluster**를 클릭합니다.\


    <figure><img src="../.gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>
2.  클러스터 이름에 **ECS-Workshop** 을 입력합니다.\


    <figure><img src="../.gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>
3.  **Infrastructure** 설정에서 아래와 같이 입력합니다.

    * `AWS Fargate` 와 `EC2 Instances` 를 모두 선택합니다.
    * Operationg System/Architecture 는 `Amazon Linux 2` 를 선택합니다.
    * EC2 Instance type 은 `m5.large` 를 선택합니다.
    * Desired Capacity 는 최소 `2`, 최대 `2`로 설정합니다.
    * Root EBS volume size 는 `100 GB` 로 설정합니다.

    \


    <figure><img src="../.gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>
4. **Network settings for Amazon EC2 instances** 에서 아래와 같이 입력합니다.
   * VPC 는 `ECSWorkshopVPC` 를 선택합니다.
   * Subnet 은 `Private Subnet` 2개를 선택합니다.
   *   Security Group 은 `ECS-ASG-SG` 를 선택합니다.\


       <figure><img src="../.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>
5.  **Monitoring** 설정에서는 **Container Insights** 기능을 활성화합니다. 마지막으로 **Create** 버튼을 클릭해서 클러스터를 생성합니다. 클러스터 생성시 약 5분정도가 소요됩니다.\


    <figure><img src="../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>
6.  Amazon ECS 클러스터가 생성됐습니다.\


    <figure><img src="../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>
7. 생성 된 EC2 컨테이너 인스턴스 2대에 사용자 지정 속성을 추가합니다.

<figure><img src="../.gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

사용자 지정 속성 이름을 `role` 로, 값을 `webserver` 로 추가합니다.

<figure><img src="../.gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>
