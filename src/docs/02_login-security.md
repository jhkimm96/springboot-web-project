## 📁 02. Spring Security 로그인 처리 및 인증 흐름

### **✅ 개요**

Spring Security를 통해 로그인, 인증 실패, 패스워드 암호화, 이메일 인증 처리 등을 구성한다.

---

### 🔐 1. Spring Security 기본 동작

- 관련 라이브러리: `spring-boot-starter-security`
- 기본 실행 시 `http://localhost` 접속하면 Spring Security에서 제공하는 로그인 폼 자동 출력
- `application.yml` 또는 실행 로그에 생성된 `username`, `password`로 로그인 가능

---

### ⚙️ 2. Security Configuration 설정

### 📌 Security 설정 클래스 생성

```java
@Configuration // 해당 클래스가 설정 클래스임을 명시
@EnableWebSecurity // Spring Security를 활성화하는 어노테이션
@RequiredArgsConstructor // final 필드를 자동 생성자로 주입
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {
//WebSecurityConfigurerAdapter를 상속받아 사용자 맞춤 보안 설정 가능

    private final MemberService memberService;
    private final LoginFailHandler loginFailHandler;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
            .antMatchers("/**").permitAll() // 모든 페이지 접근 허용
            .and()
            .formLogin()
            .loginPage("/member/login")
            .failureHandler(loginFailHandler)
            .permitAll();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

- `Ctrl + O`: 오버라이드 가능한 메서드 목록 확인 가능
- `formLogin()`에 로그인 페이지 URL, 실패 핸들러 설정
- `PasswordEncoder`는 반드시 `@Bean`으로 등록

---

### 🚫 3. 로그인 실패 핸들러 구현

```java
@Component
public class LoginFailHandler extends SimpleUrlAuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response,
                                        AuthenticationException exception) throws IOException, ServletException {
        String errorMessage = "아이디 또는 비밀번호가 올바르지 않습니다.";

        if (exception instanceof InternalAuthenticationServiceException) {
            errorMessage = exception.getMessage();
        }

        setDefaultFailureUrl("/member/login?error=true&errorMessage=" + URLEncoder.encode(errorMessage, "UTF-8"));
        super.onAuthenticationFailure(request, response, exception);
    }
}

```

- 실패 시 에러 메시지를 쿼리스트링으로 전달
- `LoginFailHandler`는 `@Component`로 등록한 뒤, Security 설정 클래스에서 `@Bean` 또는 생성자로 주입

---

### 👤 4. MemberService에 UserDetailsService 구현

```java
public interface MemberService extends UserDetailsService {
    boolean register(MemberInput parameter);
}

```

### ✅ ServiceImpl에서 구현 (**`loadUserByUsername()` 오버라이드)**

```java
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    Member member = memberRepository.findById(username)
        .orElseThrow(() -> new MemberNotFoundException("존재하지 않는 회원입니다."));

    if (!member.isEmailAuthYn()) {
        throw new MemberNotEmailAuthException("이메일 인증이 완료되지 않았습니다.");
    }

    List<GrantedAuthority> authorities = List.of(new SimpleGrantedAuthority("ROLE_USER"));
    return new User(member.getUserId(), member.getPassword(), authorities);
}

```

- `UserDetailsService`를 구현하면 Spring Security가 사용자 정보를 이 서비스로 조회함
- 이메일 인증 여부 체크도 이곳에서 수행 가능
- 사용자 조회 로직 직접 구현
- `UserDetails` 객체는 Spring Security 내부 인증에 사용됨
- `GrantedAuthority`: 사용자 권한 (ex. ROLE_USER)

---

### 🔐 5. 로그인 화면 구현 (`/member/login`)

```html
<form method="post" action="/member/login">
  <input type="text" name="username" />
  <input type="password" name="password" />
  <button type="submit">로그인</button>
</form>

<!-- 로그인 실패 시 메시지 출력 -->
<p th:if="${param.error}" th:text="${param.errorMessage}"></p>
```

- 필드명은 반드시 `username`, `password` 사용해야 Spring Security가 인식 가능

---

### **🔒** 6. 회원가입 시 비밀번호 암호화

- 회원 저장 로직에서 암호화 적용:

```java
member.setPassword(passwordEncoder.encode(parameter.getPassword()));
```

- `SecurityConfiguration`의 `@Bean PasswordEncoder` 주입하여 사용

---

### 🚨 7. 이메일 인증 여부 예외 처리

- 인증되지 않은 사용자는 커스텀 예외 발생

```java
if (!member.isEmailAuthYn()) {
    throw new MemberNotEmailAuthException("이메일 인증이 필요합니다.");
}
```

- 인증이 완료되지 않은 회원은 `MemberNotEmailAuthException` 등으로 커스텀 예외 발생
- `exception` 패키지 생성 후 `RuntimeException` 상속받아 정의:

```java
public class MemberNotEmailAuthException extends RuntimeException {
    public MemberNotEmailAuthException(String message) {
        super(message);
    }
}
```

---

### ✅ 로그인 전후 테스트

1. 인증되지 않은 이메일로 로그인 시 → "이메일 인증이 필요합니다." 메시지
2. 인증 후 로그인 시 정상적으로 `ROLE_USER` 권한 부여
3. 로그인 실패 시 `LoginFailHandler`에서 메시지 전달

### ✅ 전체 인증 흐름 요약

```
A -->[브라우저 요청] --> B[/member/login 페이지 접속]
B --> C[사용자 입력: username/password]
C --> D[Spring Security가 Service 호출]
D --> E[loadUserByUsername() 실행]
E -->|존재하지 않음| F[예외 발생 → 실패 핸들러]
E -->|이메일 미인증| F
E -->|정상 로그인| G[SecurityContext에 인증정보 저장]
G --> H[다음 요청부터 인증된 사용자 처리]