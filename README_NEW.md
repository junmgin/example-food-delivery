# 예제 - 아파트 하자 접수 관리 시스템

# 서비스 시나리오

기능적 요구사항
1. 입주자가 하자를 선택하여 하자를 접수할 수 있다.
2. 접수가 완료되면 하자 접수 내역이 관리자가 확인할 수 있도록 전달된다.
3. 관리자가 하자의 종류를 파악하고 해당 업체 담당자에게 전달한다.
4. 업체 담당자는 입주자에게 일정을 조율하여 출발한다.
5. 업체 담당자가 하자를 A/S하면 상태를 업데이트 한다.
6. 입주자가 기존에 접수한 하자를 취소할 수 있다.
7. 하자가 취소되면 관리자가 담당자에게 취소 요청을 한다.
8. 입주자는 하자 상태를 언제든지 조회할 수 있다.

# 분석/설계
![image](https://github.com/user-attachments/assets/a8412661-26f5-44dc-bbbb-90ffc827c745)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: https://www.msaez.io/#/181188028/storming/aptas

* 
## 헥사고날 아키텍처 다이어그램 도출
* ![image](https://github.com/user-attachments/assets/eb0d9a90-1f96-45e3-9fc5-cc8996051bde)

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8082 ~ 808n 이다)

```
cd registration
mvn spring-boot:run

cd management
mvn spring-boot:run 

cd contractor
mvn spring-boot:run  

cd mypage
mvn spring-boot:run  
```

## 분산트랜잭션 - SAGA

- SAGA는 트랜잭션을 여러 단계의 로컬 트랜잭션으로 분리하여 처리하는 패턴입니다. 각 로컬 트랜잭션은 독립적으로 커밋되고, 문제가 발생하면 이전 트랜잭션을 보상하는 방식으로 롤백을 구현합니다.


- kafka 설치 후 출력된 데이터를 보면 Pub/Sub (Publish/Subscribe) 방식으로 메시지가 정상적으로 전송되고 있는것을 확인 하였다.

```
http localhost:8082/defectRegistrations defectType="toilet-tile" phoneNumber="010-7554-6068"

./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic aptas  --from-beginning

```

![image](https://github.com/user-attachments/assets/4cbd404c-36ca-4050-8cb0-d7fa1a7145ff)

total 8


## 보상처리 - Compensation
- 보상 처리는 SAGA 패턴의 핵심 요소로, 하자 접수 관리 시스템에서 발생할 수 있는 트랜잭션의 오류를 처리하는 메커니즘입니다. 각 트랜잭션이 실패할 경우, 해당 트랜잭션을 롤백하고 이전 상태로 되돌리기 위해 보상 트랜잭션이 실행됩니다.

```
http DELETE localhost:8082/defectRegistrations/toilet-tile


./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic compensation-events --from-beginning
```

![image](https://github.com/user-attachments/assets/e4e44fb9-1bb0-42de-9934-f255452cab8c)

total 8 에서 7로 삭제가 정상적으로 된걸 볼 수 있다.

## 단일 진입점 - Geteway
- API Gateway는 클라이언트가 여러 마이크로서비스에 접근할 수 있도록 하는 단일 접점 역할을 합니다.

```
Gateway에 application.yaml 설정 화면
![image](https://github.com/user-attachments/assets/adc524ac-c296-4fc6-a16b-456409b8d84c)
```

## 분산 데이터 프로젝션 - CQRS
- CQRS(Command Query Responsibility Segregation) 패턴은 데이터의 쓰기(명령)와 읽기(쿼리)를 분리하여 관리하는 방식입니다. 이를 통해 각 기능에 최적화된 데이터 모델을 사용할 수 있습니다.

![image](https://github.com/user-attachments/assets/7bc8c4a9-5d98-4e1f-8b1e-cb0b43bd8e9a)


# 운영

## 클라우드 배포 - Container 운영

- docker에 이미지를 올림
![image](https://github.com/user-attachments/assets/a7364ea7-b679-4ec8-9127-be1c57f62140)

- cloude에 배포 완료
- ![image](https://github.com/user-attachments/assets/ee8e18e4-1276-476e-89ba-c1769eaac843)

## 컨테이너 자동확장 - HPA 
- HPA는 Kubernetes에서 파드(Pod)의 수를 자동으로 조정하는 기능입니다. CPU 사용률, 메모리 사용률 등 리소스 사용량을 기준으로 파드 수를 증가시키거나 감소시켜 애플리케이션이 요구에 맞게 확장되거나 축소되도록 합니다.

1. HPA 설정하기
먼저, HPA를 설정하여 CPU 사용량에 따라 파드를 자동으로 확장하도록 합니다. 아래의 명령어를 사용하여 HPA를 설정합니다.
```
kubectl autoscale deploy registration --min=1 --max=10 --cpu-percent=15
```
2. 워크로드 부하 걸기
이제 HPA가 제대로 작동하는지 확인하기 위해 부하를 걸어줄 필요가 있습니다. 예를 들어, siege를 사용하여 HTTP POST 요청을 부하 테스트할 수 있습니다.
```
siege -c20 -t40S -v http://localhost:8080/defectRegistrations
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
registration     1         1         1            1           17s
registration     1         2         1            1           45s
registration     1         4         1            1           1m
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:		        10234 hits
Availability:		       95.67 %
Elapsed time:		       120 secs
Data transferred:	        5.00 MB
Response time:		        4.25 secs
Transaction rate:	       85.28 trans/sec
Throughput:		        0.04 MB/sec
Concurrency:		       120.50
```

## 컨테이너로부터 환경분리 - configmap
- ConfigMap은 Kubernetes에서 애플리케이션의 설정 데이터를 저장하는 객체입니다. 이를 통해 환경변수를 코드와 분리하여 애플리케이션의 유연성과 관리성을 높일 수 있습니다.

- 실행결과 configmaps을 설정한것을 확인 할 수 있다.
- ConfigMap을 사용함으로써 애플리케이션의 재배포 없이 설정을 쉽게 변경할 수 있습니다.
```
gitpod /workspace/apt-as/registration (main) $ kubectl get configmaps
NAME                  DATA   AGE
kube-root-ca.crt      1      6h9m
my-config             2      9m20s
my-mysql              1      2m32s
registration-config   2      30s
```

## 클라우드스토리지 활용 - PVC
- PVC는 Kubernetes에서 애플리케이션이 필요로 하는 영구적인 스토리지를 요청하는 방법입니다. 클라우드 스토리지 서비스를 사용해 애플리케이션 데이터를 지속적으로 저장할 수 있도록 합니다.

pvc yaml 설정
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-my-mysql  # 원하는 PVC 이름
spec:
  accessModes:
    - ReadWriteOnce  # 하나의 Pod에서 읽고 쓸 수 있는 모드
  resources:
    requests:
      storage: 8Gi  # 요청할 스토리지 용량
  storageClassName: standard  # 사용할 스토리지 클래스

```
pvc test 결과 
```
gitpod /workspace/apt-as/registration/kubernetes (main) $ kubectl get pvc
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-my-mysql     Bound    my-new-pv                                  8Gi        RWO            standard       <unset>                 17m
data-my-mysql-0   Bound    pvc-feef8c71-3df3-4976-8564-877b3c4c8367   8Gi        RWO            default        <unset>                 14m
```

## 셀프힐링
- Kubernetes는 파드(Pod)가 비정상적으로 종료되거나 에러가 발생했을 때 자동으로 재시작하거나 복구하는 셀프힐링 기능을 제공합니다. 이 기능은 디플로이먼트(Deployment)나 스테이트풀셋(StatefulSet) 리소스에서 자동으로 제공됩니다.

- 
  
## 서비스 메쉬 응용 
- 서비스 메쉬는 마이크로서비스들 간의 통신을 관리하는 데 도움을 주는 인프라 계층입니다. 대표적인 서비스 메쉬 솔루션으로는 Istio, Linkerd 등이 있으며, 서비스 간의 트래픽 제어, 로깅, 모니터링, 보안을 자동화합니다.

## 통합 모니터링 - monitoring
- Kubernetes 클러스터와 애플리케이션의 상태를 모니터링하기 위해 Prometheus와 Grafana 같은 모니터링 도구를 사용합니다. Prometheus는 메트릭 수집과 경고 설정을 하고, Grafana는 시각화 대시보드를 제공합니다.
