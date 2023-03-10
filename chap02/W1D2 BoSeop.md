# 2장 의존성 역전하기

# 단일 책임 원직

기존의 SRP의 일반적인 해석은 다음과 같다

![Untitled](https://user-images.githubusercontent.com/39071638/212551949-26c67f8d-cecb-4b72-a8b5-bc84131b3e82.png)
하지만 여기서는 이렇게 주장한다

![Untitled 1](https://user-images.githubusercontent.com/39071638/212551965-8e15622c-ece7-4aaf-9210-06b976984c83.png)
만약 컴포넌트에서 변경될 이유가 오로지 한 가지라면, 컴포넌트는 딱 한가지 일만 하게 된다.

더 중요한 것은 변경할 이유가 오직 한 가지라는 그 자체다.

그렇다면 이것의 아키텍쳐에서의 의미는 무엇일까??

→ 컴포넌트를 변경할 이유가 한 가지라면, 다른 이유로 sw를 변경한다면, 신경 쓸 필요가 없다.

하지만…. 변경할 이유 라는 것은 컴포넌트들 간의 의존성을 통해 너무나도 쉽게 전파된다…

![Untitled 2](https://user-images.githubusercontent.com/39071638/212551971-29ce2b78-2452-4f7b-a78f-623af41c47c5.png)
A의 경우 여러 다른 컴포넌트들에 의존한다

반면 E는 하나도 의존하지 않는다

E 가 변경된다는 것은 E의 기능을 변경해야할 때 뿐이다. 하지만 A는 의존하고 있는 다른 컴포넌트들의 기능이 변경되더라도 본인 또한 변경이 되어야 한다. 

이렇게 의존성이 증가하면 SRP를 위반하기가 더욱 쉬워지고 유지보수하기는 더더욱 어려워진다. 

# 의존성 역전 원칙

계층형 아키텍쳐에서는 상위 계층이 하위 계층에 의존한다. 이 말인 즉슨 상위 계층은 하위 계층보다 훨씬 더 자주 변경될 가능성이 높다는 점이다.

영속성 계층이 변하면 도메인 계층은 변해야 한다.

하지만 도메인 계층 내의 비즈니스 로직은 해당 어플리케이션의 핵심 로직이다. 이러한 핵심 로직은 쉽게 바꾸고 싶지 않다. 그렇다면 어떻게 해서 이 의존성을 역전시킬 수 있을까??

DIP에서 그 해답을 얻을 수 있다

![Untitled 3](https://user-images.githubusercontent.com/39071638/212551983-757fd2ee-e5f1-4a08-9892-e299291eadf0.png)
위와 같은 방식으로 영속성 계층에는 리포지토리의 구현체를

도메인 계층에는 추상체를 놓음으로써 의존성을 역전시킬 수 있다

# 클린 아키텍쳐

로버트 c마틴에 따르면 클린 아키텍쳐에서는 설계가 비즈니스 규칙의 테스트를 용이하게 하고, 비즈니스 규칙은 프레임워크, DB, UI기술, 그 밖의 외부 App이나 인터페이스로부터 독립적일 수 있다고 이야기 했다.

이는 도메인 코드가 바깥으로의 향하는 어떠한 의존성이 없어야 함을 의미한다. 대신 DI원칙의 도움으로 모든 의존성이 도메인 코드를 향하게 된다

![Untitled 4](https://user-images.githubusercontent.com/39071638/212551988-a5c153ef-8f62-442b-af91-e6fcb750518e.png)
위와 같이 도시화 될 수 있다. 

엔티티와 같은 코어에는 유스케이스, 컨트롤로와 같은 동심원으로 둘러 쌓여 있다.

그리고 유스케이스는 앞에서 나왔던 넓은 서비스의 문제를 피하기 위해 SRP를 가지기 위해서 조금 더 세분화 되어 있다.

코어 주변으로는 비즈니스 규칙을 지원하는 App의 다른 모든 컴포넌트들을 확인할 수 있다.

도메인 코드는 어떠한 의존성을 바깥으로 가지지 않기 때문에, 영속성 프레임워크나 UI 프레임워크에 의존하지 않는다. 그래서 어떤것이 사용되는지 알 수가 없어서 특화된 코드를 가지지 않고, 비즈니스 규칙에 집중할 수 있게 된다.

DDD의 가장 순수한 형태로 적용해 볼 수도 있다.

하지만 역시 대가가 따르기 마련.

도메인 계층이 영속성이나 UI와 같은 외부 계층과 철저하게 분리가 되어 있기 때문에, App의 엔티티에 대한 모델을 각 계층에서 유지보수 해야 한다.

예를 들어 영속성 계층에서 ORM을 사용한다고 가정했을 때, 도메인 계층과 영속성 계층에서 사용할 엔티티는 분리되어야 한다. 두 계층을 서로 모르기 때문이다. 즉, 두 계층이 서로 데이터를 주고 받을 때 두 엔티티를 변환해야 한다는 의미이다. 이는 다른 계층에서도 통용되는 이야기이다.

저자는 이를 바람직한 상태라고 기술한다.

JPA에서는 ORM이 관리하는 엔티티에 인자가 없는 기본 생성자(no args constructor)를 추가 하도록 강제한다.

이것이 바로 도메인 모델에는 포함해서는 안될 프레임워크에 특화된 결합의 예이다.

자 이제 조금 더 깊게 들어가서 클린 아키텍처의 원칙들을 구체적으로 만들어주는 헥사고날 아키텍처를 보겠다

# 헥사고날 아키텍쳐

![Untitled 5](https://user-images.githubusercontent.com/39071638/212551930-d83f6b2a-5ad7-4be5-ae73-8204d4e81ae5.png)
육각형이지만 육각형이 아니어도 된다. 그냥 4개 이상의 면만 가져도 된다. 

육각형 안에는 도메인 엔티티와 상호 작용하는 유스케이스가 존재한다. 

그리고 육각형 외부로 향하는 의존성은 없어야 한다

육각형 바깥에는 다양한 어댑터들이 존재한다.

웹, 영속성 등등이 존재한다

App의 코어 부분과 어댑터들간의 통신이 가능 하러면, 코어가 각각 포트를 제공해 주어야한다.

주도하는 어댑터(driving adapter)에게는 그러한 포트가 코어에 있는 유스케이스 클래스들에 의개 구현되고 호출되는 인터페이스가 될 것이고, 

주도되는 어댑터(driven adapter)에게는 그러한 포트가 어댑터에 의해 구혀노디고 코어에 의해 호출되는 인터페이스가 될 것이다
