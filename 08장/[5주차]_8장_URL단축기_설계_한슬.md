8장 URL 단축기 설계를 읽기 전에

URL 단축기가 하는 역할이 무엇이고, 이것을 설계해야 하는 이유가 무엇인지에 대해서 알아보았습니다.

---

# 0️⃣ URL 단축기란?

1. **사용자 편의성 증대**
    - 간결함
        - `https://www.example.com/products/electronics/smartphones/2025-model-x?utm_source=newsletter&utm_medium=email` 과 같은 긴 주소를
        `https://bit.ly/3xYabc` 처럼 짧게 만들어 줍니다.
    - **공유 용이성**
        - 짧은 주소는 문자 메세지(SMS), SNS(글자수 제한이 있는 트위터), 등 구두 전달 등에서 공유하기 편리 합니다.
    - **오프라인 활용**
        - 포스터, 명함, 책 등 인쇄물에 짧은 주소는 사용자가 직접 입력하기 쉽습니다.
        
2. **데이터 추적 및 분석**
    
    URL 단축기는 단순한 리다이렉션 서비스가 아닙니다. 사용자가 짧은 URL을 클릭하면, 
    
    최종목적지로 바로 이동하는 것이 아니라 아주 잠깐 URL 단축기 서버를 거쳐 갑니다. 
    
    바로 이 순간, 단축기 서버는 클릭에 대한 모든 정보를 기록할 수 있습니다.
    
    ![image.png](attachment:e190f186-6b38-4bf1-924a-5a81a78f95e3:image.png)
    
    - **클릭 수**: 얼마나 많은 사람이 링크를 클릭했는지
    - **클릭 시간**: 언제 주로 클릭이 발생했는지
    - **사용자 위치**: 어느 국가나 도시에서 클릭했는지
    - **리퍼러**: 어떤 경로를 통해 유입되었는지
    - **사용자 기기 및 브라우저**: 모바일인지, PC인지, 어떤 브라우저를 사용했는지 등
3. **링크 관리 유연성**
    
    단축 URL은 일종의 ‘영구적인 별명’과 같습니다. 이 별명이 가리키는 실제 주소(최종 목적지)는 언제든지 바꿀 수 있습니다.
    
    예를 들어, 이벤트 홍보 포스터에 단축 URL을 인쇄해 배포했는데, 나중에 이벤트 페이지 주소가 변경되었다고 가정해봅니다.
    
    원본 URL을 사용했다면 모든 포스터를 폐기해야 하지만, 단축 URL을 사용했다면 단축 URL 서비스 대시보드에서 최종 도착 주소만 변경하면 됩니다.
    
    인쇄된 포스터의 단축 URL은 그대로 유효합니다.
    
4. **단축기 서비스**

> https://buly.kr/
https://url.kr/index.php
> 

---

# 1️⃣ 문제 이해 및 설계 범위 확정

- URL 단축기가 어떻게 동작해야 하는지?
    - `https://www.example.com/products/electronics/smartphones/2025-model-x?utm_source=newsletter&utm_medium=email` 과 같은 긴 주소를
    `https://bit.ly/3xYabc` 와 같은 단축 URL을 결과로 제공해야 하며, 이 URL에 접속하면 원래 URL로 갈 수 있어야 한다.
- 트래픽 규모
    - 매일 1억개의 단축 URL을 만들어 낼 수 있어야 한다.
- 단축 URL의 길이는 어느정도?
    - 짧을수록 좋다
- 단축 URL에 포함될 문자에 제한이 있는지?
    - 숫자 (0~9), 영문자 (소문자, 대문자 포함 a-z, A-Z)
- 단축된 URL을 시스템에서 삭제하거나 갱신할 수 없다고 가정.

**개략적 추정**

- 쓰기 연산 : 매일 1억개의 단축 URL 생성
- 초당 쓰기 연산: 1억 / (24 * 60 * 60 ) =  약 1157.40 ⇒ 1160
- 읽기 연산: 읽기 연산과 쓰기 연산 비율을 10:1 이라고 가정한다.
- 초당 읽기 연산: (1160 * 10) =  11,600
- URL 단축 서비스를 10년간 운영한다고 가정하면?
    - 1억 * 365 * 10 = 3650 억 개의 레코드를 보관해야 한다.
