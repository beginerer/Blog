# 복잡한 쿠폰 조건 if else? => Chain of Responsibility 패턴 적용기

## 기존의 쿠폰 시스템

<img width="790" height="502" alt="스크린샷 2025-07-10 오후 8 55 58" src="https://github.com/user-attachments/assets/8655f658-3a3a-434c-98cb-ff8d8704573e" /> </br>


지금까지 구현한 쿠폰 시스템 입니다. 기존의 쿠폰정책은 가계 사장님이 쿠폰 정책을 등록/수정하는 기본적인 요구사항을 잘 충족시키고 있지만 복잡한 쿠폰 조건을 추가하는 데에는 한계가 존재했습니다. 

이에 좀더 유연하고, 다양한 비즈니스 요구사항을 충족시킬 수 있도록, 복잡한 쿠폰 조건을 적용할 수 있는 쿠폰 시스템을 만들기로 결정했습니다.

하지만 쿠폰 조건이 증가할 수록 조건문이 많아지게 되어 유지보수나 생산성 측면에서 부정적인 영향을 미치게되는 구조적인 문제가 발생합니다. 

이런 구조적 문제 개선을 위한 관련 자료를 찾아보던 중 어느 한 블로그에서 복잡한 조건에 Chain of Responsibility패턴을 적용하는 아이디어를 보고, 이를 쿠폰시스템에 적용하기로 결정했습니다. </br></br>

<img width="688" height="607" alt="스크린샷 2025-07-10 오후 8 56 11" src="https://github.com/user-attachments/assets/75bdf09a-5008-4d94-8848-8edbe65008ea" /> </br>


조건들을 각각 컴포넌트 단위로 보고, 쿠폰 정책 생성시에 조건에 맞는 각 컴포넌트를 조립해서 하나의 chain을 만드는 방식입니다. </br>
chain이 쿠폰 정책에 기반해서 동적으로 생성되는 방식이기 때문에 if-else문 없이 단순히 condition spce만 잘 기술하면 된다는 점이 장점입니다.

또한 filter들의 위치를 원하는 대로 정할 수 있고, 논리적 조건(AND | OR)등 복잡한 조건을 쉽게 적용할 수 있습니다.

### chain of Responsibility 구현

이러한 패턴을 어떻게 적용할지 고민하던 중 chain of Responsibilty 패턴이 spring security의 filterchain과 유사하다는 생각이 들어서, 해당 라이브러리 코드를 참고해서 구현했습니다. 

다만 구조적으로 동일하게 만들지는 못했습니다.
filterchain에서는 filter에 context뿐만아니라 filterchain에 대한 정보를 주입해주어, filter에서 filterchain을 호출하는 전형적인 chain 방식입니다. 

그러나 저의 쿠폰 시스템에서는 향후 캐싱이나 객체 재사용성을 고려해서 stateless 방식으로 설계를 해야했기 때문에 chain 방식은 적용할 수 없었습니다.
따라서 엄밀하게 말하면 이번 쿠폰시스템에 Chain of Responsibiltiy를 적용했다고 보기는 어렵습니다.

## Chain of Responsibility 적용한 쿠폰시스템

<img width="821" height="590" alt="스크린샷 2025-07-10 오후 9 39 54" src="https://github.com/user-attachments/assets/648d2031-c861-4af6-9360-303c40ff7698" /> </br>

쿠폰 조건 유효성 검사요청이 들어오면, db에 저장된 condition spec을 바탕으로 ApplicationCheckerConfig 객체를 생성합니다. </br>
이 객체를 통해 CheckerChainManager(chekerChain의 상위개념, FilterChainProxy처럼 여러 checkerChain의 호출을 제어하는 클래스)와 Context가 동적으로 동시에 생성되는 방식입니다. 

또한, ApplicationCheckerConfig는 immutable하고 stateless이기 때문에 향후 캐싱하기 용이합니다.

번외로, CheckerChainManager객체 생성 비용을 줄이기 위해 object pooling 방식도 고려해봤지만, 오버엔지니어링이거나 혹은 오히려 성능이 느려질 것같다는 판단을 했습니다. </br></br>

먼저 CheckerChainManager에 object pooling을 적용하기 위해서는 다음의 조건을 만족해야 했습니다.

1.CheckerChainManager은 stateful하기 때문에 초기화 메서드가 필요하다. </br></br>
2.객체에 동시접근을 막기 위해 락이나 CAS연산이 요구된다. </br></br>
3.Object pooling처럼 동일한 객체가 pool에 존재하는 것이 아니라, 인터페이스를 구현한 다른 종류의 객체가 존재할 수 있다. 만일 동일한 객체만을 위한 pooling을 하기 위해서는 pool을 동적으로 생성할 수 있어야한다. </br></br>

object pooling방식을 통해 객체 생성비용을 줄이고 싶었지만, 근본적으로 객체가 stateful하기 때문에 조회연산에서 경합이 발생하거나, 리스트를 순회하는 데 비용이 많이 발생하여, 트래픽이 몰리는 경우에는 오히려 성능 병목지점이 된다고 판단했습니다. 

