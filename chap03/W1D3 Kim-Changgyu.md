### 계층으로 구성하기

**코드를 구조화**하는 첫 번째 접근법은 **계층**을 이용하는 것이다.

![IMG1](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F638915cf-860e-4e9b-bc79-e19861f6734b%2FUntitled.png?id=6fa0ad11-b08e-4f0b-bbf3-1d0e0b197381&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=2000&userId=&cache=v2)

웹 - 도메인 - 영속성 계층을 각각의 패키지로 구분하고 앞장에서 설명했듯 인터페이스를 활용해서 영속성 계층이 도메인 계층을 의존하도록 역전시켰다.

그러나 **최소 세 가지 이유로 이 패키지 구조가 최적의 구조는 아니다.**

1. 애플리케이션의 기능 조각이나 특성을 구분 짓는 패키지 경계가 없다.
    - 사용자 관리 기능을 추가해야한다면 연관되지 않은 기능끼리 패키지로 묶이게 된다.
        - domain : User, Account, UserRepo., AccountRepo., UserService, AccountService
        - web : UserController, AccountController
        - persistence : UserRepositoryImpl, AccountRepositoryImpl
2. 애플리케이션이 어떤 유스케이스를 제공하는지 파악할 수 없다
    - AccountService, AccountController 만으로는 어떤 유스케이스를 구현했는지 알기 어렵다.
    - 따라서 특정 기능을 찾기 위해서 어떤 서비스가 구현했는지, 어떤 메서드에 있는지의 공수가 든다.
3. 패키지 구조를 통해서 우리가 목표로 하는 아키텍처를 파악할 수 없다.
    - 계층형 아키텍처이거나 헥사고날 아키텍처이거나 어떤 아키텍처 스타일로든 추측이 가능해진다.
    - 아키텍처를 잘 반영하지 않기 때문에 한눈에 세부적인 요소를 알아보기 어렵다.

---

### 기능으로 구성하기

“계층으로 구성하기”의 몇 가지 문제를 해결해보자.
다음 접근법은 코드(패키지)를 **기능**으로 구성한 것이다.

![IMG2](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F7a2ee66d-a7f4-488d-a0e7-ec78b2dd3305%2FUntitled.png?id=5e0d4c50-405f-4077-bf3f-0e39fcb97b19&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=2000&userId=&cache=v2)

계층 패키지를 없애고 특정 엔티티(도메인)와 관련된 코드를 상위 패키지로 재구성했다.
여기에 account 외부에서 접근이 불가능하도록 각 클래스에서는 package-private 접근 수준을 이용할 수 있다.
(패키지 경계를 강화하면 각 기능 사이의 불필요한 의존성을 방지할 수 있다)

또 책임을 좁히기 위해 AccountService 클래스명을 구체적인 유스케이스를 의미하도록 SendMoneyService로 변경했다. 송금 관련 유스케이스를 구현한 코드는 직관적으로 클래스명을 통해 찾을 수 있게 된다.

**그러나, 기능에 의한 패키징 방식은 사실 계층에 의한 패키징 방식보다 아키텍처의 가시성이 훨씬 떨어진다.**
입/출력 포트나 어댑터 등을 나타내는 패키지명이 없고, 도메인 코드와 영속성 코드 간의 의존성을 역전시킬 때 같은 패키지 내에 있기 때문에 실수로 도메인 코드가 영속성 코드를 의존하는 것을 방지할 수 없다.

---

### 아키텍처적으로 표현력 있는 패키지 구조

![IMG3](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1fbec24f-b5a2-4873-9493-0de1604d9cdd%2FUntitled.png?id=03ae87c2-1be3-4b8a-ad0d-d9a6df4f767c&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=1890&userId=&cache=v2)

최상위에는 해당 패키지가 Account와 관련된 유스케이스를 구현한 모듈임을 나타내는 account가 있다.
그 다음 레벨에는 도메인 모델이 속한 domain 패키지와 도메인 모델을 둘러싸고 서비스 계층을 포함하는 application 패키지가 있다.

SendMoneyService는 입력 포트 인터페이스인 SendMoneyUseCase를 구현하고, 출력 포트 인터페이스이자 영속성 어댑터에 의해 구현된 LoadAccountPort, UpdateAccountStatePort를 사용한다.

