---
title: "CRUD 게시판 만들기4 controller"
categories:
- "board project"
date: 2023-06-16
toc: true
toc_label: CATEGORY
toc_sticky: true
---


목표: CRUD 게시판의 controller을 구현하여 지금까지 만든 기능들을 합쳐 잘 동작되는지 확인한다.

# controller

이전 포스팅으로 Entity, Repository, view 파일까지 만들었다. 

이번 포스팅으로는 controller를 만들어 지금까지 만들었던 코드가 잘 구현되는지 확인할 것이다~

## controller에 사용한 애노테이션

postController에는 밑의 코드처럼 총 4가지의 애노테이션이 들어간다.

**postController.java**

```java
@Slf4j
@Controller
@RequestMapping("/posts")
@RequiredArgsConstructor
public class postController{
...
}
```

어떤 역할을 하는지 하나씩 확인해보자.

**@Slf4j**: log를 작성하게 하는 애노테이션이다. 

**@Controller**: spring been에 등록하는 애노테이션

 사실 기능으로는 @Component 와 다른게 없지만 개발자에게 controller파일이라 알려주는 역할도 있다.

**@RequestMapping(”/posts”)**: 개발자가 입력한 path가 서버에 들어오면 이 controller파일로 매핑시켜주는 애노테이션

**@RequiredArgsConstructor**: final선언한 인스턴스를 생성자 주입으로 의존성 주입을 해주는 애노테이션

위의 3개의 애노테이션은 이해했는데 마지막 **@RequiredArgsConstructor** 애노테이션은 잘 이해가되지 않아 동작을 확인하기 위해 postController.class파일 찾아보았다.

> postController.class 파일같이 확장자가 .class인 파일들은 자바 컴파일러가 .java파일을 읽고 컴파일하여 java bytecode로 작성된 파일이다.
JVM이 읽을 수 있는 java bytecode로 작성되는 과정을 거치기 때문에 자바코드는 OS에 상관없이 독립적으로 실행되는 환경을 가진다.
.class파일을 확인하면 몇몇 애노테이션의 동작을 확인할 수 있다.
> 

### postController.class

postController.java와 중복되는 코드는 최대한 제외하고 애노테이션때문에 생성된 코드만 가져왔다.

```java
// .java에서 가져온 코드
private final MapPostRepository mapPostRepository;

// @RequiredArgsConstructor에 의해 생성된 코드
public postController(MapPostRepository mapPostRepository) {
        this.mapPostRepository = mapPostRepository;
}
```

.java코드에서는 mapPostRepository 인스턴스가 final로 선언되었지만 .class파일에서 확인해보니

 mapPostRepository생성자가 만들어져 있다. 

스프링은 이 생성자를 참고하여 의존성 주입(spring been에 등록)을 한다. (생성자가 하나 있을 때는 @Autowired을 사용하지 않아도 된다!! )

그리고 이건 class파일을 찾아보면서 안 사실인데 @Slf4j 애노테이션을 사용하면 .class파일에

```java
private static final Logger log = LoggerFactory.getLogger(postController.class);
```

이런 코드가 생성된다. 

강의에서 LoggerFactory에서 대해서 들었지만 이렇게 직접 확인해보니 신기했다.

이 코드때문에 `log.info()` 같은 log를 마음껏 쓸 수 있는 것이다~

## post controller code

애노테이션을 포함한 controller 전체 코드이다.