이를 보완하기 위해 현재 가용한 objcet개수를 캐시에 기록해서 1차적으로 필터링하는 방식도 생각해 보았지만, 모든 쿠폰 사용요청을 기록해야 하는 단점이 존재했습니다. 

그리고 유연한 시스템을 위해서는 objcet개수를 조절해야하는 메커니즘이 필요했는데 objcet를 삭제할때 pool에 요청이 들어오면 오류가 발생할 가능성이 존재해서, pool에 별도의 락이나 cas같은 추가적인 동시성 제어가 필요했습니다.

따라서, stateful한 객체의 object pooling은 오버엔지니어링이거나 혹은 오히려 성능 병목의 원인이 된다고 생각해서 보류한 상태입니다.

### 효율적인 Context 생성

chain of responsibilty 패턴 적용에서 했던 여러 고민들 중에 하나는 효율적인 context 생성이였습니다. 
조건별로 필요한 context내용이 다르기 때문에 최소한의 db I/O를 발생시키는 목표였습니다.

따라서, 쿠폰조건을 쿼리단위로 그룹화해서 하나의 도메인을 형성했습니다. 즉, 쿠폰조건별이 아니라 쿠폰 조건 도메인별 DB I/O를 발생시키는 원리입니다. 
만약 단순한 select문으로 해결할 수 없는 통계쿼리나, 복잡한 쿼리(월간 총 주문금액, 월간 주문 횟수)의 경우에는 각 쿠폰조건 자체가 도메인이 되는 방식으로 구현했습니다.

<img width="1181" height="647" alt="스크린샷 2025-07-14 오후 3 58 00" src="https://github.com/user-attachments/assets/009bb874-abac-4f43-b4b2-055844a3d53e" />
<img width="536" height="332" alt="스크린샷 2025-07-14 오후 3 58 08" src="https://github.com/user-attachments/assets/8d569df0-afab-4d08-b852-de80f1573da7" />

### Checker

<img width="1053" height="799" alt="스크린샷 2025-07-14 오후 4 54 03" src="https://github.com/user-attachments/assets/0e1a9aa3-f4c1-4714-b976-4d6aed04c251" />
<img width="836" height="426" alt="스크린샷 2025-07-14 오후 4 54 18" src="https://github.com/user-attachments/assets/c4fbc092-4555-4999-a7bd-9e98527d96a6" />

### CheckerChain

<img width="937" height="849" alt="스크린샷 2025-07-14 오후 3 45 07" src="https://github.com/user-attachments/assets/87b4ef22-7491-4ce1-b371-e7a29dc7ca9f" />

### CheckerChainManager

<img width="1116" height="404" alt="스크린샷 2025-07-14 오후 3 45 31" src="https://github.com/user-attachments/assets/ed0f00b0-5738-41ac-84cf-ba7bae65dbe7" />
<img width="1228" height="903" alt="스크린샷 2025-07-14 오후 7 54 48" src="https://github.com/user-attachments/assets/6e188abe-abaa-417f-99bc-67eb93dd0c51" />



### 작동 테스트

쿠폰 조건 : "카테고리는 CAFE이고, 결제 카드 회사는 "TOSS"이고 신규회원" </br>
          OR "지난달 주문 횟수 3회이상"

<img width="558" height="386" alt="스크린샷 2025-07-14 오후 4 40 31" src="https://github.com/user-attachments/assets/198c76e5-e51c-4ff6-8998-5da99690537a" />
<img width="874" height="826" alt="스크린샷 2025-07-13 오후 4 05 01" src="https://github.com/user-attachments/assets/62b5690d-c29b-4297-9cbb-84d710ee09a1" /> </br></br>

jmeter) 1000 threads 10 seconds

<img width="862" height="591" alt="스크린샷 2025-07-13 오후 4 22 46" src="https://github.com/user-attachments/assets/edb4aac7-93dc-45fc-a207-cc9695855416" /> </br>

아직은 캐시를 적용하지 않은 상태입니다. 추후에 캐시를 적용하고, 필터링순서를 최적화하는 등 성능을 개선해보는 경험을 해볼 예정입니다.

### 마치며

복잡한 아키텍처를 적용한 것은 아니지만 단순한 CRUD에서 벗어나 구조적인 관점에서 시스템을 최적화해보는 재밌는 경험을 해보았습니다.

비록 chain of responsibilty 아이디어를 스스로 떠올리지는 못했지만 이번 경험을 통해, 구조적인 관점에서 시스템을 바라보는 뜻깊은 경험이었습니다.

또한, 실제 비즈니스는 복잡하기 때문에 분산 트랜잭션에 대한 처리가 필수적이라는 사실을 알게 되었습니다. 이에 대해서는 추후에 공부해서 시스템에 적용해 볼 정입니다.

마지막으로, 이제는 단순히 CRUD수준에서 머무르는 것이 아니라, 아키텍처적인 관점에서 시스템을 바라볼 수 있는 개발자가 되기 위해 노력할 것입니다.