apdater 패키지에는 애플리케이션 계층의 입력 포트를 호출하는 어댑터와 애플리케이션 계층의 출력 포트에 대한 구현을 제공하는 어댑터를 포함한다. 만약 Key-Value 기반 데이터베이스로 개발했다가 관계형 데이터베이스로 전환해야한다면 관련 출력 포트 어댑터를 구현하면 된다.

특정 출력 어댑터를 찾을 때 adapter/out/<어댑터 이름> 을 직관적으로 찾을 수 있어 도움이 될 것이다.
이 패키지 구조는 이른바 **‘아키텍처-코드 갭’** 혹은 **‘모델-코드 갭’**을 효과적으로 다룰 수 있는 강력한 요소이다.
즉, 패키지 구조(코드)가 시간이 지남에 따라 목표하던 아키텍처로부터 멀어지지 않도록 아키텍처에 대한 고민을 하게끔 한다는 것이다.

또 DDD 개념에 직접적으로 대응시킬 수 있다는 장점이 있다.
account 같은 상위 패키지는 다른 바운디드 컨텍스트와 통신할 전용 진입점과 출구를 포함하는 바운디드 컨텍스트에 해당한다. domain 패키지 내에서는 DDD가 제공하는 모든 도구를 이용해 우리가 원하는 어떤 도메인 모델이든 만들 수 있다. (?)

패키지 구조를 프로젝트 내내 유지하기 위해 이러한 규칙을 잘 지켜야 하는 반면 프로젝트 규모가 점점 커짐에 따라 현재 패키지 구조가 적합하지 않아서 변경할 수도 있고 의도적으로 아키텍처-코드 갭을 넓히고 아키텍처를 반영하지 않는 패키지 구조를 만들 수도 있기 때문에 완벽한 정답은 없다.
단 표현력 있는 패키지 구조가 코드와 아키텍처의 갭을 줄인다는 것이 핵심이다.

---

### 의존성 주입의 역할

패키지 구조가 클린 아키텍처에 도움이 되긴 하지만 클린 아키텍처의 본질은 애플리케이션 계층이 입/출력 어댑터에 의존성을 갖지 않는 것이다.

예제 코드의 입력 어댑터에서는 그저 애플리케이션 계층의 서비스를 호출할 뿐이다. 그렇다고 입력 어댑터가 서비스를 직접 호출하게 되면 강한 결합이 발생할 수 있다. 따라서 애플리케이션 계층으로의 진입점을 구분 짓기 위해 실제 서비스를 포트 인터페이스 사이로 숨길 수 있다.

![IMG4](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fe0bd0871-676c-4b99-9378-6d238201d10c%2FUntitled.png?id=7ddb8993-2e22-4726-8d6c-8eb1849e1321&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=1990&userId=&cache=v2)

애플리케이션 계층에 인터페이스를 만들고 어댑터에 해당 인터페이스(포트)를 구현한 클래스를 두면 된다.

그런데 인터페이스(포트)를 구현한 실제 객체를 누가 애플리케이션 계층에 제공해야 할까?
인터페이스(포트)를 애플리케이션 계층 안에서 수동으로 초기화하고 싶지 않을 것이다. 그렇게 되면 애플리케이션 계층에서는 어댑터에 강하게 의존하기 때문이다.

이 부분에서 **의존성 주입**을 활용할 수 있다.
모든 계층에 의존성을 갖는 가장 **중립적인 컴포넌트**를 도입하고 이 컴포넌트가 아키텍처를 구성하는 대부분의 클래스를 초기화하는 역할을 수행하게 하면 된다.

![IMG5](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F111c1560-6055-4154-8a7a-bae9a68676d6%2FUntitled.png?id=afbe7b5a-b9bf-4ac8-8d9b-bd890ebbc941&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=1520&userId=&cache=v2)

---

### 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?

이번 장에서 알아본 패키지 구조에서는 아키텍처의 특정 요소를 찾으려면 아키텍처 다이어그램의 박스 이름을 따라 패키지 구조를 탐색하면 된다. 이로써 의사소통, 개발, 유지보수가 조금 수월해질 수 있다.
