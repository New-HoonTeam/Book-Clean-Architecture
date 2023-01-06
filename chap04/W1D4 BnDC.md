# 도메인 모델 구현하기

객체 지향적인 방식으로 Account 엔티티 모델링하고, 
출금 계좌에서 입금 계좌로 돈을 송금하는 유스케이스를 구현

```java
package buckpal.domain;

	public class Account {

		private AccountId id;
		private Money baselineBalance;
		private Activitywindow activitywindow;

		// 생성자와 getter는 생략

		public Money calculateBalance() {
			return Money.add(
				this.baselineBalance,
				this.activitywindow.calculateBalance(this.id));
		}

		public boolean withdraw(Money money, Accountld targetAccountld) { 
			if (!mayWithdraw(money)) {
				return false; 
		}

			Activity withdrawal = new Activity( 
				this.id, 
				this.id, 
				targetAccountld, 
				LocalDateTime.now(), 
				money);
		
			this.activitywindow.addActivity(withdrawal); 
			return true;
		}

		private boolean mayWithdraw(Money money) { 
			return Money.add(
				this.calculateBalance(), 
				money.negate())
			.isPositive();
		}

		public boolean deposit(Money money, AccountId sourceAccountId) {
			Activity deposit = new Activity( 
				this.id, 
				sourceAccountId, 
				this.id, 
				LocalDateTime.now(), 
				money);
			this.activitywindow.addActivity(deposit); 
			return true;
		} 
}
```

Account 엔티티는 실제 계좌의 현재 스냅숏을 제공한다.

- 계좌에 대한 모든 입금과 출금은 Activity 엔티티에 담기지만, 
모든 내역이 항상 메모리에 올리는 것은 비효율적이므로 
AcitivityWindow 값 객체(value object)에 
지난 며칠 혹은 몇 주간의 범위에 해당하는 내역만 보유한다.
- 계좌에서 일어나는 입금과 출금은 각각 withdraw()와 deposit() 메서드에서 
새로운 거래는 activityWindow에 추가한다.
- 출금하기 전에는 자고를 초과하는 금액은 출금할 수 없도록 하는 비즈니스 규칙을 검사한다.

입금과 출금을 할 수 있는 Account 엔티티가 있으므로 
이것을 통해서 바깥에서 유스케이스를 구현하기 위해 사용될 수 있다.

# 유스케이스 둘러보기

일반적으로 유스케이스는 다음과 같은 단계를 따라 처리 된다.

1. 인커밍 어댑터로부터 입력을 받는다.
    
    유스케이스는 도메인 로직에만 집중하고, 입력 유효성 검증은 다른 곳에서 처리한다.
    
2. 비즈니스 규칙을 검증한다.
유스케이스는 엔티티와 함께 비즈니스 규칙을 검증할 책임은 가진다.
3. 모델 상태를 조작한다.
    
    비즈니스 규칙을 충족하면, 모델의 상태를 변경하고, 
    영속성 어댑터를 통해 이 상태가 저장될 수 있도록 한다.
    그 이후 다른 아웃고잉 어댑터를 호출할 수도 있다.
    
4. 출력을 반환한다.
마지막 단계는 아웃고잉 어댑터에서 온 출력값을 
유스케이스로 호출한 어댑터 반환할 출력 객체로 변환하는 것이다.

이 단계를 염두에 두고 ‘송금하기’ 유스케이스를 구현해보자.

앞선 장에서 보았던 넓은 서비스 문제를 피하기 위해서 
각 유스케이스별로 분리된 각각의 서비스로 만들겠다.

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

- 인커밍 포트 인터페이스인 SendMoneyUseCase를 구현하고,
- 계좌를 불러오기 위해 아웃고잉 포트 인터페이스인 LoadAccountPort를 호출한다.
- 계좌 상태를 업데이트하기 위해 UpdateAccountStatePort를 호출한다.

