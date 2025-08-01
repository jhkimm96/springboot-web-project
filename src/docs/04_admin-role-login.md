## 📁 04. 관리자 로그인 기능

### ✅ 개요

Spring Security를 활용해 특정 사용자에게만 관리자 권한을 부여하고,

관리자만 접근 가능한 URL 및 화면을 구성합니다.

---

### 🧠 1. 권한 설계 방향 고려

- 관리자 전용 테이블을 만들 것인가?
- `Member` 엔티티 내 권한 필드(`role`, `adminYn`, `authYn`)를 둘 것인가?
- 예: `ROLE_USER`, `ROLE_ADMIN`, `ROLE_SPECIAL` 등

> 예제에선 adminYn 또는 authYn 으로 간단하게 구현
>

---

### 🏗 2. Member Entity에 adminYn 필드 추가

```java
@Entity
@Data
public class Member {
    ...
    private boolean adminYn; // true = 관리자
}
```

DB에 직접 `adminYn` 값을 true 로 설정해서 관리자 여부 테스트 가능

---

### 👤 3. 로그인 시 권한 부여 로직 (UserDetailsService)

```java
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    Member member = memberRepository.findById(username)
        .orElseThrow(() -> new UsernameNotFoundException("존재하지 않는 사용자"));

    List<GrantedAuthority> authorities = new ArrayList<>();
    authorities.add(new SimpleGrantedAuthority("ROLE_USER"));

    if (member.isAdminYn()) {
        authorities.add(new SimpleGrantedAuthority("ROLE_ADMIN"));
    }

    return new User(member.getUserId(), member.getPassword(), authorities);
}
```

> loadUserByUsername()은 UserDetailsService 인터페이스의 추상 메서드로, Spring Security가 자동 호출
>

---

### 🔐 4. Security 설정에 ROLE_ADMIN 접근 제한

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .authorizeRequests()
        .antMatchers("/admin/**").hasAuthority("ROLE_ADMIN") // 관리자만 접근 가능
        ...
        .and()
        .exceptionHandling()
        .accessDeniedPage("/error/denied"); // 권한 없는 경우 이동
}
```

---

### ⚠️ 5. 접근 권한 없을 때 보여줄 에러 페이지

### 🔁 컨트롤러 작성

```java
@Controller
public class ErrorController {

    @GetMapping("/error/denied")
    public String denied() {
        return "error/denied";
    }
}
```

### 📄 templates/error/denied.html

```html
<h2>접근 권한이 없습니다.</h2>
<a href="/">홈으로 이동</a>
```

---

### 🔎 전체 흐름 요약

| 단계 | 설명 |
| --- | --- |
| Member에 adminYn 필드 추가 | 권한 구분 플래그 |
| 로그인 시 ROLE_ADMIN 권한 포함 여부 판단 | `loadUserByUsername()` |
| Security 설정에서 `/admin/**` 권한 제한 | `hasAuthority("ROLE_ADMIN")` |
| 접근 실패 시 `/error/denied` 페이지로 이동 | `exceptionHandling().accessDeniedPage(...)` |

---

### 📌 기타 설명

- `ROLE_` 접두사는 Spring Security에서 권장되는 권한 네이밍 규칙
- `GrantedAuthority`: Spring Security에서 사용자 권한을 표현하는 인터페이스
- `SimpleGrantedAuthority`: 문자열 권한을 래핑해주는 클래스

---

### 📁 관련 파일 구조 예시

```
src/
 └── main/java/com/example/
     ├── config/SecurityConfiguration.java
     ├── controller/ErrorController.java
     ├── member/entity/Member.java
     ├── member/service/MemberServiceImpl.java
 └── resources/templates/
     └── error/denied.html
```