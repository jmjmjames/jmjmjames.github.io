---
layout: post
title: "[Spring JPA] 다대다(N:M) 관계 매핑, 좋아요 기능 구현"
subtitle: "Querydsl 활용하기"
categories : Spring Spring_JPA
tags : [Spring JPA, Querydsl]
date: 2022-12-07 13:00:00 +0900
---

## 시작하기에 앞서

---

두 테이블끼리 연관 관계를 맺을 때, 다양한 연관 관계를 가진다.

이는 일대다(1:N), 다대일(N:1), 일대일(1:1), 다대다(N:N)으로 이루어져있다.

일대다와 다대일 같은 경우 서로 관계를 맺기 위해 한 쪽 테이블에서 하나의 외래키를 가져 연관 관계를 관리한다.

(테이블에서는 다(N)쪽에서 외래키를 관리하고 JPA에서는 다(N)쪽에서 관리하는 것을 지향한다.)

일대일 같은 경우에서도 일대다와 다대일과 비슷하게 한 쪽 테이블에서 하나의 외래키를 관리하면 된다.

하지만, 위와 같이 어느 한쪽에서 꼭 관리해야될 필요는 없고 둘 중 한 곳에서만 관리하면 된다.

보통, 서버 개발자의 경우 참조의 용이성 등 같은 이유로 주 테이블에 외래키를 두는 것을 선호한다.

하지만, DBA의 경우 외래키의 NULL 값 허용, 추후 주테이블과 보조테이블간의 관계의 일대다(N:1) 관계 변경으로 인한 변경 파급력 최소화 등의 이유로 보조 테이블에 외래키를 두는 것을 선호한다.

그렇지만 결국 일대다, 다대일, 일대일은 정규화된 테이블 2개로 관계를 표현할 수 있다는 점이다.

<br>

그렇다면 다대다를 살펴보자.

DB를 공부한 사람은 다 알겠지만 다대다의 관계는 정규화된 테이블 2개로 관계를 표현할 수 없다.

그에 따라, 두 테이블 사이에 중간 테이블을 두어 관계를 풀어줘야한다.

JPA에서는 다음 경우를 위해 @ManyToMany어노테이션을 제공한다.

하지만, 이 어노테이션은 우리가 지양해야할 부분 중 하나이다. 그 이유는 이 부분이 실무에서 한계가 존재하기 때문이다.

그 이유는 다음과 같다.

- 중간 테이블을 생성해주긴 하지만 묵시적으로 생성해주기 때문에 자기도 모르는 복잡한 조인의 쿼리가 발생하는 경우가 생길 수 있다.
- 우리는 중간 테이블에 두 테이블의 기본키를 기본키이자 외래키로 들고와서 추가로 필요한 컬럼이 존재할 확률이 크지만, 중간 테이블에 필요한 추가 컬럼을 사용할 수 없다. (두 테이블에 추가된 컬럼에 대해 매핑이
  되지 않기 때문이다.)
  우리는 이를 해결하기 위해 다대다 관계를 일대다, 다대일 관계로 풀어 직접 사용하는 것이 좋다.

이를 간단한 예시를 통해 알아보자.

<br>

## 엔티티

---

학생은 여러 게시물에 대해 좋아요를 누를 수 있고, 한 게시물은 여러 학생에 의해 좋아요를 받을 수 있다.

이 문장을 보면 이 관계가 다대다 관계임을 알 수 있다.

이 부분에 대해서 중간 테이블을 직접 생성하여 일대다 다대일 관계로 풀어 해결해보자한다.

클래스 다이어그램을 살펴보면, 다음과 같다.

<img src="https://github.com/SangHyunGil/Blog/blob/master/img/%EB%8B%A4%EB%8C%80%EB%8B%A4/1.png?raw=true">

<br>

### 학생

---