![image](https://user-images.githubusercontent.com/86050295/211036218-9ffc326f-f885-49be-ad80-c4fb5c55bbb8.png)

# 입력 유효성 검증

입력 유효성 검증은 애플리케이션 계층의 책임에 해당한다.

- 호출하는 어댑터가 유스케이스에 입력을 전달하기 전에 입력 유효성을 검증할 경우 단점
    - 유스케이스는 여러 어댑터에서 호출되는데, 
    유효성 검증을 각 어댑터에서 전부 구현해야 한다.
    - 유효성 검증을 누락/실수할 가능성이 있다.
- 코어의 바깥에서 유효하지 않은 입력값을 받게 되면, 모델의 상태를 해칠 수 있기 때문에
애플리케이션 계층에서 입력 유효성을 검증해야 한다.

유스케이스가 아닌 애플리케이션 계층의 어디에서 입력 유효성 검증을 해야 할까?

- 입력 모델(input model)의 생성자 내에서 입력 유효성 검증하게 한다.
- 예제 코드에서 본 SendMoneyCommand 클래스가 이 역할을 한다.

## 직접 생성자서 검증

```java
package buckpal.application.port.in;

@Getter
public class SendMoneyCommand {

	private final AccountId sourceAccountId;
	private final AccountId targetAccountId; private final Money;
	
	public SendMoneyCommand(AccountId sourceAccountId, Accountld targetAccountId, Money money) {
		this.sourceAccountId = sourceAccountId;
		this.targetAccountId = targetAccountId;
		this.money = money
		requireNonNull(sourceAccountId);
		requireNonNull(targetAccountId);
		requireNonNull(money);
		requireGreaterThan(money, 0); 
	}
}
```

- 송금을 위해서는 출금 계좌, 입금 계좌ID, 송금할 금액이 필요하다
- 모든 파라미터가 null이 아니어야 하고, 송금액은 0보다 커야 한다.
- 하나라도 위배되면, 객체를 생성할 때 예외를 던져서 객체 생성을 막으면 된다.
- 모든 필드에 final 키워드를 지정해 불변 필드로 만든다.
- 변경될 수 없으므로 생성에 성공하면, 유효한 상태를 보장할 수 있다.

유효성 검증이 애플리케이션 코어에 남아있지만, 인커밍 포트 패키지에 위치하여, 
신성한 유스케이스 코드를 오염시키지 않는다.

## Bean Validation API 이용

자바에서 Bean Validation API는 유효성 검증을 해주는 사실상 표준 라이브러리다

필요한 유효성 규칙들을 필드의 애너테이션으로 표현할 수 있다.

```java
package buckpal.application.port.in;

@Getter
public class SendMoneyCommand extends SelfValidating<SendMoneyCommand> {

	@NotNull
	private final Account.Accountld sourceAccountld;
	@NotNull
	private final Account.Accountld targetAccountld;
	@NotNull private final Money;
	
	public SendMoneyCommand(
		Account.AccountId sourceAccountId, Account.Accountld targetAccountld, Money money
	) {
		this.sourceAccountld = sourceAccountld;
		this.targetAccountld = targetAccountld; 
		this.money = money;
		requireGreaterThan(money, 0); 
		this.validateSelf();
	} 
}
```

- SelfValidating 추상 클래스는 validateSelf() 메서드를 제공하며, 
이 메서드가 필드에 지정된 Bean Validation 애너테이션(@NonNull 같은)을 검증하고, 
유효성 검증 규칙을 위반한 경우 예외를 던진다.
- Bean Validation이 특정 유효성 검증 규칙을 표현하기에 충분하지 않다면 
송금액이 0보다 큰지 검사했던 것처럼 직접 구현할수도 있다.
- 잘못된 입력을 호출자에게 돌려주는 
유스케이스 구현체 오류 방지 계층(anti corruption layer)을 만들었다.
(여기서 계층은 layered 아키텍처에서 계층을 의미 하지 않는다. 
단지 유스케이스 앞에 위치해서 잘못된 입력을 검증해주는 보호막 역할을 의미한다)

SelfValidating 클래스의 구현은 다음과 같다.

```java
package shared;

public abstract class SelfValidating<T> {

	private Validator;
	public SelfValidating() {
		ValidatorFactory factory = Validation.buildDefaultValidatorFactory(); 
		validator = factory.getValidator();
	}

	protected void validateSelf() {
		Set<ConstraintViolation<T>> violations = validator.validate((T) this);
		if (!violations.isEmpty()) {
			throw new ConstraintViolationException(violations);
		} 
	}
}
```

# 생성자의 힘

SendMoneyCommand는 생성자에 많은 책임

- 불변 클래스 → final키워드가 붙은 속성이 생성자 인자에 모두 들어가야 함
- 생성자가 파라미터 유효성 검증

→ 유효하지 않은 상태의 객체를 만드는 것은 불가능

만약 파라미터가 더 많다면 어떻게 해야 할까?

- 빌더(Builder) 패턴을 활용 → 긴 파라미터 리스트를 private로 만들고 빌더의 
build() 메서드 내부 생성자 호출을 숨길 수 있다.
- 파라미터가 20개인 생성자를 호출하는 대신 다음과 같이 객체를 생성할 수 있다.

```java
new SendMoneyCommandBuilder()
	.sourceAccountId(new AccountId(41L))
	.targetAccountId(new AccountId(42L)) 
	// ... 다른 여러 필드를 초기화
	.build();
```

유효성 검증 로직은 생성자에 그대로 둬서 빌더가 
유효하지 않은 상태의 객체를 생성하지 못하도록 막을 수 있다.

SendMoneyCommandBuilder에 필드를 새로 추가해야 하는 상황에서
빌더를 호출하는 코드에 새로운 필드를 추가하는 것을 잊을 수도 있다.

- 런타임에 유효성 검증 로직이 누락된 파라미터에 대해 에러를 던지긴 하겠지만,
유효하지 않은 상태의 불변 객체를 만들려는 시도에 대해서 경고해주지 못한다.
- 빌더 패턴이 아닌 생성자를 직접 사용했다면, 필드를 추가하거나 삭제할 때마다,
 컴파일 에러를 따라 변경사항을 반영할 수 있었을 것
    - 좋은 IDE는 파라미터명 힌트도 주고, 파라미터 리스트도 깔끔하게 포매팅할 수 있다.

![image](https://user-images.githubusercontent.com/86050295/211036335-36123670-38c1-4bf1-8dd7-1c325fae88a0.png)

이 정도면 컴파일러를 믿고 생성자만으로 검증해보는 것도 좋지 않을까?

# 유스케이스마다 다른 입력 모델

다른 유스케이스에 동일한 입력 모델을 사용하고 싶은 유혹

- 계좌 등록하기, 계좌 정보 업데이트는 거의 똑같은 계좌 상세 정보가 필요하다.
- 차이점은 등록하기는 소유자의 ID 정보를, 업데이트는 계좌 ID 정보를  필요로 한다는 것
    
    → 두 유스케이스에서 같은 입력 모델을 공유할 경우 등록하기에 
    계좌 ID에 null, 업데이트에서는 소유자 ID에 null 값을 허용해야 한다.
    

불변 커맨드 객체의 필드에 대해서 null이 허용되는 것은 그 자체로 code smell(나쁜 코드)이다.

- 등록하기, 업데이트 유스케이스 각각에 서로 다른 유효성 검증 로직이 필요 
→ 유스케이스에 커스텀 유효성 검증 로직을 넣어야 한다.
- 비즈니스 코드를 입력 유효성 검증과 관련된 관심사로 오염시키게 된다.
- 만약, 등록하기 유스케이스에 계좌 ID 필드에 null이 아닌 값이 들어오면 어떻게 해야 할까?
    - 에러를 던져야 할까? 무시해야 할까?
    - 이 코드를 유지보수할 엔지니어들이 코드를 보면 충분히 던질 법한 질문들

⇒ 유스케이스 마다 전용 입력 모델을 사용하게 되면, 유스케이스를 훨씬 명확하게 만들고 
다른 유스케이스와의 결합도를 제거해서 불필요한 부수효과가 발생하지 않게 한다.

# 비즈니스 규칙 검증하기

언제 입력 유효성을 검증하고, 언제 비즈니스 규칙을 검증해야 할까?

- 입력 유효성을 검증하는 것은 구문상(syntactical)의 유효성을 검증하는 것이고,
- 비즈니스 규칙은 유스케이스의 맥락 속에서 의미적인(semantical)유효성을 검증하는 일이다.

이 책에서 제시하는 구분법

도메인 모델의 현재 상태에 접근이 필요한 것은 비즈니스 규칙 검증이고,
그럴 필요 없는 검증은 입력 유효성 검증이다.

- 출금 계좌는 초과 출금되어서는 안 된다는 규칙
모델의 현재 상태에 접근해야 하기 때문에 비즈니스 규칙이다.
- 송금되는 금액은 0보다 커야 한다는 규칙
모델에 접근하지 않고도 검증될 수 있으므려, 입력 유효성 검증으로 볼 수 있다.

→ 논쟁의 여지가 있을 수 있다 송금액이 비즈니스에서 매우 중요한 값이기 때문에 
이것을 검증하는 것은 어떤 경우라도 비즈니스 규칙으로 다뤄야 한다는 주장을 할 수도 있다.

→ 앞선 구분 법은 검증 로직을 코드 상의 어느 위치에 둘지 결정하고 
나중에 그것이 어디에 있는지 더 쉽게 찾는 데 도움이 된다.

비즈니스 규칙 검증은 어떻게 구현 해야 할까?

1. 도메인 안에서 하기 
→ 이렇게 하면 비즈니스 로직과 비즈니스 규칙과 같은 곳에 위치하게 되어 추론하기 쉽다.
2. 유스케이스 코드에서 하기
(만약 도메인 엔티티에 비즈니스 규칙을 검증하기 여의치 않다면)

출금 계좌는 초과 인출되어서는 안 된다는 비즈니스 규칙

엔티티에서 검증하는 예시

```java
package buckpal.domain;

public class Account {

	// ...
	public boolean withdraw(Money, Accountld targetAccountId) {
		if (!mayWithdraw(money)) { 
			return false;
		} 
	// ...
	} 
}
```

유스케이스에서 검증하는 예시

```java
package buckpal.application.service;

@RequiredArgsConstructor
@Transactional
public class SendMoneyService implements SendMoneyUseCase {

	// ...
	@Override
	public boolean sendMoney(SendMoneyCommand command) { 
		requireAccountExists(command.getSourceAccountId()); 
		requireAccountExists(command.getTargetAccoiintId());
	}
}
```

유효성을 검증이 실패할 경우 유효성 검증 전용 예시를 던진다.

사용자와 통신하는 어댑터는 이 예외를 에러 메시지로 
사용자에게 보여주거나 적절한 다른 방법으로 처리

더 복잡한 비즈니스 규칙의 경우 먼저 데이터베이스에서 
도메인 모델을 로드해서 상태를 검증해야 할 수도 있다. 
(이렇게 하더라도 어쨌든 도메인 엔티티 내에 비즈니스 규칙을 구현해야 한다.)

# 풍부한 도메인 모델 vs 빈약한 도메인 모델

풍부한 도메인 모델 (Account 엔티티 예시에서 사용한 방식)

- 엔티티에서 가능한 많은 도메인 로직을 구현
- 많은 비즈니스 규칙이 구현체 대신 엔티티에 위치
- 엔티티에서는 비즈니스 규칙에 맞는 유효한 변경만을 허용
- 유스케이스는 도메인 모델의 진입점으로 동작
    - 엔티티들은 상태를 변경하는 메서드를 제공하고, 유스케이스에서는 엔티티 메서드 호출

빈약한 도메인 모델 

- 엔티티는 상태를 표현하는 필드와 getter, setter 메서드만 포함하고 어떤 도메인 로직도 가지고 있지 않다.
- 도메인 로직이 유스케이스 클래스에 구현돼 있다
- 비즈니스 규칙을 검증하고 엔티티 상태를 바꾸고, 데이터베이스 저장을 담당하는 아웃고잉 포트에 엔티티를 전달할 책임이 유스케이스 클래스에 있다

각자의 필요에 맞는 스타일을 자유롭게 택해서 사용하면 된다.

# 유스케이스마다 다른 출력 모델

유스케이스가 할 일을 다하고 호출자에게 무엇을 반환해야 할까?

- 출력은 호출자에게 꼭 필요한 데이터만 들고 있어야 한다.

송금하기 유스케이스에서는 boolean 값 하나를 반환했지만, 
업데이트된 Account를 반환하고 싶을 수도 있다.

→ 정답은 없지만, 유스케이스를 가능한 한 구체적으로 유지하기 위해서 이것이 필요한 값인지 계속해서 의문을 가져야 하고, 만약 확신이 없다면 가능한 한 적게 반환하자.

유스케이스들이 같은 출력 모델을 공유하게 되면 유스케이스들이 강하게 결합된다.

- 어떤 유스케이스 출력에 새로운 필드가 필요해지면, 
이 값과 관련 없는 유스케이스 출력에도 이 필드를 처리해야 한다.
- 공유 모델은 장기적으로 점점 커지게 된다.
- 단일 책임 원칙을 적용하고 모델을 분리해서 
유지하는 것은 유스케이스의 결합을 제거하는데 도움이 된다.

도메인 엔티티를 출력 모델로 사용하고 싶은 유혹도 견뎌야 한다.

- 도메인 엔티티가 변경될 이유를 필요 이상으로 늘리지 말아야 한다.
- 엔티티를 입력 모델이나 출력 모델로 사용하는 것에 대해서 11장에서 자세히 다룰 것

# 읽기 전용 유스케이스는 어떨까?

읽기 전용 작업을 유스케이스라고 말하는 것은 이상하다

(만약 전체 프로젝트 맥락에서 이런 작업이 유스케이스로 분류된다면 
어떻게든 다른 유스케이스와 비슷한 방식으로 구현해야 한다.)

애플리케이션 코어의 관점에서는 이 작업은 간단한 데이터 쿼리다.
실제 유스케이스와 구분하기 위해 쿼리로 구현할 수 있다.

쿼리를 위한 인커밍 전용 포트를 만들고, 이를 ‘쿼리 서비스(query service)’에 구현하는 것이다.

```java
package buckpal.application.service;

@RequiredArgsConstructor
class GetAccountBalanceService implements GetAccountBalanceQuery {
	private final LoadAccountPort loadAccountPort;

	@Override
	public Money getAccountBalance(AccountId accountId) {
		return loadAccountPort.loadAccount(accountId, LocalDateTime.now())
			.calculateBalance();
	} 
}
```

쿼리 서비스는 유스케이스 서비스와 동일한 방식으로 동작한다.

- GetAccountBalanceQuery라는 인커밍 포트를 구현하고,
- 데이터베이스로부터 실제로 데이터를 로드하기 위해 
LoadAccountPort라는 아웃고잉 포트를 호출한다.
- 이처럼 읽기 전용 쿼리는 쓰기가 가능한 
유스케이스(또는 ‘커맨드’)와 코드 상에서 명확하게 구분된다.
    - 이런 방식은 CQS(Command-Query Separation)나
    CQRS(Command - Query Responsibility Segregation) 같은 개념과 아주 잘 맞는다.

앞의 예제 코드에서 서비스는 아웃고잉 포트로 쿼리를 전달하는 것 외에 다른 일을 하지 않는다. 

CQRS : 데이터를 조회하는 작업과 데이터를 업데이트하는 작업의 책임을 분리하는 패턴으로서 ‘명령 쿼리 책임 분리’

# 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?

유스케이스별로 입출력 모델을 각각 만들면

단점

- 모델을 공유할 때보다 더 많은 작업이 필요하다.
    - 각 유스케이스마다 별도의 모델을 만들어야 한다.
    - 모델마다 엔티티을 매핑해야 한다.

장점

- 유스케이스를 명확하게 이해할 수 있다
- 장기적으로 유지보수하기 쉽다
- 여러 개의 유스케이스를 동시에 작업할 수 있다.
