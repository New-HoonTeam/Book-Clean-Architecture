여러 사례를 통해 계층으로 구성된 웹 애플리케이션을 접하거나 개발해봤을 것이다.

![IMG1](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F21dd0873-ed53-403c-86b2-55a6d980ff95%2FUntitled.png?id=dd794b12-49c0-49a8-bab1-e70be85016f8&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=960&userId=&cache=v2)

3-Tier Architecture

**웹** 계층에서 요청을 받아 **도메인**(또는 비즈니스 계층의 서비스)로 요청을 보낸다. 서비스에서 필요한 비즈니스 로직을 수행하고 도메인 엔티티의 현재 상태를 조회하거나 변경하기 위해 **영속성** 계층의 컴포넌트를 호출한다.

계층형 아키텍처는 견고한 아키텍처 패턴이고 적절하게 활용한다면 **1)기존 기능에 영향을 주지 않고 새로운 기능을 추가**할 수 있고 **2)웹, 영속성 계층에 독립적인 도메인 로직을 작성**할 수 있으며 반대로 **3)도메인 로직에 영향을 주지 않고 웹, 영속성 계층에 사용된 기술을 변경**할 수 있다.

→ 변화하는 요구사항과 외부 요인에 빠르게 적응할 수 있도록 한다.

문제는 계층형 아키텍처를 잘 못 활용하기 쉽다는 것이다. 잘 못 활용할 때 발생하는 문제들은 다음과 같다.

### 계층형 아키텍처는 데이터베이스 주도 설계를 유도한다.

우리가 만드는 대부분의 애플리케이션의 목적은 비즈니스를 관장하는 규칙(정책)을 반영한 모델을 만들어서 사용자가 편리하게 활용할 수 있게 한다. 즉 **상태가 아닌 행동 중심으로 모델링**한다.
그렇다면 왜 우리는 도메인 로직(행동)이 아닌 데이터베이스를 토대로 아키텍처를 설계할까?
(애플리케이션을 구현할 때 도메인 로직보다 영속성 계층, 데이터베이스의 구조를 먼저 설계했을 것이다)

계층형 아키텍처의 의존성 방향에 따라 자연스럽게 구현한 것이기 때문에 합리적이지만 비즈니스 관점에서는 적절하지 않다고 본다. **도메인 로직을 제대로 이해하고 있는지 먼저 작성하고 이를 기반으로 영속성 계층과 웹 계층을 만들어야 한다.**

또 대체적으로 ORM 프레임워크를 활용하기 때문에 비즈니스 규칙을 영속성 관점과 잘 구분하지 못 하게 된다.

![IMG2](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fd8c90872-fe28-4154-a295-274e5fff07df%2FUntitled.png?id=8cee2900-f8b3-4e87-a9f1-54c9d46edab1&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=2000&userId=&cache=v2)

그림과 같은 예시가 일반적으로 설계하는 방법이다.
이렇게 되면 서비스는 도메인 로직뿐만 아니라 즉시/지연로딩, 트랜잭션, 캐시 플러시 등 영속성 계층과 관련된 작업들을 해야만 해서 **도메인, 영속성 계층 간의 강한 결합이 발생하고 유연성을 떨어트린다**.

### 지름길을 택하기 쉬워진다.

계층형 아키텍처의 공통 규칙은 계층 내 혹은 하위 계층의 컴포넌트에만 접근이 가능하다는 것이다.
필요에 따라 상위 계층에 위치한 컴포넌트에 접근해야 할 경우 컴포넌트를 하위 계층으로 끌어내리면 된다.

그런데 매번 이 ‘지름길’을 택하게 되면 결국 아래와 같이 하위 계층은 비대해질 수 밖에 없다.

![IMG3](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F11ebd520-9a80-4121-a8af-0a88624b63a7%2FUntitled.png?id=4190ec64-a120-4e84-95b8-3fdd01e5fd91&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=2000&userId=&cache=v2)

이를 방지하기 위해서는 적절한 아키텍처 규칙을 적용하거나 계층형 아키텍처를 선택하지 않아야 한다.
(상위 계층의 컴포넌트를 쉽게 하위 계층으로 끌어내렸을 때 빌드가 실패한다거나…)

### 테스트하기 어려워진다.

계층형 아키텍처에서 정말 간단한 경우(예를 들어 엔티티의 필드를 단 하나만 살짝 수정하는…)에는 웹 계층에서 영속성 계층에 접근하도록 해서 도메인 계층을 건드리지 않도록 할 수 있지 않을까 라는 생각을 할 수 있다.

![IMG4](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F32cb29df-f741-45a7-9cd9-e2392bc65cb8%2FUntitled.png?id=dbde0024-92f9-478f-9ceb-985d6f4a6587&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=2000&userId=&cache=v2)

이는 두 가지의 문제점을 야기한다.

첫 번째는 **도메인 로직을 웹 계층에 구현**하게 된다는 것이다. 도메인 로직이 한 군데(도메인 계층)에서 관리되지 않고 애플리케이션 전반에 걸쳐 책임이 섞이고 핵심 로직이 여기저기 흩어지게 된다.

두 번째는 도메인 계층뿐 아니라 영속성 계층도 모킹해야 한다는 것이다. 단위 테스트의 복잡도가 올라간다.

### 유스케이스를 숨긴다.

계층형 아키텍처가 특정 유스케이스에서 수정이나 기능 추가시 적절한 코드를 빠르게 탐색할 수 있도록 할까?

![IMG5](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F453b7867-0252-4440-a0a7-cd36bad1c6e1%2FUntitled.png?id=4ea74976-a81c-435e-b152-0684477220f4&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=2000&userId=&cache=v2)

단일로 여러 유스케이스를 담당하는 서비스(UserService 같은)는 시간이 지나면 아주 비대해진다.
거대한 **서비스로 인해 테스트도 어려워지고 특정 유스케이스의 코드를 찾기도 어려워진다.**

따라서 UserService의 사용자 등록 유스케이스를 RegisterUserService 등으로 특화된 서비스가 특정 유스케이스를 담당하게 한다면 어떨지 고민해보자.

### 동시 작업이 어려워진다.

소프트웨어의 개발에는 주어진 날짜와 예산이 존재하고 지연되지 않기 위해 인력을 더 투입할 수 있다.
프로젝트에 인원이 투입될 경우 더 빠르게 진행된다고 기대할 수 있지만 아키텍처가 동시 작업을 지원해야 인력이 늘어난만큼 효과를 볼 수 있을 것이다. 그런 관점에서 계층형 아키텍처는 별 도움이 되지 않는다.

계층형 아키텍처로 개발할 때 일반적으로 하위 계층에 의존하기 때문에 계층별로 인원을 분배해서 작업하기란 쉽지 않다. 그렇다면 인터페이스를 미리 정의하고 인터페이스 기반으로 작업하면 되지 않을까? 물론 데이터베이스 주도 설계를 하지 않는다면 가능하다. 데이터베이스 주도 설계는 영속성 로직과 도메인 로직이 섞이거나 의존적이기 때문에 동시 작업이 매우 어렵다는 것이다.

또 여러 유스케이스를 담당하는 거대한 서비스가 있고 같은 서비스를 동시 편집한다면 이로 인한 병합 충돌과 특정 이슈로 인해 이전 코드로 되돌려야 하는 잠재적 문제들이 발생할 수 있다.

### 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?

올바르게 아키텍처를 구축하고 위에서 다룬 단점을 보완하기 위해 몇 가지 추가 규칙을 적용하면 계층형 아키텍처는 여전히 유지보수하기 쉽고 유연성을 갖는 아키텍처라고 할 수 있다.