```java
@Slf4j
@Controller
@RequestMapping("/posts")
@RequiredArgsConstructor
public class postController {
    private final MapPostRepository mapPostRepository;

    @GetMapping()
    public String posts(Model model) {
        List<Post> posts = mapPostRepository.findAll();
        model.addAttribute("posts", posts);
        return "posts/posts";
    }

    @GetMapping("/add")
    public String addForm(Model model) {
        model.addAttribute("post", new PostSaveForm());
        return "posts/addPost";
    }

    @PostMapping("/add")
    public String addPost(@ModelAttribute("post") @Validated PostSaveForm createPost, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "posts/addPost";
        }
        Post post = new Post(createPost.getWriter(), createPost.getTitle(), createPost.getContent());
        mapPostRepository.save(post);
        return "redirect:/posts";
    }

    @GetMapping("/{id}")
    public String post(@PathVariable Long id, Model model) {
        log.info("id={}", id);
        Post findPost = mapPostRepository.findById(id);
        mapPostRepository.updateViewCount(findPost);
        model.addAttribute("post", findPost);
        return "posts/post";
    }

    @GetMapping("/edit/{id}")
    public String editForm(@PathVariable Long id, Model model) {
        Post findPost = mapPostRepository.findById(id);
        model.addAttribute("post", findPost);
        return "posts/editPost";
    }

    @PostMapping("/edit/{id}")
    public String editPost(@PathVariable Long id, @Validated @ModelAttribute("post") PostUpdateForm form, BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            log.info("errors={}", bindingResult);
            return "posts/editPost";
        }
        Post post = new Post(form.getWriter(), form.getTitle(), form.getContent());
        mapPostRepository.update(id, post);
        return "posts/post";
    }

    @GetMapping("/delete/{id}")
    public String deletePost(@PathVariable Long id) {
        mapPostRepository.deletePost(id);
        return "redirect:/posts";
    }
}
```

`@GetMapping` 애노테이션을 쓰는 메서드는 크게 다른 점이 없다. postController에서는 모든 경로를 경로 변수로 받을 수 있게 설계하여 `@PathVariable` 애노테이션을 사용하여 받아 Repository의 메서드를 사용하여 처리한다.

`@PostMapping` 를 사용한 메서드는 검증에 필요한 `@Validated`와 `BindingResult` 를 추가했고, html form으로 들어오는 post값을 받기 위해 `@ModelAttribute` 애노테이션을 사용했다.

addPost와 editPost 메서드의 `@ModelAttribute`를 보면 받는 form을 다르게 만들었다.

addPost - postSaveForm

editPost - postEditForm

### postSaveForm

```java
@Data
public class PostSaveForm {
    @NotBlank(message = "작성자를 입력하세요.")
    private String writer;
    @NotBlank(message = "제목을 입력하세요.")
    private String title;
    @NotBlank(message = "내용을 입력하세요.")
    private String content;
}
```

### postEditForm

```java
@Data
public class PostUpdateForm {
    private Long id;
    @NotBlank(message = "작성자를 입력하세요.")
    private String writer;
    @NotBlank(message = "제목을 입력하세요.")
    private String title;
    @NotBlank(message = "내용을 입력하세요.")
    private String content;
}
```

사실 처음에 만들 때는 이렇게 Form을 만들지 않고 Entity객체인 Post를 사용하여 받았다.

하지만 다른 블로그나 강의에서 Entity로 받으면 절대~ 안된다고 했던게 생각나 빠르게 Form객체를 만들어 분리했다.

Entity가 아직 하나밖에 없는 작은 웹을 만들때에도 생성할 때와 수정할 때 받는 Form이 차이가 난다.(받을 때는 id가 필요가 없지만 수정할 때는 필요)

이제 이것보다 사이즈를 점점 키워갈텐데 앞으로는 만들때에는 처음부터 form을 고려하면서 개발해야겠다.

## end?

이번 controller를 끝으로 CRUD 게시판을 완성했다. 하지만 매우 간단하다보니 부족한 점이나 만들고 싶은 기능들이 생각난다.

controller파일에 controller기능만 있는게 아니고 service기능도 같이 포함되어 있다, entity가 하나밖에 없어 다른 entity와 관계를 만들고 싶어도 만들 수가 없다, 등등 부족한 부분을 내버려두는 것은 좋은 개발자가 되는 길이 아니다!! 다음 포스팅에서 간단한 후기와 앞으로 구현할 기능을 포스팅해보자!