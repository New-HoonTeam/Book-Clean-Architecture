계층형 아키텍처 - 웹, 도메인, 영속성 계층

![image](https://user-images.githubusercontent.com/86050295/210373973-ea0c3249-6e0c-4936-a6cd-4652b5a1eced.png)

- 웹 계층 : 요청을 받아 비즈니스 계층에 있는 서비스로 요청
- 도메인 계층 : 엔티티 조회 or 수정하며 비즈니스 로직을 수행
- 영속성 계층 : 엔티티의 영속성 관리

계층형 아키텍처 : 견고환 아키텍처 패턴


장점

- 이상적으로(ideally) 도메인 계층은 웹, 영속성 계층에 독립적으로 도메인 로직을 작성할 수 있다.
    
   <t> → 도메인 로직에 영향을 주지 않고, 웹 계층과 영속성 계층에 사용된 기술을 변경할 수 있다.
    
   <t> → 기존 기능에 영향을 주지 않고 새로운 기능을 추가할 수 있다.
    
   <t> ⇒ 변화하는 요구사항과 외부 요인에 빠르게 적응할 수 있게 해준다.
    

단점

- 코드에 나쁜 습관이 들기 쉽다
- 소프트웨어를 점점 더 변경하기 어렵게 만드는 수많은 허점들이 존재

# 데이터베이스 주도 설계를 유도한다.

일반적으로 데이터베이스 구조를 먼저 생각하고 도메인 로직을 구현한다.

  <t> → 비즈니스 관점에서는 전혀 맞지 않는 방법 원래는 도메인 로직을 먼저 만들어야 한다.

  <t> ⇒ 그 방법이 맞지 않음에도 데이터베이스를 먼저 고려하는 이유는 ORM 프레임워크를 사용하기 떄문

ORM 프레임워크를 계층형 아키텍처와 결합하면 
<br> 비즈니스 규칙을 영속성 관점과 섞고 싶은 유혹을 쉽게 받는다.

![image](https://user-images.githubusercontent.com/86050295/210374179-f12c5362-c604-41de-b213-05cdb1b682c6.png)

영속성 계층과 도메인 계층 사이에 강한 결합

도메인(서비스) 계층에서 영속성 모델을 비즈니스 모델처럼 사용하게 되고,
<br> 이로 인해 도메인 로직뿐만 영속성 계층과 관련된 작업들을 (필연적으로) 해야만 한다.

- 즉시로딩 / 지연로딩
- 데이터베이스 트랜잭션
- 캐시 플러시

  <t> ⇒ 영속성 코드가 사실상 도메인 코드에 녹아 하나만 바꾸는 것이 어려워 진다.

# 지름길을 택하기 쉬워진다.

상위 계층에서 할 일을 귀찮아서, 
간단히 하위 계층에서 해결해 버릴려고 하는 나쁜 습관이 들기 쉽다.

![image](https://user-images.githubusercontent.com/86050295/210374255-789043f5-67a0-4a12-ae57-5c4a56a990f6.png)

그래서 시간이 갈 수록 영속성 계층이 비대해 진다.

# 테스트하기 어려워진다.

![image](https://user-images.githubusercontent.com/86050295/210374340-d16c7e03-dc1c-49d8-8d61-6aff95ae3a62.png)

도메인(서비스) 계층을 건너뛰어 웹 계층에서 바로 영속성 계층에 접근하여 
비즈니스 로직을 구현하고 싶은 유혹에 빠지기 쉽다.

  <t> → 책임이 섞이고 핵심 도메인 로직들이 흩어질 확률이 높다.

  <t> → 이렇게 되면 웹 계층 테스트에서 도메인 계층뿐만 아니라 영속성 계층도 모킹해야 한다.

# 유스케이스를 숨긴다.

개발자들은 새로운 코드를 짜는 시간보다 
기존 코드를 바꾸거나 파악하는 데 더 많은 시간을 쓰는게 일반적이다.

아키텍처는 코드를 빠르게 탐색하는 데 도움이 돼야 한다.

도메인 로직이 여러 계층에 걸쳐 흩어지기 쉬워 
새로운 기능을 추가할 적당한 위치를 찾는 일이 어려워진다.


여러 개의 유스케이스를 담당하는 아주 넓은 서비스가 만들어지기도 한다.

넓은 서비스는 영속성 계층에 많은 의존성을 갖게 되고, 
<br> 서비스를 테스트하기도 어려워지고 작업해야 할 유스케이스를 
<br> 책임지는 서비스를 찾기도 어려워진다.

# 동시 작업이 어려워진다.

영속성 계층 위에서 만들어지기 때문에, 영속성 계층을 먼저 개발하고, 
<br> 그 다음에 도메인 계층을, 그리고 마지막으로 웹 계층을 만들어야 하기 떄문에, 
<br> 동시에 한 명의 개발자만 작업 할 수 있다.

데이터베이스 주도 설계를 하지 않는 경우에만 동시작업이 가능하다.
<br> 영속성 로직이 도메인 로직과 너무 뒤섞여서 각 측면을 개별적으로 작업할 수 없기 때문

서로 다른 유스케이스에 대한 작업을 하게 되면, 같은 서비스를 동시에 편집하는 상황이 발생하고, 
<br> 이는 병합 충돌과 코드 롤백 가능성을 높인다.

# 유지 보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
만드는 데 어떻게 도움이 될까?

올바르게 구축하고 몇 가지 추가적인 규칙들을 적용하면 
<br> 유지보수 하기 매우 쉬워지며 코드를 쉽게 변경하거나 추가할 수 있게 된다.

계층형 아키텍처의 함정을 염두에 두면 지름길을 택하지 않고, 
<br> 유지보수하기에 더 쉬운 솔루션을 만드는데 도움이 될 것이다.
