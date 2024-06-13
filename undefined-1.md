# 네트워크 환경 구성

## 스택 배포

아래에서 CloudFormation 템플릿을 다운로드하여 실습에 필요한 리소스를 배포할 수 있습니다.&#x20;

{% file src=".gitbook/assets/ecs-workshop.yaml" %}

1.  [CloudFormation 콘솔](https://ap-northeast-2.console.aws.amazon.com/cloudformation/home)로 이동하여 **스택 생성**을 클릭하고 템플릿을 업로드한 후 **다음**을 클릭합니다.\


    <figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>
2. **ecs-workshop**를 **스택 이름**으로 입력합니다.
3. **스택 옵션 구성**을 건너뛰고 **다음**을 클릭합니다.
4.  검토 및 생성 - **AWS CloudFormation이 IAM 리소스를 생성할 수 있음을 인정합니다**를 선택하고 **제출**을 클릭합니다.\


    <figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>
5.  스택 상태가 **CREATE\_COMPLETE**로 바뀔 때까지 기다립니다. 최대 5분이 소요될 수 있습니다.\


    <figure><img src=".gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

