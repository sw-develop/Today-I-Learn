# JPA Entity와 데이터베이스간 ENUM 매핑

<br>

### 간단 패턴화
- 방식1 : @Enumerated를 사용해 Enum의 index 값 or 텍스트 값 자체 저장
- 방식2 : AttributeConverter 인터페이스를 구현한 커스텀 Converter 클래스 생성하여 처리

<br>

## 📌 상황
회사에서 약관 조회 API 관련 업무를 진행하며 약관 필드의 타입이 Enum으로 설정되어 있었고, 이를 계기로 Enum 저장 및 반환에 대해 정리하였다. 

<br>

## 📌 Enum이란 무엇인가
기존에 상수를 정의할 때 사용하던 ```final static String 방식```을 개선하여 나온 것으로, Enum은 열거형이라고 불리며, 서로 연관된 상수들의 집합을 의미한다.

<br>

## 📌 Enum 타입의 Entity 필드를 데이터베이스에 저장하는 방법

<br>

### ▶️ 방식1. @Enumerated(value = EnumType.{...})

![image](https://user-images.githubusercontent.com/69254943/180801697-b4dd57a3-d1a1-4431-a3ab-5bcfde6c2831.png)

- ORDINAL : 해당 Enum의 index 값이 DB에 저장된다.
  - 단점
    - Enum 클래스에 정의해둔 Enum 값들의 순서가 바뀔 경우 원하는 데이터를 잘못 읽어오는 문제가 발생한다.


- STRING : 해당 Enum의 텍스트 값 그대로 저장된다.
  - 단점
    - String을 DB에 저장하기 때문에, int 보다는 메모리 용량을 더 차지한다. (하지만 이는 데이터가 아주 많을 때의 문제이기는 함)
    - Enum 클래스에 정의해둔 Enum 값의 명칭을 코드 상에서만 변경한 경우 데이터베이스로부터 데이터를 읽어올 때 알맞은 Enum 값이 없어 에러가 발생한다. (가장 치명적인 문제)

<br>

### ▶️ 방식2. Enum에 대한 AttributeConverter 인터페이스를 구현한 Converter 클래스 생성

- 기존에 정의되어 있던 Enum 값들의 순서가 바뀌거나 명칭이 바뀌는 경우에 대한 단점을 보완한다.
- Enum 값을 DB에 저장할 때와 DB에서 읽어올 때(JPA Entity에 매핑할 때) 작동하는 함수를 오버라이딩하는 로직을 추가해줘야 한다.

<br>

### 구현 방식 예시
- Enum 클래스

```java
@Getter
public enum TermsTypeCd {

    ServiceUseTerms("10", "ServiceUseTerms", "서비스 이용약관"),
    PrivacyPolicy("20", "PrivacyPolicy", "개인정보 취급방침")
    ;

    private String codeValue;
    private String nameValue;
    private String korNameValue;

    TermsTypeCd(String codeValue, String nameValue, String korNameValue) {
        this.codeValue = codeValue;
        this.nameValue = nameValue;
        this.korNameValue = korNameValue;
    }

    public static TermsTypeCd enumOf(String codeValue) {
        return Arrays.stream(TermsTypeCd.values())
                .filter( t -> t.getCodeValue().equals(codeValue))
                .findAny().orElse(null);
    }
}
```

- Entity의 Enum 필드에 @Converter(..) 설정

```java
//Entity의 Enum 필드에 @Convert(converter = ..)
@Convert(converter = TermsTypeCdConverter.class)
private TermsTypeCd termsType;
```

- AttributeConverter 인터페이스를 구현한 Converter 클래스

```java
public class TermsTypeCdConverter implements AttributeConverter<TermsTypeCd, String> {

    @Override
    public String convertToDatabaseColumn(TermsTypeCd attribute) {	//DB에 저장
        if(attribute == null) {
            return null;
        }
        return attribute.getCodeValue();
    }

    @Override
    public TermsTypeCd convertToEntityAttribute(String dbData) {	//Entity로 반환
        if(dbData == null){
            return null;
        }
        return TermsTypeCd.enumOf(dbData);
    }
}
```

<br>

## 📌 추가) Controller에서 존재하지 않는 Enum 요청 값에 대한 예외 처리

<br>

### ▶️ 예시 상황

```java
@GetMapping("/pharm/terms/{pharmTermsTypeCd}")
public ResponseEntity<SingleResult<PharmTermsResDto>> findPharmTerms(
	@PathVariable("pharmTermsTypeCd") PharmTermsTypeCd pharmTermsTypeCd
) {
        ...
        return new ResponseEntity<>(result, HttpStatus.OK);
}
```

- 위의 코드에서처럼 PathVariable로 들어온 값을 Enum(여기서는 PharmTermsTypeCd) 값과 매칭 시켜 처리한다.
- 전달된 값에 해당하는 Enum 값이 존재하지 않는 경우, 기본으로 ```MethodArgumentTypeMismatchException 예외```를 반환한다.

<br>

## 📌 추가) 내가 채택한 존재하지 않는 Enum 값에 대한 예외 처리 방안

처리 방안으로는 아래와 같이 2가지 방식이 존재한다.

### ▶️ 방식1. Enum 타입 자체로 받아서 스프링부트에서 예외 처리 (바로 위에서 언급한 내용)
- 기본으로 MethodArgumentTypeMismatchException 예외를 발생시킨다.
- @RestControllerAdvice와 @ExceptionHandler를 사용해 글로벌 예외처리 가능하다.
- 구체적인 예외 메시지 반환을 위해 발생한 특정 예외에 대한 메시지를 잡아서 반환해줘야 한다.

<br>

### ▶️ 방식2. String으로 받고 Enum으로 변환할 때 예외 처리
- ‘해당 약관이 존재하지 않는다’는 구체적인 예외 내용 반환 가능하다.
- try-catch문 코드를 추가해 처리해줘야 한다.

<br>

### ▶️ 결정
- 방식1을 채택하였고, 그 이유는 다음과 같다.
  - 팀장님께 위의 2가지 방식에 대해 말씀드렸고, 답변으로 지금 상황에서 Enum 값 자체로 받을 수 있는데, 방식2처럼 String으로 받는 것은 불필요하다고 말씀해주셨다.
  - 또한 만약 해당 API가 사용자에 의한 요청이면 방식1로 하되 구체적인 예외 처리를 반드시 해줘야 하지만, 해당 API는 프론트에 의한 요청이므로 500 에러 반환을 유지하기로 하였다.

<br>

### 🌈 참고
- https://techblog.woowahan.com/2600/ → 회사 코드도 해당 글의 내용처럼 Converter의 공통된 부분을 분리할 필요가 있어보인다.
- https://pamyferret.tistory.com/entry/Enum-JPA로-enum-name-그대로-DB에-저장하기Enumerated
- https://jojoldu.tistory.com/122?category=635881
- https://techblog.woowahan.com/2527/