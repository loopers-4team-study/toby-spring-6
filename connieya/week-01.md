# 1주차 스터디 정리

## 학습 내용

### InputStream → InputStreamReader → BufferedReader 이해

#### 코드 예시

```java
BufferedReader br = new BufferedReader(new InputStreamReader(connection.getInputStream()));
```

#### 각 단계별 역할

##### 1. InputStream (바이트 스트림)

- **역할**: 바이트 단위로 데이터를 읽음
- **특징**:
    - 네트워크에서 받은 데이터는 바이트 단위로 전송됨
    - 1바이트 = 8비트 (바이너리 데이터: 0과 1)
    - 모든 데이터는 결국 바이너리로 저장됨

##### 2. InputStreamReader (바이트 → 문자 변환)

- **역할**: 바이트를 문자(char)로 변환하는 브릿지 역할
- **특징**:
    - 인코딩(UTF-8, EUC-KR 등)을 고려하여 바이트를 문자로 변환
    - 단순히 "사람이 알아보기 쉽게"가 아니라 **문자 인코딩을 처리한 문자 변환**
    - UTF-8에서 ASCII 영역(0-127)은 ASCII와 동일하게 매핑됨

##### 3. BufferedReader (효율적인 문자 읽기)

- **역할**: 버퍼링을 통한 효율성 향상 + 편의 메서드 제공
- **특징**:
    - 여러 문자를 묶어서 읽어 I/O 횟수를 줄여 성능 향상
    - `readLine()` 메서드를 제공하여 한 줄씩 읽기 편리
    - "보기 쉽게"보다는 **효율성과 편의성**이 주목적

#### 데이터 변환 과정 예시

```
네트워크에서 받은 데이터 (바이트):
[01001000] [01100101] [01101100] [01101100] [01101111]

↓ [2진법 → 10진법 변환]

InputStream이 읽는 값:
[72] [101] [108] [108] [111] (바이트 값들)

↓ [인코딩 해석 - ASCII/UTF-8]

InputStreamReader가 변환:
'H' 'e' 'l' 'l' 'o' (문자들)

↓ [버퍼링 + 편의 메서드]

BufferedReader로 문자열 조합:
"Hello" (문자열)
```

#### 변환 과정 상세 설명

**2진법 → 10진법 변환 예시:**

```
01001000 (2진법) = 0×2⁷ + 1×2⁶ + 0×2⁵ + 0×2⁴ + 1×2³ + 0×2² + 0×2¹ + 0×2⁰
                 = 0 + 64 + 0 + 0 + 8 + 0 + 0 + 0
                 = 72 (10진법)
```

**10진법 → 문자 변환 (ASCII/UTF-8):**

```
72 → 'H'
101 → 'e'
108 → 'l'
111 → 'o'
```

#### 핵심 정리

```
InputStream (바이트 단위로 읽기)
    ↓ [인코딩 변환]
InputStreamReader (문자로 변환)
    ↓ [버퍼링 + 편의 메서드]
BufferedReader (효율적인 문자 읽기)
```

### JSON 파싱 관련 학습

#### Jackson 라이브러리 사용

- `ObjectMapper`를 사용하여 JSON 문자열을 Java 객체로 변환

#### 어노테이션 활용

##### @JsonIgnoreProperties(ignoreUnknown = true)

- **용도**: JSON에 있지만 Java 클래스에 없는 필드를 무시
- **사용 이유**:
    - API 응답에 불필요한 필드가 많을 때
    - 필요한 필드만 매핑하고 나머지는 무시하고 싶을 때

##### @JsonProperty

- **용도**: JSON 필드명과 Java 필드명을 매핑
- **사용 이유**:
    - JSON은 `conversion_rates` (스네이크 케이스)
    - Java는 `conversionRates` (카멜 케이스)
    - Java 컨벤션을 유지하면서 JSON 매핑 가능

**예시:**

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record ExRateData(
        String result,
        @JsonProperty("conversion_rates")
        Map<String, BigDecimal> conversionRates
) {
}
```

### 실습 내용

1. **HttpURLConnection을 사용한 API 호출**

    - URL, HttpURLConnection을 사용하여 외부 API 호출
    - ExchangeRate API를 활용한 환율 정보 조회

2. **JSON 응답 파싱**
    - Jackson의 ObjectMapper를 사용하여 JSON → Java 객체 변환
    - Record를 활용한 데이터 모델링

## 참고사항

- 네트워크 데이터는 항상 바이트 단위로 전송됨
- 텍스트도 결국 바이너리로 저장되고 전송됨
- 인코딩을 올바르게 처리하지 않으면 문자가 깨질 수 있음
- BufferedReader는 단순히 "보기 쉽게"가 아니라 성능 최적화와 편의성을 위한 것