- 축약 전 URL의 평균 길이는 100(byte) 이라고 가정한다.
- 따라서 10년 동안 필요한 저장 용량은 3650억 * 100byte = **약 36.5TB**
    - 이후에 보면 hashTable 혹은 관계형db로 저장관리를 할것이므로 사실 계산한 용량보다는 더 필요합니다.

---

# 2️⃣ 개략적 설계안 제시

## **URL 단축기 API 엔드포인트 명세 (API Endpoint)**

### **1. 단축 URL을 생성**

원본 URL(longUrl)을 전달받아 고유한 단축 URL을 생성하고 반환합니다.

- **HTTP Method**: `POST`
- **Endpoint**: `/api/v1/data/shorten`
- **Request Body (**application/json**)**
    
    ```json
    {
      "longUrl": "https://www.example.com/products/electronics/smartphones/2025-model-x?category=123"
    }
    ```
    
- **Response Body (Success)**
    
    ```json
    {
      "shortUrl": "https://your.domain/3xYabc"
    }
    ```
    

### **2. 단축 URL 리다이렉션**

생성된 단축 URL로 접근 시, 저장된 원본 URL로 리다이렉트시킵니다.

- **HTTP Method**: `GET`
- **Endpoint**: `/{shortUrlCode}`
    - *참고: `shortUrlCode`는 `https://your.domain/` 뒷부분의 고유 코드(예: `3xYabc`)를 의미합니다. 사용자가 이 경로로 직접 접속하는 것을 처리합니다.*
- **Success Response**
    - **Status Code**: `301 Moved Permanently` 또는 `302 Found`
    - **Headers**: `Location` 헤더에 원본 URL이 포함됩니다.
    - **Body**: 없음
    
    ```bash
    HTTP/1.1 301 Moved Permanently
    Location: https://www.example.com/products/electronics/smartphones/2025-model-x?category=12
    ```
    
- **💡 `301` vs `302` 리다이렉션**
    - **`301 Moved Permanently`**
    영구적으로 주소가 변경되었음을 의미합니다. 
    브라우저는 이 응답을 캐싱(기억)하여 다음 요청부터는 단축기 서버를 거치지 않고 바로 원본 URL로 접속할 수 있습니다. 
    **서버 부하를 줄일 수 있지만, 클릭 수 분석은 첫 방문에만 집계될 수 있습니다.**
    - **`302 Found`**
    일시적인 리다이렉션을 의미합니다. 
    브라우저는 이 응답을 캐싱하지 않으므로 **매번 클릭할 때마다 단축기 서버를 거칩니다. 
    따라서 모든 클릭에 대한 통계를 정확하게 추적할 수 있습니다.**

---

# 3️⃣ 상세 설계

## 적절한 해시 값 길이는?

hashValue는 [0-9, a-z, A-Z]의 문자들로 구성된다.

사용할 수 있는 문자의 갯수는 10 + 26 + 26 = 62개

hashValue의 길이를 정하기 위해서는    62^n ≥ 3650억 인 n의 최솟값을 찾아야 한다.

**___ ___ ___ ___ ___ ___ ___  …     n → 7** 

경우의수   62 * 62 * 62 * …. (n)번 ..  n의 최솟값은 7 

![image.png](attachment:0c33518c-1189-4610-ade5-1b274a667761:image.png)

---

## 단축URL 시나리오

### **해시 후 충돌 해소**

![image.png](attachment:254672da-0cb4-42e0-8e59-b84e5e01327f:image.png)

간단한 해시 함수로는  CRC32, MD5, SHA-1 등이 있다.
https://www.convertstring.com/ko/Hash 을 축약해보자.

| Hash Function | Result |
| --- | --- |
| SHA1 | 91CF8DBC4F2681BD858B8B8A4DD72A34E90FB869 |
| MD5 | 2ACDFDCF22FDB2D93A571F22F65E2F28 |
| CRC32 | F3E0B8F0 |