```java
@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "student_id")
    private Long id;
    private String email;
    private String password;
    private String name;
    private String nickname;
    private String department;
    private String major;

    protected Student() {
    }

    @Builder
    public Student(String email, String password, String name, String nickname, String department, String major) {
        this.email = email;
        this.password = password;
        this.name = name;
        this.nickname = nickname;
        this.department = department;
        this.major = major;
    }
}
```

- 학생은 식별자와 다양한 정보를 가지고 있다.
- 좋아요와 1:N관계를 가져 @OneToMany어노테이션을 선언하였다

<br>

### 좋아요

---

```java
@Getter
@Entity
public class PostLike {
    @Id @GeneratedValue
    @Column(name = "postlike_id")
    private Long id;

    @ManyToOne
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne
    @JoinColumn(name = "board_id")
    private Board board;

    private LocalDateTime likeTime;
    
    protected PostLike() {
    }

    @Builder
    public PostLike(Student student, Board board, LocalDateTime likeTime) {
        this.student = student;
        this.board = board;
        this.likeTime = likeTime;
    }
}
```

- 좋아요는 식별자와 학생 엔티티와 게시물 엔티티를 외래키로 받는다.
- 추가적으로 좋아요를 누른 시각을 저장한다.
- 단방향으로 설계하였다.

<br>

### 게시물

---

```java
@Getter
@Entity
public class Board {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "board_id")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id")
    private Student writer;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;

    @OneToMany(mappedBy = "board", cascade = CascadeType.ALL)
    private List<Attachment> attachedFiles = new ArrayList<>();

    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime writeTime;
    private Boolean isDeleted;
    private String title;
    private String content;
    private Integer hit;

    @Builder
    public Board(Student writer, Category category, List<Attachment> attachedFiles, LocalDateTime writeTime, Boolean isDeleted, String title, String content, Integer hit) {
        this.writer = writer;
        this.category = category;
        this.attachedFiles = attachedFiles;
        this.writeTime = writeTime;
        this.isDeleted = isDeleted;
        this.title = title;
        this.content = content;
        this.hit = hit;
    }

    public void setAttachment(Attachment attachment) {
        this.attachedFiles.add(attachment);
        attachment.setBoard(this);
    }
}
```

- 게시물은 식별자와 다양한 정보들을 가지고 있다.
- 좋아요와 1:N관계를 가져 @OneToMany어노테이션을 선언하였다.

<br><br>

## Repository

단순한 CURD에 대해서는 Spring Data JPA를 활용히여 간단하게 처리했다.

```java
public interface PostLikeRepository extends JpaRepository<PostLike, Long>, PostLikeCustomRepository {
}
```

추가적으로 다른 기능을 구현하기 위해 Custom한 Repository를 구성하여 Querydsl로 구현하였다.

PostLikeCustomRepository를 구현한 후, 이를 구현하는 구현체를 구현하였다.

```java
public class PostLikeCustomRepositoryImpl extends QuerydslRepositorySupport implements PostLikeCustomRepository{

    public PostLikeCustomRepositoryImpl() {
        super(PostLike.class);
    }
  
    public Optional<PostLike> exist(Long studentId, Long boardId) {
        QPostLike postLike = QPostLike.postLike;
        
        PostLike pLike = from(postLike)
                .select(postLike)
                .where(postLike.student.id.eq(studentId),
                        postLike.board.id.eq(boardId))
                .fetchFirst();

        return Optional.ofNullable(pLike);
    }

    public long findPostLikeNum(Long boardId) {
        QPostLike postLike = QPostLike.postLike;
        
        return from(postLike)
                .select(postLike)
                .where(postLike.board.id.eq(boardId))
                .fetchCount();
    }
}
```

exist는 좋아요를 누른 적이 있는지를 확인하는 메소드이다.

해당 부분은 학생과 게시물이 주어지면 해당 값을 통해 비교하여 단순히 찾아지면 바로 종료하여 효율적으로 처리할 수 있게 진행했다.

추가적으로 해당 PostLike 객체를 사용하여 Boolean이 아니라 PostLike 객체를 반환했다.

