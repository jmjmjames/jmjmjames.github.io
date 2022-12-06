---
layout: post
title: "[Spring JPA] Querydsl을 통한 게시판 검색 구현"
subtitle: "Querydsl 활용하기"
categories : Spring Spring_JPA
tags : [Spring JPA, Querydsl]
date: 2022-12-06 17:00:00 +0900
---

> 게시판 해시태그 검색에서 해시태그 전체 조회를 할 때 Querydsl 을 적용하려한다.

<br>

##### ArticleServiceTest.java

```java
@DisplayName("비즈니스 로직 - 게시글")
@ExtendWith(MockitoExtension.class)
class ArticleServiceTest {

    // sut: system under test
    // @InjectMocks 은 온전히 생성자 주입은 어려움 Target: Field
    @InjectMocks
    private ArticleService sut;

    @Mock
    private ArticleRepository articleRepository;
    @Mock
    private UserRepository userRepository;
    
    @DisplayName("검색어 없이 해시태그를 통해 게시글을 조회하면, 빈 페이지를 반환한다.")
    @Test
    void givenNoSearchParameters_whenSearchingArticlesViaHashtag_thenReturnsEmptyPage() {
        // Given
        Pageable pageable = Pageable.ofSize(20);

        // When
        Page<ArticleDto> articles = sut.searchArticlesViaHashTag(null, pageable);

        // Then
        assertThat(articles).isEqualTo(Page.empty(pageable));
        then(articleRepository).shouldHaveNoInteractions();
    }

    @DisplayName("검색어 없이 해시태그를 통해 게시글을 검색하면, 게시글 페이지를 반환한다.")
    @Test
    void givenHashtag_whenSearchingArticlesViaHashtag_thenReturnsArticlePage() {
        // Given
        String hashtag = "#spring";
        Pageable pageable = Pageable.ofSize(20);
        given(articleRepository.findByHashtag(hashtag, pageable)).willReturn(Page.empty(pageable));

        // When
        Page<ArticleDto> articles = sut.searchArticlesViaHashTag(hashtag, pageable);

        // Then
        assertThat(articles).isEqualTo(Page.empty(pageable));
        then(articleRepository).should().findByHashtag(hashtag, pageable);
    }
    
    @DisplayName("해시태그를 조회하면, unique 해시태그 리스트를 반환한다.")
    @Test
    void givenNothing_whenCalling_thenReturnsHashTags() {
        // Given
        List<String> expectedHashtags = List.of("#spring", "#java", "#jpa", "#코딩테스트");
        given(articleRepository.findAllDistinctHashtags()).willReturn(expectedHashtags);

        // When
        List<String> actual = sut.getHashtags();

        // Then
        assertThat(actual).isEqualTo(expectedHashtags);
        then(articleRepository).should().findAllDistinctHashtags();
    }

    private Article createArticle() {
        Article article = Article.of(
                createUser(),
                "title",
                "content",
                "hashtag"
        );
        ReflectionTestUtils.setField(article, "id", 1L);

        return article;
    }

    private ArticleDto createArticleDto() {
        return createArticleDto("title", "content", "#spring");
    }

    private ArticleDto createArticleDto(String title, String content, String hashtag) {
        return ArticleDto.of(
                1L,
                createUserDto(),
                title,
                content,
                hashtag,
                LocalDateTime.now(),
                "Jm",
                LocalDateTime.now(),
                "Jm");
    }

    private UserDto createUserDto() {
        return UserDto.of(
                1L,
                "jm@email.com",
                "password",
                "Jm",
                "This is Memo",
                LocalDateTime.now(),
                "jm",
                LocalDateTime.now(),
                "jm"
        );
    }

    private User createUser() {
        return User.of(
                "jm@email.com",
                "password",
                "Jm",
                null
        );
    }   
}
```

##### Querydsl 적용해야할 부분
```java
@DisplayName("해시태그를 조회하면, unique 해시태그 리스트를 반환한다.")
@Test
void givenNothing_whenCalling_thenReturnsHashTags() {
    // Given
    List<String> expectedHashtags = List.of("#spring", "#java", "#jpa", "코딩테스트");
    given(articleRepository.findAllDistinctHashtags()).willReturn(expectedHashtags);

    // When
    List<String> actual = sut.getHashtags();

    // Then
    assertThat(actual).isEqualTo(expectedHashtags);
    then(articleRepository).should().findAllDistinctHashtags();
}
```

- `given(articleRepository.findAllDistinctHashtags()).willReturn(expectedHashtags);`: <br> Query의 출력 결과과 도메인이 아닌 **List\<String>** 모양이다.

- 기본적으로 도메인 단위로 출력 단위이기에 **Querydsl** 도입이 필요하다.

<br><br>

## 구현 ❗️❗️

---

### 패키지 구조

---

<img width="500" src="https://user-images.githubusercontent.com/74996516/205855587-aa90452a-9bfe-4dfc-8cf2-6bc00f89646c.png">

<br>

### Querydsl 구현하기 ❗️


##### ArticleRepositoryCustom

---

```java
public interface ArticleRepositoryCustom {
    List<String> findAllDistinctHashtags();
}
```

##### ArticleRepositoryCustomImpl

---

```java
public class ArticleRepositoryCustomImpl extends QuerydslRepositorySupport implements ArticleRepositoryCustom {

    public ArticleRepositoryCustomImpl() {
        super(Article.class);
    }

    @Override
    public List<String> findAllDistinctHashtags() {
        QArticle article = QArticle.article;

        /*
          제네릭 타입을 꼭 넣어주자!
          JPQLQuery<String> query = from(article)
                          .distinct()
                          .select(article.hashtag)
                          .where(article.hashtag.isNotNull());

                  return query.fetch();
         */
        return from(article)
                .distinct()
                .select(article.hashtag)
                .where(article.hashtag.isNotNull())
                .fetch();
    }
}
```
- ArticleRepositoryCustom : ArticleRepositoryCustom이는 이름의 제약이 존재하지 않는다. <br> 예) ArticleRepositoryMy

