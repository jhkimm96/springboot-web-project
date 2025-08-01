## 📁 05. 회원 목록 조회 + 검색 + 페이징 기능 (MyBatis 기반)

### ✅ 개요
SpringBoot + MyBatis + Thymeleaf 조합으로 회원 리스트 조회
검색 기능 및 페이지네이션 구현

---

## ✅ 회원 리스트 조회 로직 구성

### 1. 컨트롤러 → 서비스 → MyBatis 흐름
- `MemberController` → `MemberService` → `MemberServiceImpl`
- `@RequiredArgsConstructor`로 DI (생성자 주입)  
  → 생성자 주입이 필요한 이유는 **final 필드를 초기화**하기 위해서이며, Lombok의 `@RequiredArgsConstructor`는 이 과정을 자동으로 처리해줌.

### 2. Repository vs Mapper
- 기존 JPA 사용 시 `MemberRepository.findAll()` 사용 가능
- MyBatis 사용 시 `MemberMapper` 인터페이스 생성하고 XML 매핑 파일 작성 필요

---

## ✅ MyBatis 설정

### 1. build.gradle 추가 의존성
```groovy
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.3.0'
```

### 2. application.yml 설정
```yaml
mybatis:
  mapper-locations: classpath:/mybatis/*.xml
  configuration:
    map-underscore-to-camel-case: true
  type-aliases-package: com.example.member.dto
```

---

## ✅ Mapper와 DTO 구성

### 1. DTO vs Model vs Entity 차이점
| 구분 | 용도 |
|------|------|
| Entity | DB 테이블 매핑용 |
| DTO | Controller ↔ Service ↔ View 데이터 전달용 |
| Model | 웹 View 전용(Spring MVC의 Model 객체) |

💡 DTO 하나만 통일해서 사용하는 경우도 많으나, **역할 구분이 명확한 프로젝트일수록 DTO/Entity 분리**가 권장됨.

---

## ✅ 검색 기능

- 검색 파라미터용 DTO: `MemberParam`
- Mapper XML에서 `parameterType="MemberParam"` 지정 가능
- 컨트롤러에서 받은 `MemberParam`을 서비스 → Mapper로 전달

---

## ✅ 페이징 기능

### 1. 핵심 유틸 클래스: `PageUtil`

- totalCount, pageIndex, pageSize 등을 받아 HTML 페이징 생성
- `StringBuilder`를 사용하는 이유는 **문자열 연결 성능 최적화** 때문
    - `+` 연산보다 빠르며 반복 처리에 적합

### 2. SQL (회원 수 조회)
```sql
SELECT COUNT(*) FROM member
```

- Mapper XML에 `selectListCount` 등록
- `resultType="long"`으로 설정

### 3. SQL (목록 조회)
```sql
SELECT * FROM member
LIMIT #{pageStart}, #{pageSize}
```

- `MemberParam`에 `getPageStart()`, `getPageSize()` 등 메서드 정의

---

## ✅ DTO 순번 처리

- 화면에 보여줄 순번을 계산하여 `setSeq()` 호출
```java
memberDto.setSeq(totalCount - parameter.getPageStart() - i);
```

---

## 🔍 참고 코드: PageUtil

```java
public class PageUtil {
    private long totalCount;
    private long pageSize = 10;
    private long pageBlockSize = 10;
    private long pageIndex;
    private long totalBlockCount;
    private long startPage;
    private long endPage;
    private String queryString;

    public PageUtil(long totalCount, long pageIndex, String queryString, long pageSize, long pageBlockSize) {
        this.totalCount = totalCount;
        this.pageSize = pageSize;
        this.queryString = queryString;
        this.pageBlockSize = pageBlockSize;
        this.pageIndex = pageIndex;
    }

    public String pager() {
        init();

        StringBuilder sb = new StringBuilder();

        long previousPageIndex = startPage > 1 ? startPage - 1 : 1;
        long nextPageIndex = endPage < totalBlockCount ? endPage + 1 : totalBlockCount;

        String addQueryString = "";
        if (queryString != null && queryString.length() > 0) {
            addQueryString = "&" + queryString;
        }

        sb.append(String.format("<a href='?pageIndex=%d%s'>&lt;&lt;</a>", 1, addQueryString));
        sb.append(String.format("<a href='?pageIndex=%d%s'>&lt;</a>", previousPageIndex, addQueryString));

        for(long i = startPage; i<= endPage; i++) {
            if (i == pageIndex) {
                sb.append(String.format("<a class='on' href='?pageIndex=%d%s'>%d</a>", i, addQueryString, i));
            } else {
                sb.append(String.format("<a href='?pageIndex=%d%s'>%d</a>", i, addQueryString, i));
            }
        }

        sb.append(String.format("<a href='?pageIndex=%d%s'>&gt;</a>", nextPageIndex, addQueryString));
        sb.append(String.format("<a href='?pageIndex=%d%s'>&gt;&gt;</a>", totalBlockCount, addQueryString));

        return sb.toString();
    }

    private void init() {
        if (pageIndex < 1) pageIndex = 1;
        if (pageSize < 1) pageSize = 1;

        totalBlockCount = totalCount / pageSize + (totalCount % pageSize > 0 ? 1 : 0);
        startPage = ((pageIndex - 1) / pageBlockSize) * pageBlockSize + 1;
        endPage = startPage + pageBlockSize - 1;

        if (endPage > totalBlockCount) {
            endPage = totalBlockCount;
        }
    }
}
```

---

## 📝 정리 요약

| 항목 | 내용 |
|------|------|
| 조회 방식 | MyBatis + XML Mapper |
| 검색 파라미터 | DTO (`MemberParam`) |
| 페이징 | PageUtil 클래스 |
| 페이징 구성 요소 | pageSize, pageIndex, totalCount, queryString |
| DTO vs Model vs Entity | 역할 구분 및 상황별 사용 기준 이해 필요 |

---

### 📌 부가 메서드 설명

#### ✅ `getPageStart()` – 시작 위치 계산

```java
public long getPageStart() {
    if (pageIndex < 1) {
        pageIndex = 1;
    }
    return (pageIndex - 1) * pageSize;
}
```

- 현재 페이지 인덱스 기준으로 SQL `LIMIT`의 시작값을 구함
- 예: 2페이지일 경우, 시작 위치는 `(2-1) * 10 = 10`

---

#### ✅ `getPageSize()` – 페이지당 데이터 수

```java
public long getPageSize() {
    return pageSize;
}
```

- 한 페이지당 출력할 데이터의 개수 (기본값: 10)

---

#### ✅ `setSeq()` – 리스트 항목에 순번 부여

```java
memberDto.setSeq(totalCount - parameter.getPageStart() - i);
```

- 역순 정렬 시 사용
- 예: 총 150명 중 2페이지(시작 10번째) 첫 번째 항목은 `150 - 10 - 0 = 140`