`(count의 경우 전체를 탐색한 후 count() > 0을 진행한다.)`

<br>

findPostLikeNum은 게시물을 입력받고 해당 게시물의 좋아요 총 개수를 구하는 메소드이다.

단순히 fetchCount를 이용해 구한 뒤 반환하였다.


## Service

```java
@Service
@Transactional
@RequiredArgsConstructor
public class PostLikeServiceImpl implements PostLikeService{

    private final PostLikeRepository postLikeRepository;
    private final BoardRepository boardRepository;

    // 좋아요 및 취소 
    public Boolean pushLikeButton(PostLikeDto postLikeDto) {
        postLikeRepository.exist(postLikeDto.getStudent().getId(), postLikeDto.getBoardId())
                .ifPresentOrElse(
                        postLike -> postLikeRepository.deleteById(postLike.getId()),
                        () -> {
                            Board board = getBoard(postLikeDto);
                            postLikeRepository.save(new PostLike(postLikeDto.getStudent(), board, LocalDateTime.now()));
                        });

        return true;
    }

    @Transactional(readOnly = true)
    public Board getBoard(PostLikeDto postLikeDto) {
        return boardRepository.findById(postLikeDto.getBoardId()).orElseThrow(() -> new IllegalArgumentException("해당 게시글은 존재하지 않습니다."));
    }

    // 좋아요 개수
    @Transactional(readOnly = true)
    public PostLikeResponseDto getPostLikeInfo(PostLikeDto postLikeDto) {
        long postLikeNum = getPostLikeNum(postLikeDto);
        boolean check = checkPushedLike(postLikeDto);

        return new PostLikeResponseDto(postLikeNum, check);
    }

    @Transactional(readOnly = true)
    public boolean checkPushedLike(PostLikeDto postLikeDto) {
        return postLikeRepository.exist(postLikeDto.getStudent().getId(), postLikeDto.getBoardId())
                .isPresent();
    }

    @Transactional(readOnly = true)
    public long getPostLikeNum(PostLikeDto postLikeDto) {
        return postLikeRepository.findPostLikeNum(postLikeDto.getBoardId());
    }
}
```


<br>

## Test

### 좋아요

---

```java
@Test
public void 좋아요() throws Exception {
    //given
    Student student = new Student("testID@gmail.com", "testPW", "테스터", "테스터", "컴공", "백엔드");
    Student joinStudent = studentService.join(student);

    BoardAddForm boardAddForm = new BoardAddForm("테스트 글", "테스트 글입니다.", CategoryType.BACK, null, null);
    BoardPostDto boardPostDto = boardAddForm.createBoardPostDto(joinStudent);
    Board board = boardService.post(boardPostDto);

    //when
    PostLikeDto postLikeDto = new PostLikeDto(joinStudent, board.getId());
    postLikeService.pushLikeButton(postLikeDto);

    //then
    assertEquals(1, postLikeService.getPostLikeInfo(postLikeDto).getPostLikeNum());
}
```

- 가입한 한 학생이 게시물을 작성하고 좋아요를 눌렀다.
- 좋아요의 개수는 1과 같아야 한다.

### 좋아요 취소 

---

```java
@Test
public void 좋아요취소() throws Exception {
    //given
    Student student1 = new Student("testID1@gmail.com", "testPW1", "테스터A", "테스터A", "컴공", "백엔드");
    Student joinStudentA = studentService.join(student1);

    Student student2 = new Student("testID2@gmail.com", "testPW2", "테스터B", "테스터B", "컴공", "백엔드");
    Student joinStudentB = studentService.join(student2);

    BoardAddForm boardAddForm = new BoardAddForm("테스트 글", "테스트 글입니다.", CategoryType.BACK, null, null);
    BoardPostDto boardPostDto = boardAddForm.createBoardPostDto(joinStudentA);
    Board board = boardService.post(boardPostDto);

    PostLikeDto postLikeDtoA = new PostLikeDto(joinStudentA, board.getId());
    postLikeService.pushLikeButton(postLikeDtoA);

    PostLikeDto postLikeDtoB = new PostLikeDto(joinStudentB, board.getId());
    postLikeService.pushLikeButton(postLikeDtoB);

    //when
    postLikeService.pushLikeButton(postLikeDtoB);

    //then
    assertEquals(1, postLikeService.getPostLikeInfo(postLikeDtoB).getPostLikeNum());
}
```

