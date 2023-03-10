# Chapter 4 : 유스케이스 구현하기
육각형 아키텍처에서 UseCase 구현하기  
- 육각형 아키텍처는 Domain 중심의 architecture에 적합    
  -> Domain Entity 만들고 Entity 중심으로 UseCase 구현
  
<br/>
 
- [유스케이스 둘러보기](#유스케이스-둘러보기)
  - [flow](#flow)
    - [입력 유효성 검증은 어디서 하는게 좋을까](#입력-유효성-검증은-어디서-하는게-좋을까) 
    - [입력받기](#입력받기)
    - [비즈니스 규칙 검증하기](#비즈니스-규칙-검증하기)
    - [출력하기](#출력하기)
- [풍부한 도메인 모델 vs 빈약한 도메인 모델](#풍부한-도메인-모델-vs-빈약한-도메인-모델)
- [읽기 전용 유스케이스는 어떨까](#읽기-전용-유스케이스는-어떨까)
- [마무리](#마무리)    

<br/>

## 유스케이스 둘러보기 

### flow

- `1) 입력을 받는다`
  - 입력 유효성 검증 안함    
    (유스케이스 코드에서는 입력 유효성을 검증하지 않고, 입력 유효성 검증은 다른 곳에서 처리)
    - 도메인 로직에만 신경쓰기  
    - 입력 유효성 검증은 책임에 벗어남
- `2) 비즈니스 규칙을 검증한다`
- `3) 모델 상태를 변경한다`
  - Domain 객체의 상태를 바꾸고 -> port(영속성 adapter를 통해 구현된)로 바꾼 상태를 전달해서 저장 
- `4) 출력을 반환한다`
  - `출력값`(아웃 고잉 어댑터에서 온)을 어댑터(유스케이스를 호출한)로 반환할 `출력 객체로 변환`   

<br/>
   
![제목 없음](https://user-images.githubusercontent.com/103614357/210598643-6941437b-6110-4ece-817f-adba900ca307.png)    

**ex) 송금하기**  

- 하나의 서비스(SendMoneyService)가
  - 하나의 유스케이스(인커밍 포트 인터페이스인 SendMoneyUseCase) 구현
    - 계좌 불러옴 (아웃고잉 포트 인터페이스인 LoadAccountPort 호출해서)
  - 도메인 모델 변경(송금한만큼 변경)  
  - 변경된 상태(변경된 계좌상태)를 저장(데이터베이스에)하기 위해 아웃고잉 포트(UpdateAccountStatePort) 호출  

<br/>

### 입력 유효성 검증은 어디서 하는게 좋을까

- adapter -> UseCase에 입력 전달 전에 입력 유효성을 검증한다면? 
  - 모든 adapter에서 유효성 검증을 구현해야 함
  
<br/>
 
=> `application 계층에서 입력 유효성`을 검증하자       
- 그렇지 않으면 애플리케이션 코어 바깥쪽으로부터 유효하지 않은 입력 값을 받게 되어 모델의 상태를 해침   
- but UseCase class에서 입력 유효성을 검증하지는 않음
- 유스케이스 `입력 모델의 생성자`에서 `입력 유효성을 검증`하는 것이 좋음  

  ```java
  @Value
  @EqualsAndHashCode(callSuper = false)
  public
  class SendMoneyCommand extends SelfValidating<SendMoneyCommand> {

      @NotNull
      private final AccountId sourceAccountId;

      @NotNull
      private final AccountId targetAccountId;

      @NotNull
      private final Money money;

      public SendMoneyCommand(
              AccountId sourceAccountId,
              AccountId targetAccountId,
              Money money) {
          this.sourceAccountId = sourceAccountId;
          this.targetAccountId = targetAccountId;
          this.money = money;
          this.validateSelf();
      }
  }
  ```

- 생성자에서 유효성 검증을 했을 때의 `장점`     
  - 클래스가 불변이기 때문에 생성자의 인자 리스트에 클래스의 각 속성에 해당하는 파라미터를 포함
  - 생성자 내에서 파라미터의 유효성 검증 -> `유효하지 않은 상태의 객체`를 만드는 것이 `불가능`  

<br/><br/> 

**빌더 패턴을 사용하면?**   
- 유효성 검증 로직은 생성자에 그대로 있음 -> 빌더가 유효하지 않은 상태의 객체를 생성하지 못하도록 막을 수 있음   
  - 그런데 만약 새로운 필드를 추가하면? + 빌더를 호출하는 코드에 새로운 필드를 추가하는 것을 잊으면?   
    - 컴파일러에서는 유효하지 않은 불변 객체를 만들려는 시도에 대해 경고해주지는 못하지만    
      `런타임에` 유효성 검증 로직이 동작해서 누락된 파라미터에 대한 `에러`를 던질 것    
      (`생성자`를 직접 사용했다면 `컴파일 에러`)
       
<br/>

- `저자`는 `빌더 패턴 없이 생성자를 호출`하는 것이 좋다는 점 어필   
  - 컴파일의 도움을 받을 수 있음 (생성자를 직접 사용했다면 컴파일 에러에 따라 나머지 코드에 즉각 변경사항을 반영할 수 있었을 것)
  - IDE에서 파라미터명 보여줌 -> 생성자에 많은 인자가 들어가도 어떤 필드가 누락되었는지 잘 알 수 있음   
   
<br/><br/>

### 입력받기
- 저자는 `Use Case마다 입력모델을 분리`하는 것 권장
  - ex) MoneyCommand 객체 -> SendMoneyCommand / GetMoneyCommand 등으로 분리   
- 각 Use Case 전용 입력 모델은   
  - null을 받을 필요도 없고 
  - Use Case를 훨씬 정확하게 만들고   
  - 다른 Use Case와의 결합도 제거   
 
<br/><br/>

### 비즈니스 규칙 검증하기
- 비즈니스 규칙 검증
  - `도메인 모델의 현재 상태`에 접근 -> Use Case의 맥락 속에서 `의미적인 유효성 검증`하는 일
    - ex) 출금 계좌는 초과 출금되어서는 안된다       
      (입력 유효성 검증은 ex) 송금되는 금액은 0보다 커야 한다)     
  - `Domain Entity 안에` 넣는 것이 좋음
    - 만약 상황상 그게 어렵다면, Use Case 코드에서 Domain Entity를 사용하기 전에 해도 됨   
      - 유효성 검증하는 코드를 호출하고, 유효성 검증이 실패할 경우 유효성 검증 전용 예외를 던지게          

<br/><br/>

### 출력하기  
- 입력과 비슷하게 출력도 각 유스케이스에 맞게 `구체적`일수록 좋음   
  - `공유 모델`은 장기적으로 보았을 때 `점점 비대`해지게 됨 + 공유모델로 인해 `유스케이스들도 강하게 결합`됨  
- 출력 모델은 `호출자에게 꼭 필요한 데이터`만 들고 있어야함  

<br/><br/>

## 풍부한 도메인 모델 vs 빈약한 도메인 모델

**풍부한 도메인 모델(rich domain model)**       
- application core에 있는 `Entity에서 많은 도메인 로직`이 구현   

  = 대부분의 비즈니스 규칙들은 Entity에 위치하게 됨  

  => Use Case에는 사용자의 의도만을 표현하게 됨

<br/>

**빈약한 도메인 모델(anemic domain model)**     
- 일반적으로 getter / setter 정도만 표현

  => `Use Case에 비즈니스 로직`이 구현되게 됨

<br/><br/>

## 읽기 전용 유스케이스는 어떨까  

- ex) 계좌 잔액 `조회 기능`을 위해 새로운 유스케이스를 구현해야 할까?   
  - 읽기전용 작업을 유스케이스라고 언급하는 것 조금 이상  
  - application core 관점에서 이 작업은 간단한 `데이터 쿼리`      
    => 프로젝트 맥락에서 유스케이스로 간주되지 않는다면, `실제 유스케이스와 구분`하기 위해 `쿼리로 구현`할 수 있음
	
<br/>   

**읽기전용 쿼리서비스를 구현하는 방법**     

- `쿼리를 위한 인커밍 전용포트`를 만들고 이를 `쿼리서비스에 구현`하기   

```java
@RequiredArgsConstructor
class GetAccountBalanceService implements GetAccountBalanceQuery {
	private final LoadAccountPort loadAccountPort;

	@Override
	public Money getAccountBalance(AccountId accountId){
		return loadAccountPort.loadAccount(accoutId, LocalDateTime.now()).calculateBalance();
	}
}
```

- 쿼리 서비스 구현(GetAccountBalanceService)
  - Use Case 서비스와 동일한 방식으로 동작
    - GetAccountBalanceQuery라는 인커밍 포트 구현 + 아웃고잉 포트(LoadAccountPort)를 호출해 `쿼리를 전달`하는 것 외에 다른 일을 하지 않음  

<br/><br/>

## 마무리    
- `유스케이스 간에 모델을 공유`하는 것보다 `별도 모델을 만들고, 엔티티를 매핑`하는 등의 `추가 작업`을 해주자   
  - 유스케이스를 명확하게 `이해`할 수 있고 장기적으로 `유지보수`하기도 더 쉬움      
  - 여러명이 개발할 때, 같은 유스케이스를 건드리는 일 없이 여러개의 유스케이스를 `동시에 작업`할 수 있음         

=> 꼼꼼한 입력 유효성 검증, `유스케이스 별 입출력 모델`은 지속가능한 코드를 만드는데 큰 도움이 된다   

<br/>
