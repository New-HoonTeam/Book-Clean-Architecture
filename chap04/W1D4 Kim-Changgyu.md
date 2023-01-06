### 도메인 모델 구현하기

한 계좌에서 다른 계좌로 송금하는 유스케이스를 구현하기 위한 한 가지 방법은 입금과 출금을 할 수 있는 Account 엔티티를 만들고 출금 계좌에서 출금하여 입금 계좌로 입금하는 것이다.

```java
public class Account {
	private Accountld id;
	private Money baselineBalance;
	private Activitywindow activitywindow;

	// ...

	public Money calculateBalance() { ... }

	public boolean withdraw(...) { ... }

	private boolean mayWithdraw(...) { ... }

	public boolean deposit(...) { ... }
}
```

Account 엔티티는 실제 계좌의 현재 스냅샷, Activity는 계좌에 대한 모든 입금과 출금을 담는다.
그러나 계좌에 대한 모든 활동들을 항상 메모리에 올리는 것은 부담되기 때문에 특정 기간에 해당하는 활동만 Activitywindow에 담아 보유하도록 한다.

현재 잔고를 계산하기 위해서는 baselineBalance + sum(activitywindows) 로 계산한다.
출금에는 현재 잔액을 초과하는 금액을 인출할 수 없도록 비즈니스 규칙(mayWithdraw)을 검사한다.

입출금의 책임을 갖는 Account 엔티티가 있으므로 이를 중심으로 유스케이스를 구현하기 위해 바깥 방향으로 나아갈 수 있다.

---

### 유스케이스 둘러보기

일반적인 유스케이스는 다음과 같은 단계를 따른다.

1. 입력을 받는다.
2. 비즈니스 규칙을 검증한다.
3. 모델 상태를 조작한다.
4. 출력을 반환한다.

유스케이스는 입력 어댑터로부터 입력을 받는데, 이 단계를 ‘입력 유효성 검증’으로 부르지 않는 이유는 유스케이스 코드가 도메인 로직에만 신경써야하고 입력 유효성 검증으로 오염되지 않아야 한다고 생각하기 때문이다.
그러나 **유스케이스는 비즈니스 규칙을 검증할 책임**이 있다. 그리고 도메인 엔티티와 이 책임을 공유한다.
(위에서 말한 입력 유효성 검증과 비즈니스 규칙 검증의 차이점은 후반부에서 다룸)

비즈니스 규칙을 충족하면 유스케이스는 입력을 기반으로 어떤 방법으로든 모델의 상태를 변경한다.
도메인 객체의 상태를 바꾸고 영속성 어댑터를 통해 상태를 저장한다든지 다른 출력 어댑터를 호출한다든지…

마지막 단계는 출력 어댑터에서 온 출력값을, 유스케이스를 호출한 어댑터로 반환할 출력 객체로 변환하는 것이다.

```java
package buckpal.application.service;

@RequiredArgsConstructor
@Transactional
public class SendMoneyService implements SendMoneyUseCase {
	private final LoadAccountPort LoadAccountPort;
	private final AccountLock accountLock;
	private final UpdateAccountStatePort updateAccountStatePort;

	@Override
	public boolean sendMoney(SendMoneyCommand command) {
		// TODO: 비즈니스 규칙 검증
		// TODO: 모델 상태 조작
		// TODO: 출력 값 반환
	}
}
```

![IMG1](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F7ba88edd-7541-4a78-855d-637e62a4e994%2FUntitled.png?id=92d14ae9-57e0-43aa-87fe-5443255bd5a3&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=1910&userId=&cache=v2)

---

### 입력 유효성 검증

앞에서 입력 유효성 검증은 유스케이스 클래스의 책임이 아니라고 했지만 이 작업은 애플리케이션 계층의 책임에 해당하기 때문에 지금 논의하는 게 적절할 것 같다.

<aside>
❓ 입력 유효성 검증을 호출하는 어댑터에서 입력을 전달하기 전에 검증한다면 어떨까?

</aside>

유스케이스에서 필요로 하는 검증을 호출자(Caller, 호출하는 어댑터)가 모두 검증했다고 확실할 수 있을까?
유스케이스는 하나 이상의 어댑터에서 호출될 수 있는데, 그러면 유효성 검증은 각 어댑터에서 전부 구현해야하지 않을까? 그 과정에서 실수하거나 잊어버리면 어떡하지…?

