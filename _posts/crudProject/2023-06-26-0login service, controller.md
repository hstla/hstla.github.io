---
title: "게시판 로그인 기능3 - service, controller 및 정리"
categories:
- "board project"
date: 2023-06-26
toc: true
toc_label: CATEGORY
toc_sticky: true
---


저번 포스팅에서 만든 member repository를 사용하여 service, controller를 구현하자.

## service

### MemberService code

```java
@Service
@RequiredArgsConstructor
public class MemberService {
    private final MapMemberRepository mapMemberRepository = new MapMemberRepository();

    /**
     * 로그인 성공 시 return member
     * 로그인 실패 시 return null
     */
    public Member login(String loginEmail, String loginPassword) {
        return mapMemberRepository.findByEmail(loginEmail)
                .filter(m -> m.getPassword().equals(loginPassword)).orElse(null);
    }

    public Member join(Member member) {
        Member saveMember = Member.builder()
                .email(member.getEmail())
                .password(member.getPassword())
                .writer(member.getWriter())
                .build();
        Member joinMember = mapMemberRepository.save(saveMember);
        return joinMember;
    }

    public void withdrawal(Long memberId) {
        mapMemberRepository.deleteMember(memberId);
    }
}
```

service 에서는 로그인, 회원 가입, 회원 탈퇴를 구현했다.

로그인 메서드는 로그인에 필요한 email과 password를 가져와 람다식을 이용해 member 객체를 찾는다. 만약 비밀번호가 다르거나 email이 없으면 null을 반환한다. 처음에는 로그인에 필요한 매개변수는 LoginForm으로 받으려고 생각했지만, domain폴더가 web폴더를 참조하면 각 계층의 독립성이 깨지기때문에 안 좋아 email과 password로 받기로 했다.

회원가입 메서드는 매개변수인 멤버객체에서 email, password, writer속성만 가져와 map에 저장하고 회원탈퇴 메서드는 memberId를 받아 삭제 시켰다.

## controller

Controller는 login기능을 수행하는 LoginController와 user page나 멤버 수정기능을 수행하는 MemberController로 나뉜다.

### LoginController code

```java
@Slf4j
@Controller
@RequiredArgsConstructor
public class LoginController {
    private final MemberService memberService;

    @GetMapping("/login")
    public String loginForm(@ModelAttribute("form") MemberLoginForm form) {
        return "login/loginForm";
    }

    @PostMapping("/login")
    public String login(@ModelAttribute("form") MemberLoginForm form, HttpServletRequest request) {
        Member loginMember = memberService.login(form.getEmail(), form.getPassword());
        if (loginMember != null) {
            HttpSession session = request.getSession();
            session.setAttribute("loginMember", loginMember);
            log.info("login={}", loginMember.getId());
            return "redirect:/posts";
        }

        return "/login/loginForm";
    }

		@GetMapping("/logout")
    public String logout(HttpServletRequest request) {
        // delete session
        HttpSession session = request.getSession(false);
        if (session != null) session.invalidate();

        String referer = request.getHeader("Referer");
        return "redirect:" + (referer != null ? referer : "/");
    }

    @GetMapping("/join")
    public String joinForm(@ModelAttribute("form") MemberJoinForm form) {
        return "login/joinForm";
    }

    @PostMapping("/join")
    public String join(@Validated @ModelAttribute("form") MemberJoinForm form, BindingResult bindingResult) {
        if (!form.getPassword().equals(form.getConfirmPassword())) {
            bindingResult.rejectValue("confirmPassword", "notSamePassword", "비밀번호가 다릅니다.");
        }

        if (bindingResult.hasErrors()) {
            return "login/joinForm";
        }

        memberService.join(form.FormTOMember());

        return "redirect:/login";
    
		@GetMapping("/withdrawal/{id}")
    public String memberWithdrawal(@PathVariable Long id, HttpServletRequest request) {
        memberService.withdrawal(id);

        HttpSession session = request.getSession(false);
        if (session != null) session.invalidate();

        return "redirect:/posts";
    }
}
```

로그인 기능을 위한 controller 코드이다.

로그인이 성공하면 `request.getSession()`을 사용하여 세션을 가져온 후 `session.setAttribute("loginMember", loginMember)` 으로 세션 이름과 멤버 객체를 반환한다.

만약 입력받은 email, password로 로그인에 실패하면 다시 로그인페이지로 돌아간다. 

로그아웃은 로그인 했을 때 보냈던 세션을 종료시키면 된다. 

 로그아웃 코드 중 `request.getSession(false)` 이 코드는 로그인의 세션을 가져오는 코드와 비슷하지만, false 매개변수를 넣었다. 

getSession()에 false이 있으면 현재 요청에 세션이 있으면 가져오지만, 없으면 세션을 생성하지 않고 null을 반환한다는 의미이다.

세션이 없으면 null을 반환하기때문에 이후 코드에서 If문으로 세션을 유무를 확인하고 세션이 있으면 `session.invalidate()` 으로 세션을 무효화 한다.

회원탈퇴 메서드에서도 로그아웃과 같은 코드로 세션을 삭제하고 Map에서 멤버를 삭제한다.

### MemberController code

```java
@Slf4j
@Controller
@RequestMapping("/member")
@RequiredArgsConstructor
public class MemberController {
    private final MemberService memberService;
    private final MapMemberRepository mapMemberRepository;

    @GetMapping("/{id}")
    public String user(@PathVariable Long id, Model model) {
        Member findMember = mapMemberRepository.findById(id);
        model.addAttribute("form", findMember);
        return "member/member";
    }

    @GetMapping("/edit/{id}")
    public String updateMemberForm(@PathVariable Long id, Model model) {
        Member findMember = mapMemberRepository.findById(id);
        model.addAttribute("form", findMember);
        return "member/editForm";
    }

    @PostMapping("/edit/{id}")
    public String updateMember(@PathVariable Long id, @Validated @ModelAttribute("form") MemberUpdateForm memberUpdateForm, BindingResult bindingResult) {
        mapMemberRepository.updateMember(id, memberUpdateForm.FormTOMember());
        return "redirect:/member/{id}";
    }
}
```

멤버 수정, 및 멤버 페이지를 볼 수 있는 controller이다. 

`MemberUpdateForm`으로 멤버 수정에 필요한 값을 받고 업데이트 후 다시 멤버 페이지로 redirect하였다.

## 정리

member entity, service, controller, view까지 코딩하여 세션을 이용한 로그인기능을 만들었다.

처음 사용해보는 Optional클래스를 정리하느라 시간이 오래걸렸고, 아직 에러처리나 검증단계가 많이 부실하다.

member의 속성인 writer와 post의 writer로 연결하고 싶지만 지금 Map를 따로 쓰고 있어 많이 복잡해질 것같아 다음에 추가할 JPA로 member entity와 post entity를 연결한 후 진행하려 한다.