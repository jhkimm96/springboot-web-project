## 📁 03. 비밀번호찾기(초기화) 및 새 비밀번호 링크 이메일 전송

### ✅ 개요

사용자가 비밀번호를 분실했을 때, 본인 인증을 통해 임시 링크로 비밀번호를 초기화할 수 있는 기능입니다.

> 해시된 비밀번호는 복호화가 불가능하기 때문에 "재설정" 방식이 필요합니다.
>

---

### 📌 기능 흐름 요약

1. 사용자 ID + 이름으로 본인 확인
2. 이메일로 초기화 링크 전송 (UUID 포함)
3. 사용자 클릭 시 reset form 렌더링
4. 새 비밀번호 입력 → 암호화 후 저장
5. 재사용 불가하도록 UUID 및 만료 시간 무효화

---

### ✅ 비로그인 상태에서 허용되는 경로 설정

Security 설정 파일 (`SecurityConfiguration.java`)에 다음과 같이 경로를 허용:

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .authorizeRequests()
        .antMatchers("/", "/member/login", "/member/register", "/member/find-password", "/member/reset-password")
        .permitAll()
        ...
}
```

---

### ✅ 비밀번호 초기화 요청

```java
@PostMapping("/member/find-password")
public String findPasswordSubmit(Model model, MemberInput parameter) {
    boolean result = memberService.sendResetPassword(parameter);
    model.addAttribute("result", result);
    return "member/find_password_result";
}
```

### 🔍 어노테이션 설명

- `@PostMapping`: 해당 경로에 POST 요청 처리
- `@ModelAttribute` 또는 DTO 파라미터: 입력값 자동 바인딩

---

### ✅ `MemberService` 인터페이스 정의

```java
boolean sendResetPassword(MemberInput parameter);
```

---

### ✅ `MemberServiceImpl` 구현

```java
@Override
public boolean sendResetPassword(MemberInput parameter) {
    Optional<Member> optionalMember =
        memberRepository.findByUserIdAndUserName(parameter.getUserId(), parameter.getUserName());

    if (!optionalMember.isPresent()) {
        return false;
    }

    Member member = optionalMember.get();
    String uuid = UUID.randomUUID().toString();

    member.setResetPasswordKey(uuid);
    member.setResetPasswordLimitDt(LocalDateTime.now().plusDays(1));
    memberRepository.save(member);

    String resetUrl = "http://localhost:8080/member/reset-password?id=" + uuid;
    String emailText = "<p>비밀번호 초기화를 위해 아래 링크를 클릭해주세요.</p>" +
                       "<p><a href='" + resetUrl + "'>비밀번호 초기화</a></p>";

    mailComponent.sendMail(member.getEmail(), "비밀번호 초기화 안내", emailText);
    return true;
}
```

### 🔍 주요 설명

- `UUID`: 고유한 링크를 생성하는 용도
- `resetPasswordLimitDt`: 유효기간 설정
- `mailComponent`: JavaMailSender를 주입받아 이메일 발송

---

### ✅ Repository에 메서드 자동 생성

```java
Optional<Member> findByUserIdAndUserName(String userId, String userName);
Optional<Member> findByResetPasswordKey(String resetKey);
```

### 🔍 왜 자동 생성 가능한가?

JPA는 메서드명을 기반으로 쿼리를 자동 생성합니다.

findBy에서 tab + tab 치면 원하는 컬럼명 추가로 넣을수있다.

예: `findByUserIdAndUserName()` → WHERE user_id = ? AND user_name = ?

---

### ✅ 비밀번호 초기화 링크 확인

```java
@GetMapping("/member/reset-password")
public String resetPassword(Model model, @RequestParam String id) {
    boolean valid = memberService.checkResetPassword(id);
    model.addAttribute("result", valid);
    model.addAttribute("id", id);
    return "member/reset_password";
}
```

```java
@Override
public boolean checkResetPassword(String uuid) {
    Optional<Member> optionalMember = memberRepository.findByResetPasswordKey(uuid);

    if (!optionalMember.isPresent()) return false;

    Member member = optionalMember.get();
    if (member.getResetPasswordLimitDt() == null || member.getResetPasswordLimitDt().isBefore(LocalDateTime.now())) {
        return false;
    }

    return true;
}
```

---

### ✅ 비밀번호 재설정 처리

```java
@PostMapping("/member/reset-password")
public String resetPasswordSubmit(Model model, MemberInput parameter) {
    boolean result = memberService.resetPassword(parameter.getId(), parameter.getPassword());
    model.addAttribute("result", result);
    return "member/reset_password_result";
}
```

```java
@Override
public boolean resetPassword(String uuid, String password) {
    Optional<Member> optionalMember = memberRepository.findByResetPasswordKey(uuid);

    if (!optionalMember.isPresent()) throw new MemberNotFoundException("회원 정보가 없습니다.");

    Member member = optionalMember.get();

    if (member.getResetPasswordLimitDt() == null || member.getResetPasswordLimitDt().isBefore(LocalDateTime.now())) {
        throw new RuntimeException("유효 기간이 만료되었습니다.");
    }

    String encryptedPassword = passwordEncoder.encode(password);
    member.setPassword(encryptedPassword);
    member.setResetPasswordKey(null);
    member.setResetPasswordLimitDt(null);
    memberRepository.save(member);

    return true;
}
```

---

### ✅ 유효성 검사 (클라이언트)

```html
<script>
  function checkPasswordMatch() {
    const pw1 = document.getElementById('password').value;
    const pw2 = document.getElementById('password2').value;
    document.getElementById('submitBtn').disabled = pw1 !== pw2;
  }
</script>
```

---

### 📌 예외 처리 클래스 (선택)

```java
public class MemberNotFoundException extends RuntimeException {
    public MemberNotFoundException(String message) {
        super(message);
    }
}
```

> 📁 exception/ 폴더에 따로 관리 추천
>

---

### ✅ 정리

| 항목 | 설명 |
| --- | --- |
| UUID | 각 링크 고유값 |
| 유효기간 | 24시간 제한 |
| resetPasswordKey | 사용 후 무효화 |
| 사용자 확인 | ID + 이름으로 식별 |
| 예외처리 | 존재하지 않거나 만료된 링크 차단 |