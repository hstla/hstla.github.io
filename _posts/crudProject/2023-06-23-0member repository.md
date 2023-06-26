---
title: "게시판 로그인 기능 - repository"
categories:
- "board project"
date: 2023-06-23
toc: true
toc_label: CATEGORY
toc_sticky: true
---

지난 포스팅에서 로그인 기능을 위해 요구사항을 분석과 member entity를 만들었다.

이번 시간에는 이어서 member repository를 만들어 보자.

# member repository

기능구현을 하기 전 일관성을 유지하기 위해 interface를 만든다.

### interface MemberRepository code

```java
public interface MemberRepository {
    Member save(Member member);
    Member findById(Long id);
    List<Member> findAll();
    void updateMember(Long id, Member upadteMember);
    void clearStore();
    void deleteMember(Long memberId);

    Optional<Member> findByEmail(String email);
}
```

전에 만든 post repository와 비슷하다. 

테스트 코드를 위해 `clearStore` 를 만들어 전체 멤버를 지울 수 있게 한다.

email을 id로 사용하기 때문에 `findByEmail` 를 만들어 사용자가 입력한 email로 member를 찾을 수 있게 한다.

member가 없으면 null처리를 할 수 있게 Optional 래퍼 클래스를 사용한다.

Optional 클래스 사용방법은 이전 포스팅으로 정리했다. [<span style="color:#3366BB">optional 사용법</span>](https://hstla.github.io/java/0%EC%9E%90%EB%B0%94%EC%9D%98-Optional-%EC%82%AC%EC%9A%A9%EB%B2%95/)

### MapMemberRepository code

```java
@Slf4j
@Repository
public class MapMemberRepository implements MemberRepository {

    private static final Map<Long, Member> store = new ConcurrentHashMap<>();

    private static AtomicLong sequence = new AtomicLong(0);

    @Override
    public Member save(Member member) {
        LocalDateTime now = LocalDateTime.now();
        Long memberId = sequence.getAndIncrement();
        Member newMember = member.builder()
                .id(memberId)
                .email(member.getEmail())
                .password(member.getPassword())
                .writer(member.getWriter())
                .createdDate(now)
                .modifiedDate(now)
                .build();
        store.put(memberId, newMember);
        return newMember;
    }

    @Override
    public Member findById(Long id) {
        return store.get(id);
    }

    @Override
    public List<Member> findAll() {
        return new LinkedList<>(store.values());
    }

    @Override
    public void updateMember(Long id, Member upadteMember) {
        Member findMember = findById(id);
        LocalDateTime now = LocalDateTime.now();
        findMember.updateWriter(upadteMember.getWriter());
        findMember.updatePassword(upadteMember.getPassword());
        findMember.updateEmail(upadteMember.getEmail());
        findMember.updateModifiedDate(now);

    }

    @Override
    public void clearStore() {
        store.clear();
    }

    @Override
    public void deleteMember(Long memberId) {
        store.remove(memberId);
    }

    @Override
    public Optional<Member> findByEmail(String email) {
        return Optional.ofNullable(store.values().stream()
                .filter(member -> member.getEmail().equals(email))
                .findFirst()
                .orElse(null));
    }
}
```

MemberRepository interface를 받아 기능을 구현한다.

주의 깊게 봐야 할 부분은 `save`, `findByEmail` 메서드이다.

`save`메서드에서 setter대신 빌더패턴을 사용하여 member객체를 생성했다. 

setter대신 builder를 처음 사용할 때는 헷갈리는 점도 많고 update할 때 어떻게 할지 고민이 많았지만 builder를 익히고 사용해보니 코드 가독성과 객체 생성을 제어할 수 있어 좋다.

`findByEmail` 메서드에서는 자바 8에서 추가된 `stream` 을 사용하여 id가 아닌 email로 member을 빠르게 찾을 수 있게 한다. email은 중복없이 설계했기 때문에 `findFirst()` 을 사용하여 찾은 member을 바로 찾을  수 있게한다.

### MapMemberRepositoryTest code

```java
@Slf4j
class MapMemberRepositoryTest {

    MapMemberRepository mapMemberRepository = new MapMemberRepository();
    static Member member1;
    static Member member2;

    @BeforeEach
    void firstSave() {
        member1 = Member.builder()
                .email("test1@gmail.com")
                .password("1234")
                .writer("writer1")
                .build();

        member2 = Member.builder()
                .email("test2@gmail.com")
                .password("1234")
                .writer("writer2")
                .build();
    }

    @AfterEach
    void clearStore() {
        mapMemberRepository.clearStore();
    }

    @Test
    void save() {
        //when
        Member saveMember1 = mapMemberRepository.save(member1);
        Member saveMember2 = mapMemberRepository.save(member2);

        //then
        assertThat(saveMember1.getId()).isEqualTo(0L);
        assertThat(saveMember1.getWriter()).isEqualTo("writer1");
        assertThat(saveMember2.getId()).isEqualTo(1L);
        assertThat(saveMember2.getWriter()).isEqualTo("writer2");
    }

    @Test
    void findById() {
        //given
        Member saveMember = mapMemberRepository.save(member1);
        //when
        Member findMember = mapMemberRepository.findById(saveMember.getId());
        //then
        assertThat(saveMember).isEqualTo(findMember);
        log.info("saveMemberId={}", saveMember.getId());
        log.info("findMemberId={}", findMember.getId());
    }

    @Test
    void findAll() {
        //given
        mapMemberRepository.save(member1);
        mapMemberRepository.save(member2);
        //when
        List<Member> allMembers = mapMemberRepository.findAll();
        //then
        assertThat(allMembers.size()).isEqualTo(2);
    }

    @Test
    void updateMember() {
        //given
        Member saveMember = mapMemberRepository.save(member1);
        //when
        Member updateMember = Member.builder()
                .writer("updateWriter")
                .password("updatePass")
                .build();

        mapMemberRepository.updateMember(saveMember.getId(), updateMember);
        Member findMember = mapMemberRepository.findById(saveMember.getId());
        //then
        assertThat(findMember.getWriter()).isEqualTo("updateWriter");
        assertThat(findMember.getPassword()).isEqualTo("updatePass");
        log.info("createDate={}", findMember.getCreatedDate());
        log.info("ModifiedDate={}", findMember.getModifiedDate());
    }

    @Test
    void deleteMember() {
        //given
        Member saveMember1 = mapMemberRepository.save(member1);
        mapMemberRepository.save(member2);
        //when
        mapMemberRepository.deleteMember(saveMember1.getId());
        List<Member> findList = mapMemberRepository.findAll();
        //then
        assertThat(findList.size()).isEqualTo(1);
    }

    @Test
    void clearStore1() {
        //given
        mapMemberRepository.save(member1);
        //when
        mapMemberRepository.clearStore();
        List<Member> allMembers = mapMemberRepository.findAll();
        //then
        assertThat(allMembers.size()).isEqualTo(0);
    }

    @Test
    void findByEmail() {
        //given
        mapMemberRepository.save(member1);
        mapMemberRepository.save(member2);
        //when
        Member findEmailMember = mapMemberRepository.findByEmail("test1@gmail.com").orElse(null);
        //then
        assertThat(findEmailMember.getEmail()).isEqualTo("test1@gmail.com");
        log.info("findEmailMember.getEmail={}", findEmailMember.getId());
    }
}
```

MapMemberRepository를 바탕으로 테스트 코드를 만들었다.

멤버를 생성하는데 코드가 너무 중복되는 것같아 @BeforeEach와 @AfterEach애노테이션을 사용하여 코드를 간단하게 줄였다.

이번 포스팅에서는 login에 필요한 repository 기능을 만들어 보았다. 

다음부터 repository  기능을 사용하여 본격적으로 login, join 기능을 만들어 보자.