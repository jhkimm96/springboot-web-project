## 📁 0. 개발 환경 & 필수 단축키

### 📌 인텔리제이 단축키 모음

| 기능 | 단축키 |
| --- | --- |
| 사용하지 않는 import 제거 | `Ctrl + Alt + O` |
| 선언된 메서드/클래스로 이동 | `Ctrl + 클릭` |
| 코드 포맷팅 | `Ctrl + Alt + L` |
| 전체 찾기 | `Shift + Shift` |
| 실행 | `Shift + F10` |
| 전체 리팩토링 | `Ctrl + Alt + Shift + T` |

---

## 📁 01. Spring Controller & HTML 연동 기본 이해

### 🧠 핵심 개념

- `@Controller`와 `@RequestMapping`은 브라우저 URL과 자바 코드를 연결하는 핵심 어노테이션.
- HTML 렌더링은 `Thymeleaf` 같은 템플릿 엔진으로 처리.

### 📌 주소 매핑 & View 처리

- `return "index"` → `resources/templates/index.html` 매핑됨
- 설정은 `application.yml`에서 `thymeleaf.prefix`, `suffix`로 변경 가능 > 설정 없어도 default 로 되어있음

---

### 📁 2. HTML 파일 vs 텍스트 파일 브라우저 출력 비교

- `index.html` : 브라우저에서 구조적으로 렌더링
- `index.txt` : 텍스트 그대로 보여짐
    
    ➡️ 브라우저는 확장자에 따라 렌더링 방식을 달리함
    

---

### 📁 3. 컨트롤러와 Response 객체 직접 사용

### ✅ 핵심 코드 예시

```java
response.setContentType("text/html;charset=UTF-8");
PrintWriter out = response.getWriter();
out.print("<h1>Hello!</h1>");
```

➡️ 이런 방식은 비효율적이라 Thymeleaf 사용을 권장

---

### 📁 4. Thymeleaf 설정과 기본 사용법

### 📌 기본 설정

```yaml

spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
```

### 📌 사용 방식

- `@GetMapping("/")` → `return "index"` 시 `templates/index.html` 렌더링(GET/POST)

---

## 📁 5. 회원가입 입력폼 처리 (GET/POST)

### 🧱 구성 요소

- `@GetMapping("/member/register")` : 입력 폼 페이지 렌더링
- `@PostMapping("/member/register")` : 서버로 데이터 전송

### 📌 form 전송 주의사항

- `form` 태그 내부의 `name` 속성이 있어야 서버에서 값 수신 가능
- DTO를 `@ModelAttribute` 또는 `@RequestParam`으로 매핑
- DTO에는 `@Data`, `@ToString` 사용 가능 (Lombok)

---

## 📁 6. JPA를 통한 DB 저장

### ✅ 설정 순서

1. `spring-boot-starter-data-jpa`, `mysql-connector-j` 의존성 추가
2. `application.yml`에 datasource, JPA 설정 추가
3. Entity 클래스 생성 (`@Entity`, `@Id`, `@Builder` 등)
4. `JpaRepository<Entity, ID>` 상속한 Repository 생성
5. `@Service`, `@RequiredArgsConstructor`로 Service 구성
6. Controller → Service → Repository 흐름 구성

### 📌 주의사항

- `save()`는 동일한 ID 존재 시 update로 동작 → 사전 중복 체크 필요
- `findById().isPresent()`로 Optional 체크 후 예외 처리 필요

---

### ✅ 회원가입 중복 체크 및 저장 예시

```java
@Override
public boolean register(MemberInput parameter) {
    Optional<Member> optionalMember = memberRepository.findById(parameter.getUserId());
    if (optionalMember.isPresent()) {
        throw new RuntimeException("이미 존재하는 아이디입니다.");
    }

    Member member = Member.builder()
            .userId(parameter.getUserId())
            .userName(parameter.getUserName())
            .password(parameter.getPassword())
            .phone(parameter.getPhone())
            .email(parameter.getEmail())
            .emailAuthYn(false)
            .emailAuthKey(UUID.randomUUID().toString())
            .build();

    memberRepository.save(member);
    return true;
}
```

---

## 📁 7. JavaMailSender로 이메일 전송

### ✅ 준비사항

- Gmail → 앱 비밀번호 생성
- `application.yml`에 메일 전송 설정

```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: your@email.com
    password: your_app_password
    properties:
      mail:
        smtp:
          starttls:
            enable: true
```

### ✅ 메일 전송 코드 예시

```java
SimpleMailMessage message = new SimpleMailMessage();
message.setTo(email);
message.setSubject("회원가입 인증");
message.setText("인증 링크: ...");
mailSender.send(message);
```

---

## 📁 8. 회원가입 후 인증메일 전송

### 🧱 로직 구성

- Entity에 `emailAuthKey`, `emailAuthYn` 컬럼 추가
- 회원가입 시 UUID 생성 후 DB 저장
- MailComponent 통해 인증 URL 포함한 메일 전송

```java
mailComponent.sendMail(
    parameter.getEmail(),
    "회원가입 인증",
    "인증 링크: http://localhost:8080/member/email-auth?uuid=" + uuid
);
```

---

## 📁 9. 이메일 인증 처리 및 계정 활성화

### ✅ 인증 처리 흐름

1. `/member/email-auth?uuid=...` 요청 수신
2. Service에서 `findByEmailAuthKey()`로 사용자 조회
3. 인증 성공 시 `emailAuthYn = true`, `emailAuthDt = LocalDateTime.now()` 설정
4. `memberRepository.save()`로 상태 갱신

---

## 📁 10. Entity를 빌더패턴으로 만드는 이유

### 🔍 이유

- 필드별 `set()` 호출 제거
- 생성자 파라미터 실수 방지
- 가독성 및 유지보수 용이

### ✅ 예시

```java
Member member = Member.builder()
    .userId("test")
    .userName("홍길동")
    .password("1234")
    .build();
```