- ArticleRepositoryCustomImpl: **이름에 Impl을 붙히고 구현을 해야한다. 이는 그렇지 않으면 설정 부분을 건드려야 하기에 이는 필수이다.** 

  - QuerydslRepositorySupport: 상속 받아야 한다.

<br>
 
### JpaRepository에 Querydsl implement 하기

---

<img width="423" src="https://user-images.githubusercontent.com/74996516/205857919-5cc407ad-97e6-4f40-b8e7-f447af1f4e20.png"> 

<br>

### 비즈니스 메서드에서 사용하기

--- 

#### ArticleService.class
```java
@Transactional(readOnly = true)
public List<String> getHashtags() {
    return articleRepository.findAllDistinctHashtags();
}
```

#### ArticleControllerTest.class
```java
@DisplayName("View 컨트롤러 - 게시글")
@Import({TestSecurityConfig.class, FormDataEncoder.class})
@WebMvcTest(ArticleController.class)
class ArticleControllerTest {

    private final MockMvc mvc;
    private final FormDataEncoder formDataEncoder;

    @MockBean
    private ArticleService articleService;

    @MockBean
    private PaginationService paginationService;

    public ArticleControllerTest(@Autowired MockMvc mvc,
                                 @Autowired FormDataEncoder formDataEncoder) {
        this.mvc = mvc;
        this.formDataEncoder = formDataEncoder;
    }
    
    @DisplayName("[view][GET] 게시글 해시태그 검색 페이지 - 정상 호출")
    @Test
    public void givenNothing_whenRequestingArticleHashtagSearchView_thenReturnsArticleSearchHashtagView()
            throws Exception {
        // Given
        List<String> hashtags = List.of("#spring", "#jpa", "#코딩테스트");
        given(articleService.searchArticlesViaHashTag(eq(null), any(Pageable.class))).willReturn(Page.empty());
        given(articleService.getHashtags()).willReturn(hashtags);
        given(paginationService.getPaginationBarNumbers(anyInt(), anyInt())).willReturn(List.of(0, 1, 2, 3, 4));
        // When & Then
        mvc.perform(get("/articles/search-hashtag"))
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.TEXT_HTML))
                .andExpect(view().name("articles/search-hashtag"))
                .andExpect(model().attribute("articles", Page.empty()))
                .andExpect(model().attribute("hashtags", hashtags))
                .andExpect(model().attributeExists("paginationBarNumbers"))
                .andExpect(model().attribute("searchType", SearchType.HASHTAG));
        then(articleService).should().searchArticlesViaHashTag(eq(null), any(Pageable.class));
        then(articleService).should().getHashtags();
        then(paginationService).should().getPaginationBarNumbers(anyInt(), anyInt());
    }

    @DisplayName("[view][GET] 게시글 해시태그 검색 페이지 - 정상 호출, 해시태그 입력")
    @Test
    public void givenHashtag_whenRequestingArticleHashtagSearchView_thenReturnsArticleSearchHashtagView()
            throws Exception {
        // Given
        String hashtag = "#spring";
        List<String> hashtags = List.of("#spring", "#jpa", "#코딩테스트");
        given(articleService.searchArticlesViaHashTag(eq(hashtag), any(Pageable.class))).willReturn(Page.empty());
        given(articleService.getHashtags()).willReturn(hashtags);
        given(paginationService.getPaginationBarNumbers(anyInt(), anyInt())).willReturn(List.of(0, 1, 2, 3, 4));

        // When & Then
        mvc.perform(
                        get("/articles/search-hashtag")
                                .queryParam("searchValue", hashtag)
                )
                .andExpect(status().isOk())
                .andExpect(content().contentTypeCompatibleWith(MediaType.TEXT_HTML))
                .andExpect(view().name("articles/search-hashtag"))
                .andExpect(model().attribute("articles", Page.empty()))
                .andExpect(model().attribute("hashtags", hashtags))
                .andExpect(model().attributeExists("paginationBarNumbers"))
                .andExpect(model().attribute("searchType", SearchType.HASHTAG));
        then(articleService).should().searchArticlesViaHashTag(eq(hashtag), any(Pageable.class));
        then(articleService).should().getHashtags();
        then(paginationService).should().getPaginationBarNumbers(anyInt(), anyInt());
    }
```

<br>

#### ArticleController.class
```java
@RequiredArgsConstructor
@RequestMapping("/articles")
@Controller
public class ArticleController {

    private final ArticleService articleService;
    private final PaginationService paginationService;
    
    @GetMapping("/search-hashtag")
    public String searchArticleHashtag(
            @RequestParam(required = false) String searchValue,
            @PageableDefault(size = 10, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable,
            Model model
    ) {
        Page<ArticleResponse> articles = articleService.searchArticlesViaHashTag(searchValue, pageable).map(ArticleResponse::from);
        List<Integer> barNumbers = paginationService.getPaginationBarNumbers(pageable.getPageNumber(), articles.getTotalPages());
        List<String> hashtags = articleService.getHashtags();

        model.addAttribute("articles", articles);
        model.addAttribute("hashtags", hashtags);
        model.addAttribute("paginationBarNumbers", barNumbers);
        model.addAttribute("searchType", SearchType.HASHTAG);

        return "articles/search-hashtag";
    }
}
```