7글자 보다 더 크기 때문에 앞 7글자만 이용 해서 데이터베이스에 저장한다.

그러나 이 방법은 해시 결과가 서로 충돌할 확률이 높아진다.

충돌이 해소될 때 까지 사전에 정한 문자열을 기존 URL에 덧붙여 해시를 계산한다. ([salt](https://ko.wikipedia.org/wiki/%EC%86%94%ED%8A%B8_(%EC%95%94%ED%98%B8%ED%95%99)))

이 시나리오는 충돌을 해소할 때 까지 데이터베이스에 접근해야 한다는 단점이 있습니다.

---

### **base-62 변환 ([Base10 to Base62](https://math.tools/calculator/base/10-62))**

**base62란 바이너리 데이터를 영문 대소문자와 숫자 62가지 문자로 표현하는 방식**으로, 

[Base64](https://www.google.com/search?sca_esv=3cba3ff7c6207a53&cs=0&sxsrf=AE3TifMYTJ07DdE1aM2ptj_hexKHp9Hbuw%3A1758548324214&q=Base64&sa=X&ved=2ahUKEwiTmaOBv-yPAxUrk68BHc-eAOwQxccNegQIAhAC&mstk=AUtExfBRL0fclJ1hf9gvqssi8BfK-YeWLfqIEf92tVurE-vMP2v5vIrrrEeYO1DvcXF2KphnYeUW41SmL4jOC0SZHqEl6vp-a6aEGAsAIx28k7uD_8dOvHQhI2bh8h_udXKa6UJ-iOqJm7EeqE5ijtz7L45aoLffagzT5IgYjHphLR2wV4Y8z5rHu05RwWYHmhGn91a-htCww2iNEPvcZZwyz9xaEpxenxo-5TDoMjAIDthvygBwdeKWs6fFN3dIGqH_obRgmz9yklxg06-yzE7sCyZ9&csui=3)와 달리 URL이나 파일 이름 등에 바로 사용할 수 있도록 특수 문자가 없어 안전하게 사용할 수 있다는 장점이 있습니다. 

Base64가 64개의 문자를 사용한다면, **Base62는 0-9, A-Z, a-z 총 62개의 문자만을 사용하여 데이터를 표현**합니다.

![image.png](attachment:3d83e89b-da4b-46bb-8ab1-10e71eafbbe7:image.png)

- 입력될 url을 https://www.convertstring.com/ko/Hash 라고 가정합니다.
- 이 URL에 대해 ID 값을 생성합니다. → 2509222242301
- 이 ID를 62진수로 변환하면  *iAvYoSj*
- 아래 표와 같은 새로운 데이터베이스 레코드를 만듭니다.

| ID | shortURL | longURL |
| --- | --- | --- |
| 2509222242301 |  iAvYoSj | https://www.convertstring.com/ko/Hash  |

위 방식은 ID의 유일성이 보장된 후에야 적용 가능한 전략이라 충돌 상황은 발생하지 않는 장점이 있습니다.

ID는  yyMMddHHmmss(seq) 형식으로 임의적으로 만들었습니다.

현 상황에서는 위 ID 생성 규칙은 적절해보이지 않네요.

적절한 방법으로 무엇이 있는지 같이 상의해보는 것이 좋을것같습니다!!

---

## URL 리다이렉션 상세 설계

![image.png](attachment:fbbde597-c3a4-4df6-a6d5-d6dc9635da71:image.png)

1. 사용자가 단축 URL을 클릭한다.
2. 로드밸런서가 해당 클릭으로 발생한 요청을 웹 서버에 전달한다.
3. 단축 URL이 이미 캐시에 있는 경우에는 원래 URL을 바로 꺼내서 클라이언트에게 전달한다.
4. 캐시에 해당 단축 URL이 없는 경우에는 데이터베이스에서 꺼낸다. 데이터베이스에 없다면 아마 사용자가 잘못된 단축 URL을 입력한 경우일 것이다.
5. 데이터베이스에서 꺼낸 URL을 캐시에 넣은 후 사용자에게 반환한다.