→ 애플리케이션 계층에서 입력 유효성을 검증해야 하는 이유는 애플리케이션 코어 바깥쪽으로부터 유효하지 않는 입력값을 받아서 모델 상태를 손상시킬 위험을 방지하기 위함이다.

<aside>
❓ 유스케이스 클래스 책임이 아니라면 어디서 입력 유효성 검증을 해야할까?

</aside>

**입력 모델(Input Model)**이 이 문제를 다루도록 할 수 있다.
’송금하기’ 유스케이스에서 입력 모델은 예제 코드에서 본 SendMoneyCommand 클래스이다.
더 정확히 말하자면 생성자 내에서 입력 유효성을 검증할 것이다.

```java
package buckpal.application.port.in;

@Getter
public class SendMoneyCommand {
	private final Accountld sourceAccountld;
	private final Accountld targetAccountld;
	private final Money;

	public SendMoneyCommand(
			Accountld sourceAccountld,
			Accountld targetAccountld,
			Money money) {
		requireNonNull(sourceAccountld);
		requireNonNull(targetAccountld);
		requireNonNull(money);
		requireGreaterThan(money, 0);

		this.sourceAccountld = sourceAccountld;
		this.targetAccountld = targetAccountld;
		this.money = money;
	}
}
```

출금, 입금 계좌의 ID와 송금할 금액에 대한 조건을 만족하지 않으면 예외를 던져 객체 생성을 막으면 된다.
불변(final) 필드로 지정했기 때문에 성공적인 검증이 이루어져 생성된 이후에는 잘못된 상태로 갈 수 없다.

SendMoneyCommand는 유스케이스 API의 일부이기 때문에 애플리케이션의 입력 포트 패키지 내에 위치한다.
그러므로 유효성 검증이 애플리케이션의 코어에 남아있지만 유스케이스 코드를 오염시키지 않게 된다.

이 귀찮은 작업들은 자바에서 **Bean Validation API**를 통해 애너테이션으로 필요한 규칙을 쉽게 표현할 수 있다.

```java
package buckpal.application.port.in;

@Getter
public class SendMoneyCommand extends SelfValidating<SendMoneyCommand> {
	@NotNull
	private final Account.Accountld sourceAccountld;
	@NotNull
	private final Account.Accountld targetAccountld;
	@NotNull
	private final Money;

	public SendMoneyCommand(
			Account.Accountld sourceAccountld,
			Account.Accountld targetAccountld,
			Money money) {
		this.sourceAccountld = sourceAccountld;
		this.targetAccountld = targetAccountld;
		this.money = money;
		requireGreaterThan(money, 0); 
		this.validateSelf();
	}
}
```

SelfValidating 추상 클래스는 validateSelf() 메서드를 제공하고 코드에서 생성자의 마지막에서 호출하고 있다.
이 메서드가 필드에 지정된 Bean Validation 애너테이션을 검증하고 유효성 검증 규칙을 위반하면 예외를 던진다.

입력 모델에 있는 유효성 검증 코드를 통해 유스케이스 구현체 주위에 **사실상 오류 방지 계층**을 만든 것이다.
(계층형 아키텍처의 계층이 아닌 유스케이스의 보호막 역할)

---

### 생성자의 힘

위 예시에서는 파라미터가 3개이기 때문에 복잡하지 않지만 더 많은 파라미터가 온다면 생성자를 통해 인스턴스를 생성하는 것보다 생성자를 private으로 만들고 빌더 패턴을 활용하는 것을 고려해보자.

기존의 유효성 검증 코드는 생성자에 그대로 두고 유효하지 않은 객체의 생성을 방지할 수 있다.

그런데 특정 필드가 추가, 삭제, 변경될 경우에는 빌더, 생성자, 유효성 검증이 모두 수정돼야 하기 때문에 자칫하다가 일관성 있게 수정하지 않을 경우 예상하지 못 한 문제가 발생할 수 있다.

좋은 IDE는 파라미터명 힌트를 주거나 충분히 깔끔하게 포매팅할 수 있기에 생성자의 힘을 충분히 믿어도 된다.

