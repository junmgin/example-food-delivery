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

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd registration
mvn spring-boot:run

cd management
mvn spring-boot:run 

cd contractor
mvn spring-boot:run  

cd mypage
python policy-handler.py 
```
## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```

