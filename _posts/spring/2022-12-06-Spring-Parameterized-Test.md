---
layout: post
title: "[Junit5] @ParameterizedTest 사용한다면?"
subtitle: "통합환경에서 테스트 진행하기 / @SpringBootTest 유의할 점"
categories : Spring Junit5
tags : [Spring, Junit5]
date: 2022-12-06 00:22:00 +0900
---

## 테스트 환경

---

> 현재 테스트 환경은 다음과 같다.  
> 117개의 sql 게시글이 존재한다.
>
> ![img.png](https://user-images.githubusercontent.com/74996516/205885063-044453b9-90be-477f-8bd7-ec8c2153b21f.png)
>
> spring.jpa.defer-datasource-initialization: true  
> ddl-auto를 먼저 실행하고, 그후 sql 스크립트 파일이 동작한다.
>
> [sql 스크립트 동작 설정방법 변경](http://ryeon9445.com/develop/1-springboot/)

<br/><br/>

```java
@DisplayName("비즈니스 로직 - 페이지네이션")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE, classes = PaginationService.class)
class PaginationServiceTest {

    private final PaginationService sut;

    // Test 에서는 꼭 생성자 주입에 @Autowired (X) -> org.junit.jupiter.api.extension.ParameterResolutionException
    // 주입 받는 것이 자기 자신이므로 그냥 new 로 구체화 할 수 있다.
    public PaginationServiceTest(@Autowired PaginationService paginationService) {
        this.sut = paginationService;
    }

    /**
     * 파라미터 값을 여러번 주입해서, 여러번 테스트를 진행할 수 있다. 입력값을 넣어주는 MethodSource 는 Method 형식으로 만든다. 이 때 테스트 메소드가 default와 다르면 적어줘야 한다.
     * ParameterizedTest DisplayName 설정 (JUNIT5 공식문서)
     */
    @DisplayName("현재 페이지 번호와 총 페이지 수를 주면, 페이징 바 리스트를 만들어준다.")
    @MethodSource
    @ParameterizedTest(name = "[{index}] 현재 페이지: {0}, 총 페이지: {1} => {2}")
    void givenCurrentPageNumberAndTotalPages_whenCalculating_thenReturnsPaginationBarNumbers(
            int currentPageNumber,
            int totalPages,
            List<Integer> expected) {
        // Given

        // When
        List<Integer> actual = sut.getPaginationBarNumbers(currentPageNumber, totalPages);

        // Then
        assertThat(actual).isEqualTo(expected);
    }

    static Stream<Arguments> givenCurrentPageNumberAndTotalPages_whenCalculating_thenReturnsPaginationBarNumbers() {
        return Stream.of(
                arguments(0, 12, List.of(0, 1, 2, 3, 4)),
                arguments(1, 12, List.of(0, 1, 2, 3, 4)),
                arguments(2, 12, List.of(0, 1, 2, 3, 4)),
                arguments(3, 12, List.of(1, 2, 3, 4, 5)),
                arguments(6, 12, List.of(4, 5, 6, 7, 8)),
                arguments(7, 12, List.of(5, 6, 7, 8, 9)),
                arguments(10, 12, List.of(8, 9, 10, 11)),
                arguments(11, 12, List.of(9, 10, 11))
        );
    }
}
```

- @SpringBootTest: deafult 값은 Mock 이다. `즉, Mocking한 환경을 주입한다.`

    - @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE): <br/> WebEnviroment을 None처리하여 무게를 경량화한다.

    - @SpringBootTest(classes = PaginationService.class): <br/> 읽을 클래스만 추가한다.

- @ParameterizedTest(name = "[{index}] 현재 페이지: {0}, 총 페이지: {1} => {2}"): 네이밍을 할 수 있다.

    - index는 예약어 이다.

<br>

## 테스트 코드를 기반으로 구현할 기능 요약

---

> 총 117개의 게시글이 존재하면 한 페이지의 10개씩 게시글을 보여준다고 한다.
>
> 즉, 총 12개의 페이지가 존재한다. 이를 통해 각 페이지의 페이징 기능을 검증하려고 한다.

## 구현 서비스 코드

---

```java
@Service
public class PaginationService {

    private static final int BAR_LENGTH = 5;

    public List<Integer> getPaginationBarNumbers(int currentPageNumber, int totalPages) {
        int startNumber = Math.max(0, currentPageNumber - (BAR_LENGTH / 2));
        int endNumber = Math.min(startNumber + BAR_LENGTH, totalPages);

        return IntStream.range(startNumber, endNumber).boxed().toList();
    }

    public int currentBarLength() {
        return BAR_LENGTH;
    }
}
```
