# Lombok 주의사항

- 인턴 업무를 할 때 Lombok이 편리해서 썼었는데, 코드 리뷰를 받으며 Lombok 어노테이션은 주의해서 써야 한다고 말씀해주셨다.
- 이를 계기로 Lombok 어노테이션 주의사항을 찾아보게 되었다.

<br>

## `@AllArgsConstructor, @RequiredArgsConstructor` 사용 주의

### ▶️ 오류 발생 가능 상황
```java
@AllArgsConstructor
public static class Order {
    private Long cancelPrice;
    private Long orderPrice;
}
Order order = new Order(5000L, 10000L);
```
- 해당 어노테이션을 사용하면, 필드의 순서대로 인자를 받는 생성자를 만들어준다.
- 이때 이후에 Order 클래스의 필드 순서를 바꾸게 되면, Lombok 어노테이션에 의해 필드 선언 순서에 맞춰 인자 순서가 바뀐 생성자를 만들게 된다.
- 위와 같은 경우 두 필드가 모두 동일한 타입이어서 기존 생성자 호출 코드에서는 오류가 발생하지 않기 때문에 순서가 바뀐 부분을 알아채기 쉽지 않다.

<br>

### ▶️ 개선
- 해당 문제는 `@AllArgsConstructor, @RequiredArgsConstructor` 사용시 발생 가능하기에 해당 어노테이션들은 사용하지 않는 것이 좋다.
- 대신, 생성자를 직접 만들거나 `@Builder`를 사용하는 것을 권장한다.
  - `@Builder`는 파라미터 순서가 아닌 이름으로 값을 설정하기 때문에 변경에 유연하다.

<br>

### 🌈 참고 자료
- https://kwonnam.pe.kr/wiki/java/lombok/pitfall