- 두 학생이 가입을 하고 한 학생이 게시물을 작성하였다.
- 두 학생 다 좋아요를 눌렀다.
- 한 학생이 좋아요를 다시 눌러 취소했다.
- 좋아요의 개수는 1과 같아야 한다.

### 좋아요 갯수

---

```java
@Test
public void 좋아요개수조회() throws Exception {
    //given
    Student student1 = new Student("testID1@gmail.com", "testPW1", "테스터A", "테스터A", "컴공", "백엔드");
    Student joinStudentA = studentService.join(student1);

    Student student2 = new Student("testID2@gmail.com", "testPW2", "테스터B", "테스터B", "컴공", "백엔드");
    Student joinStudentB = studentService.join(student2);

    BoardAddForm boardAddForm = new BoardAddForm("테스트 글", "테스트 글입니다.", CategoryType.BACK, null, null);
    BoardPostDto boardPostDto = boardAddForm.createBoardPostDto(joinStudentA);
    Board board = boardService.post(boardPostDto);

    PostLikeDto postLikeDtoA = new PostLikeDto(joinStudentA, board.getId());
    postLikeService.pushLikeButton(postLikeDtoA);

    PostLikeDto postLikeDtoB = new PostLikeDto(joinStudentB, board.getId());
    postLikeService.pushLikeButton(postLikeDtoB);

    //when
    long postLikeNum = postLikeService.getPostLikeInfo(postLikeDtoB).getPostLikeNum();

    //then
    assertEquals(2, postLikeNum);
}
```

- 두 학생이 가입을 하고 한 학생이 게시물을 작성하였다.
- 두 학생 다 좋아요를 눌렀다.
- 좋아요의 개수는 2와 같아야 한다.

### 좋아요 여부

---

```java
@Test
public void 좋아요여부확인() throws Exception {
    //given
    Student student = new Student("testID@gmail.com", "testPW", "테스터", "테스터", "컴공", "백엔드");
    Student joinStudent = studentService.join(student);

    BoardAddForm boardAddForm = new BoardAddForm("테스트 글", "테스트 글입니다.", CategoryType.BACK, null, null);
    BoardPostDto boardPostDto = boardAddForm.createBoardPostDto(joinStudent);
    Board board = boardService.post(boardPostDto);

    //when
    PostLikeDto postLikeDto = new PostLikeDto(joinStudent, board.getId());
    postLikeService.pushLikeButton(postLikeDto);
    Boolean check = postLikeService.getPostLikeInfo(postLikeDto).getCheckLiked();

    //then
    assertEquals(true, check);
}
```

- 가입한 한 학생이 게시물을 작성하고 좋아요를 눌렀다.
- 해당 학생은 좋아요를 눌렀다고 나와야 한다.

### 테스트 결과

다음과 같이 테스트가 잘 통과된 모습을 확인할 수 있다.

![img.png](https://github.com/SangHyunGil/Blog/blob/master/img/%EB%8B%A4%EB%8C%80%EB%8B%A4/2.PNG?raw=true)

## 정리

다음과 같이 최대한 다대다를 지양하고 일대다, 다대일 관계로 풀어 사용하는 것을 지향하자.

물론, 일대다 다대일 관계로 풀어 사용하는 것에 대해 식별자를 대리키로 설정하지 않고 두 테이블의 식별자를 식별자이자 외래키로 하는

복합키 전략이 존재하긴 하지만 전자의 방법이 효율적이기에 전자의 방법을 사용하는 것을 지향하는 것이 좋다.