![IMG2](https://p1dgey.notion.site/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F11e5f8e8-4930-4c80-8ab1-a43339aa0e37%2FUntitled.png?id=db1e0f4d-056a-48a1-b77e-391ff96fa130&table=block&spaceId=b76551b9-9f24-4a91-9bcd-340caa404f60&width=1890&userId=&cache=v2)

---

### 유스케이스마다 다른 입력 모델

‘계좌 정보 업데이트’ 유스케이스와 ‘계좌 등록하기’ 유스케이스가 같은 입력 모델을 사용할 경우에는 계좌 등록시에 필요없는 계좌 ID 필드를 공유하게 그에 따른 유효성 검증 규칙이 오염될 가능성이 있다.
그에 따라서 비즈니스 코드에 커스텀한 유효성 검증 로직이 붙어 오염될 수 있다. (관심사의 오염)

**각 유스케이스 전용 입력 모델은 유스케이스를 훨씬 명확하게 만들고 다른 유스케이스와의 결합도 제거해서 불필요한 부수효과를 발생하지 않게 한다.** 물론 들어오는 데이터를 각 유스케이스에 해당하는 입력 모델에 매핑해야 하기 때문에 비용이 발생하기도 한다.

---

### 비즈니스 규칙 검증하기

입력 유효성 검증은 유스케이스 로직의 일부가 아니지만 비즈니스 규칙 검증은 엄연히 유스케이스 로직의 일부다.
비즈니스 규칙은 애플리케이션 핵심이기에 적절히 잘 다뤄야 한다.

<aside>
❓ 그럼 언제 입력 유효성 검증을 하고 언제 비즈니스 규칙을 검증할까?

</aside>

둘 사이의 아주 실용적인 구분점은 **비즈니스 규칙 검증은 도메인 모델의 현재 상태에 접근**해야 하는 반면, 입력 유효성 검증은 그럴 필요가 없다는 것이다.

입력 유효성 검증은 구문상(Syntactical) 유효성을 검증하는 것. 즉 애너테이션을 붙인 것 처럼 선언적인 반면 비즈니스 규칙은 유스케이스 맥락 속에서 의미적인(Sementical) 유효성을 검증하는 일이라고 볼 수 있다.

- 비즈니스 규칙 → “출금 계좌는 초과 출금되어서는 안 된다”
    - 출금 계좌의 잔액을 확인하고 입금 계좌가 존재하는지 확인해야 하기 때문에 모델의 현재 상태에 접근
- 입력 유효성 검증 → “송금되는 금액은 0보다 커야 한다”
    - 모델에 접근하지 않고도 검증 가능

(물론 송금액에 대한 규칙은 중요하기 때문에 비즈니스 규칙으로 끌어내릴 수 있다)

<aside>
❓ 비즈니스 규칙 검증은 어떻게 구현할까?

</aside>

가장 좋은 방법은 앞에서 “출금 계좌는 초과 인출되어서는 안 된다” 규칙에서처럼 **비즈니스 규칙을 도메인 엔티티 안에 넣는 것**이다.

```java
package buckpal.domain;

public class Account {
	// ...

	public boolean withdraw(Money, Accountld targetAccountld) {
		if (!mayWithdraw(money)) {
			return false;
		}
		// ...
	}
}
```

만약 도메인 엔티티에서 검증하기 여의치 않다면 유스케이스 코드에서 도메인 엔티티를 사용하기 전에 해도 된다.

```java
package buckpal.application.service;

@RequiredArgsConstructor
@Transactional
public class SendMoneyService implements SendMoneyUseCase {
	// ...
	@Override
	public boolean sendMoney(SendMoneyCommand command) {
		requireAccountExists(command.getSourceAccountld());
		requireAccountExists(command.getTargetAccoiintld());
		...
	}
}
```

유효성 검증이 실패할 경우 유효성 검증 전용 예외를 던지면 사용자와 통십하는 어댑터를 이 예외를 에러 메시지로 사용자에게 보여주거나 적절한 다른 방법으로 처리할 수 있다.

---

### 풍부한 도메인 모델 vs. 빈약한 도메인 모델

**풍부한 도메인 모델**에서는 애플리케이션의 코어에 있는 **엔티티에서 가능한 많은 도메인 로직이 구현**된다.
엔티티들은 상태를 변경하는 메서드를 제공하고, 비즈니스 규칙에 맞는 유효한 변경만을 허용한다.
유스케이스는 도메인 모델의 진입점으로 동작한다. 이어서 유스케이스는 사용자의 의도만을 표현하면서 이 의도를 **실제 작업을 수행하는 체계화된 도메인 엔티티 메서드 호출로 변환**한다. **많은 비즈니스 규칙이 유스케이스 구현체 대신 엔티티에 위치**하게 된다.

**빈약한 도메인 모델**에서는 엔티티 자체가 굉장히 얇다. 일반적으로 엔티티는 **상태를 표현하는 필드**와 이 값을 읽고 바꾸기 위한 **Getter/Setter 메서드만 포함**하고 어떠한 도메인 로직도 가지고 있지 않다.
즉, 도메인 로직이 유스케이스 클래스에 구현돼 있다는 것이다. 비즈니스 규칙을 검증하고 엔티티의 상태를 바꾸고, 데이터베이스 저장을 담당하는 출력 포트에 엔티티를 전달한 책임 역시 유스케이스 클래스에 있다.
**(’풍부함’이 엔티티 대신 유스케이스에 존재하는 것)**

→ 정답은 없으며 현재 아키텍처에서 요구하거나 각자 필요에 맞는 스타일을 자유롭게 택해서 사용하면 된다.

---

### 유스케이스마다 다른 출력 모델

입력과 비슷하게 출력도 가능하면 각 유스케이스에 맞게 구체적일수록 좋다. 출력은 호출자에게 꼭 필요한 데이터만 들고 있어야 한다. 이 범위에 정답은 없지만 가능한 적게 반환하고 다음과 같은 고민을 끊임없이 하자.

- 유스케이스에서 정말로 이 데이터를 반환해야 할까?
- 호출자가 정말로 이 값을 필요로 할까?
- 다른 호출자도 사용할 수 있도록 해당 데이터에 접근할 전용 유스케이스를 만들어야 하지 않을까?

유스케이스들 간 같은 출력 모델을 공유하면 서로 **강하게 결합**된다. 입력 모델을 공유하게 됐을 때 처럼 **공유 모델**은 장기적으로 **각 유스케이스 별로 필요없는 데이터를 처리해야할 확률이 높아진다.**

(개인적인 생각) 또 도메인 엔티티를 출력 모델로 사용하고 싶은 유혹도 이겨내야 한다. 출력에 맞춰 도메인 엔티티를 변경할 이유가 많아지는 것을 경계하고 출력 모델과 도메인 엔티티가 항상 일치할 지도 의문을 가져야 한다.

---

### 읽기 전용 유스케이스는 어떨까?

간단한 읽기 전용 작업을 유스케이스로 분류하게 되면 애플리케이선 코어 관점에서 간단한 데이터 쿼리이기 때문에 실제 쓰기 작업이 이루어지는 작업과 달리 쿼리로 구현할 수 있다.

이 책에서 이를 구현하는 한 가지 방법으로 쿼리를 위한 입력 전용 포트를 만들고 이를 ‘쿼리 서비스’를 소개한다.

```java
package buckpal.application.service;

@RequiredArgsConstructor
class GetAccountBalanceService implements GetAccountBalanceQuery {
	private final LoadAccountPort loadAccountPort;

	@Override
	public Money getAccountBalance(AccountId accountld) {
		return loadAccountPort.loadAccount(accountld, LocalDateTime.now())
								.calculateBalance();
	}
}
```

쿼리 서비스는 유스케이스 서비스와 동일한 방식으로 동작하지만 읽**기 전용 쿼리와 쓰기가 가능한 유스케이스를 코드에서 명확하게 구분**할 수 있게 된다.

---

### 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?

유스케이스별로 (입력/출력)모델을 만들면 유스케이스를 명확하게 이해할 수 있고, 장기적으로 유지보수하기 쉽게 만들어준다. 또 여러 명의 개발자가 다른 사람이 작업 중인 유스케이스를 건드리지 않은 채로 여러 개의 유스케이스를 동시에 작업할 수 있게 된다.

→ **꼼꼼한 입력 유효성 검증(명확한 입력 유효성 검증과 비즈니스 규칙의 분리), 유스케이스별 분리된 입출력 모델은 지속 가능한 코드 생성에 도움이 된다.**
