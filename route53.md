# Route53 구성

해당 랩을 진행하기 위해서는 **반드시 퍼블릭 호스팅 영역을 설정 할 수 있는 개인 소유의 도메인**이 있어야 합니다.

아래와 같이 이미 퍼블릭 호스팅 영역이 만들어져 있는 상황을 가정하고 실습을 진행합니다.

<figure><img src=".gitbook/assets/image (69).png" alt=""><figcaption></figcaption></figure>

## On-Prem Traefik LB 용도 A record 생성

<figure><img src=".gitbook/assets/image (70).png" alt=""><figcaption></figcaption></figure>

앞서 Traefik Service 가 동작하고 있는 Traefik LB 용도의 인스턴스 퍼블릭 IP를 값으로  입력해 레코드를 생성합니다.

<figure><img src=".gitbook/assets/image (71).png" alt=""><figcaption></figcaption></figure>

생성한 A 레코드를 기반으로 서비스까지 트래픽이 도달하는 것을 확인 합니다.

<figure><img src=".gitbook/assets/image (72).png" alt=""><figcaption></figcaption></figure>

## 장애 조치 라우팅 정책 레코드 생성

앞서 생성했던 ALB의 도메인과 on-prem lb 용도 도메인을 Route 53 장애 조치(failover) 라우팅 정책으로 설정해 Active-Standby 형태로 구성합니다.

<figure><img src=".gitbook/assets/image (73).png" alt=""><figcaption></figcaption></figure>

레코드 이름을 설정하고 레코드 유형은 **CNAME** 유형 으로 설정 합니다.

그리고 나서 장애 조치 레코드 정의 버튼을 누릅니다.

<figure><img src=".gitbook/assets/image (74).png" alt=""><figcaption></figcaption></figure>

On-Prem 도메인을 **기본** 장애 조치 레코드 유형으로 설정합니다.

<figure><img src=".gitbook/assets/image (75).png" alt=""><figcaption></figcaption></figure>

레코드 정의 생성 전 상태 검사 콘솔로 이동해 상태 검사 구성을 만듭니다.

<figure><img src=".gitbook/assets/image (76).png" alt=""><figcaption></figcaption></figure>

상태 검사 생성 버튼을 클릭합니다.

<figure><img src=".gitbook/assets/image (79).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
이번 랩에서는 다루지 않지만 상태 검사 실패 시 Process를 위해 SNS 알림을 설정하고 자동화 하는데 활용 할 수 있습니다.
{% endhint %}

상태 확인 ID 새로고침 버튼을 눌러 위 절차에서 생성 한 상태 검사를 입력합니다.

레코드 ID 를 입력합니다.

장애 조치 레코드 정의 버튼을 눌러 레코드 생성을 완료합니다.

<figure><img src=".gitbook/assets/image (80).png" alt=""><figcaption></figcaption></figure>

ALB 도메인을 **보조** 장애 조치 레코드 유형으로 설정합니다.

<figure><img src=".gitbook/assets/image (81).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (82).png" alt=""><figcaption></figcaption></figure>

레코드 정의 생성 전 상태 검사 콘솔로 이동해 상태 검사 구성을 만듭니다.

<figure><img src=".gitbook/assets/image (83).png" alt=""><figcaption></figcaption></figure>

상태 확인 ID 새로고침 버튼을 눌러 위 절차에서 생성 한 상태 검사를 입력합니다.

레코드 ID 를 입력합니다.

장애 조치 레코드 정의 버튼을 눌러 레코드 생성을 완료합니다.

<figure><img src=".gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>

아래와 같이 총 2개의 기본/보조 레코드가 추가 된 것을 확인 후 레코드 생성을 완료 합니다.

<figure><img src=".gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

생성 후 레코드는 아래와 같습니다.

<figure><img src=".gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

nslookup 명령어를 통해 기본 레코드(on-prem) 주소가 정상적으로 출력 되는지 확인 합니다.

<figure><img src=".gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

의도적으로 on-prem 용도 svc의 배포 복제본 개수를 2 에서 0으로 변경시켜 상태 검사가 실패하도록 만듭니다.

ECS 콘솔로 이동해 svc-on-prem 서비스를 업데이트 합니다.

<figure><img src=".gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

Desired tasks 를 2에서 0 으로 변경해 업데이트 합니다.

<figure><img src=".gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure>

Running Task의 개수가 0/0으로 변경 되는 것을 확인합니다.

<figure><img src=".gitbook/assets/image (90).png" alt=""><figcaption></figcaption></figure>

이어서 상태 검사가 비정상으로 변경 되는 것을 확인 할 수 있습니다.

<figure><img src=".gitbook/assets/image (91).png" alt=""><figcaption></figcaption></figure>

수 분이 지난 후 nslookup 명령어를 입력 했을 때 리턴 되는 주소가 ALB 주소로 변경 된 것을 확인 할 수 있습니다.

<figure><img src=".gitbook/assets/image (92).png" alt=""><figcaption></figcaption></figure>
