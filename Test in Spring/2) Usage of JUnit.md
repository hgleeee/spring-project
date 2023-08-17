# JUnit을 활용한 Java 단위 테스트 코드 작성

## 1. Java 단위 테스트(Unit Test) 작성 준비
### 필요한 라이브러리
- 요즘 Java 단위테스트 작성에는 크게 2가지 라이브러리가 사용된다.
  - JUnit5: 자바 단위 테스트를 위한 테스팅 프레임워크
  - AssertJ: 자바 테스트를 돕기 위해 다양한 문법을 지원하는 라이브러리
- JUnit 만으로도 단위 테스트를 충분히 작성할 수 있다.
  - 하지만 JUnit에서 제공하는 assertEquals()와 같은 메소드는 AssertJ가 주는 메소드에 비해 가독성이 떨어진다.
  - 그렇기 때문에 순수 Java 애플리케이션에서 단위 테스트를 위해 JUnit5와 AssertJ 조합이 많이 사용된다.

### given/when/then 패턴 
- 요즘 단위 테스트는 거의 given-when-then 패턴으로 작성하는 추세이다.
- given-when-then 패턴이란 1개의 단위 테스트를 3가지 단계로 나누어 처리하는 패턴으로, 각각의 단계는 다음을 의미한다.
  - given(준비): 어떠한 데이터가 준비되었을 때
  - when(실행): 어떠한 함수를 실행하면
  - then(검증): 어떠한 결과가 나와야 한다.
- 추가적으로 어떤 메소드가 몇 번 호출되었는지를 검사하기 위한 verify 단계도 사용하는 경우가 있는데, 그렇게 실용성이 크지 않으므로 메소드의 호출 횟수가 중요한 테스트에서만 선택적으로 사용하면 된다.

### 테스트 코드 작성 공통 규칙
```java
@DisplayName("로또 번호 갯수 테스트")
@Test
void lottoNumberSizeTest() {
    // given

    // when

    // then
}
```

- @Test는 해당 메소드가 단위 테스트임을 명시하는 어노테이션이다.
  - JUnit은 테스트 패키지 하위의 @Test 어노테이션이 붙은 메소드를 단위 테스트로 인식하여 실행시킨다.
- 이 상태로 실행하면 테스트 이름으로 함수 이름이 default로 지정되는데, 우리는 @DisplayName 어노테이션을 사용하여 읽기 좋은 다른 이름을 부여할 수 있다.
- 또한 테스트 코드는 앞서 설명한 given-when-then 구조로 흔히 작성되는데, 단위 테스트 내에 주석으로 이 단계를 명시해주면 읽기 좋은 테스트 코드를 작성할 수 있다.

## 2. 단위 테스트 작성 예시
### 로또 생성기 Java 코드
- 예를 들어 다음과 같이 1000원을 주면 1개의 로또를 생성해주는 클래스가 있다고 하자.

```java
public class LottoNumberGenerator {

    public List<Integer> generate(final int money) {
        if (!isValidMoney(money)) {
            throw new RuntimeException("올바른 금액이 아닙니다.");
        }
        return generate();
    }

    private boolean isValidMoney(final int money) {
        return money == 1000;
    }

    private List<Integer> generate() {
        return new Random()
                .ints(1, 45 + 1)
                .distinct()
                .limit(6)
                .boxed()
                .collect(Collectors.toList());
    }

}
```

- 위와 같은 로또 번호 생성 코드에 대한 테스트 코드들을 작성해보도록 하자.

### 1) 로또 번호 갯수 테스트 
- 우선 로또를 생성받기 위해서는 로또 생성기 객체와 금액이 필요하다.
  - 그렇기에 given 단계에서는 LottoNumberGenerator 객체와 금액을 적어주면 된다.

```java
@DisplayName("로또 번호 갯수 테스트")
@Test
void lottoNumberSizeTest() {
    // given
    final LottoNumberGenerator lottoNumberGenerator = new LottoNumberGenerator();
    final int price = 1000;

    // when

    // then
}
```

- 준비가 끝났으면 주어진 금액을 지불하여 로또를 받아야 한다. 이에 대한 when 단계의 코드를 작성하면 다음과 같다.

```java
@DisplayName("로또 번호 갯수 테스트")
@Test
void lottoNumberSizeTest() {
    // given
    final LottoNumberGenerator lottoNumberGenerator = new LottoNumberGenerator();
    final int price = 1000;

    // when
    final List<Integer> lottoNumber = lottoNumberGenerator.generate(price);

    // then
}
```

- 이제 최종적으로 우리가 받은 로또가 6개의 숫자를 갖는지 검증해야 한다.

```java
@DisplayName("로또 번호 갯수 테스트")
@Test
void lottoNumberSizeTest() {
    // given
    final LottoNumberGenerator lottoNumberGenerator = new LottoNumberGenerator();
    final int price = 1000;

    // when
    final List<Integer> lottoNumber = lottoNumberGenerator.generate(price);

    // then
    assertThat(lotto.size()).isEqualTo(6);
}
```

 
### 2) 로또 번호 범위 테스트
- 이번에는 모든 로또 숫자가 1에서 45사이의 숫자인지를 boolean 값으로 검사하므로, AssertJ의 isTrue() 문법이 사용된다.
  - 그 외에도 isFalse(), isNull(), isNotNull() 등의 메소드가 있다.

```java
@DisplayName("로또 번호 범위 테스트")
@Test
void lottoNumberRangeTest() {
    // given
    final LottoNumberGenerator lottoNumberGenerator = new LottoNumberGenerator();
    final int price = 1000;

    // when
    final List<Integer> lotto = lottoNumberGenerator.generate(price);

    // then
    assertThat(lotto.stream().allMatch(v -> v >= 1 && v <= 45)).isTrue();
}
```

### 3) 잘못된 로또 금액 테스트
- 마지막으로 잘못된 금액이 발생한 경우, Runtime Exception이 발생하는 코드에 대해 테스트를 해야 한다.
- 예외가 발생하는 경우에는 when 단계에서 assertThrows()로 감싸서 처리를 해야 한다.

```java
@DisplayName("잘못된 로또 금액 테스트")
@Test
void lottoNumberInvalidMoneyTest() {
    // given
    final LottoNumberGenerator lottoNumberGenerator = new LottoNumberGenerator();
    final int price = 2000;

    // when
    final RuntimeException exception = assertThrows(RuntimeException.class, () -> lottoNumberGenerator.generate(price));

    // then
    assertThat(exception.getMessage()).isEqualTo("올바른 금액이 아닙니다.");
}
```

- 이전 코드들과 다르게 금액을 2000원으로 변경하였고, 실행하는 메소드를 assertThrows()로 감싸 주었다.
- 간단한 자바 애플리케이션이라서 어떤 메소드가 다른 객체와 메세지를 주고 받을 필요가 없는 경우라면 단위 테스트 작성이 간단하다.
  - 하지만 일반적인 애플리케이션은 상당히 복잡하고, 여러 객체들이 메세지를 주고 받는다.
  - 그렇기에 Spring과 같은 웹 애플리케이션에서는 어떻게 단위 테스트를 작성하는지에 대해 알아볼 필요가 있다.

