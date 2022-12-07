---
title: "[Spring JPA] Spring Data JPA란?"
subtitle: "Spring Data Jpa 설명 / queryDSL를 이용한 인터페이스"
categories : Spring Spring_JPA
tags: [Spring JPA]
date: 2022-12-08 02:00:00 +0900
---

## Spring Data JPA란?

---

![img.png](https://user-images.githubusercontent.com/74996516/206245632-d0d035bb-72ab-4f18-8157-414da0a149d9.png)

- JPA란 자바 어플리케이션에서 관계형 데이터베이스 사용하는 방식을 정의한 인터페이스이다.

- Spring Data JPA는 Spring에서 제공하는 모듈 중 하나로, 개발자가 JPA를 더
  쉽고 편하게 사용할 수 있도록 도와준다.

- **JPA를 한 단계 추상화시킨 Repository 인터페이스 제공**
   
  - `implementation 'org.springframework.boot:spring-boot-starter-data-jpa'`
  - `public interface ArticleRepository extends JpaRepository<Article, Long> { }`

- build.gradle 의존성 추가 및 인터페이스만 정의해주게 되면 JPA의 CRUD를 바로 사용 가능하다.

- **단, 영속성 컨텍스트 및 Dirty Checking 개념을 잘 이해하고 사용하지 않으면
  데이터 손실 및 성능 이슈가 있을 수 있다.**

<br>

## Spring Data JPA 사용시 주의사항

---

![img_1.png](https://user-images.githubusercontent.com/74996516/206245638-77d4cb25-df0d-4551-8107-3f344c6807cb.png)

- **JPA의 모든 데이터 변경은 아래와 같이 트랜잭션 안에서 실행된다!**

- Spring Data JPA는 Spring에서 제공하는 모듈 중 하나로, 개발자가 JPA를 더
  쉽고 편하게 사용할 수 있도록 도와준다.

- 즉, 트랜잭션 밖에서 데이터 변경은 반영되지 않는다. 

- Spring Data JPA 구현 코드를 살펴보면, 변경이 일어나는 코드는 @Transactional이 이미 추가 되어 있다.
![img_2.png](https://user-images.githubusercontent.com/74996516/206245645-8a2ec38a-c20a-4b71-aeec-df4b0ef4817e.png)

- 즉, 구현 코드를 정확히 이해하지 않고 사용시 문제가 발생할 수 있다.

<br>

## 영속성 컨텍스트(Persistence Context)

- 영속성 컨텍스트는 entity를 저장하고 관리하는 저장소이며, 어플리케이션과 데이터베이스 사이에 entity를 보관하는 가상의 데이터베이스 같은 역할

- **Spring Data JPA에서 제공하는 save메소드 구현 코드를 보면 em.persist를 통해 영속성 컨텍스트에 저장**

- **이때, entity는 영속상태(`영속성 컨텍스트에서 관리되는 상태`)**이다.
- 이미 영속성태인 경우 merge를 통해 덮어 쓴다.


## 영속성 컨텍스트를 왜 사용하까?

- Database와 어플리케이션 사이의 중간 계층에 있으면서 여러가지 이점이 존재해서 사용한다.

- 영속성 컨텍스트 내에 1차 캐시

- 영속성 컨텍스트 내에 쓰기 지연 SQL 저장소
  
- 엔티티 수정(Dirty Checking)

<br>

### 영속성 컨텍스트 - 1차 캐시

---

![img_3.png](https://user-images.githubusercontent.com/74996516/206245651-be5817cd-b830-45bb-955f-6f7f91b74fe8.png)

- 영속성 컨텍스트 내부에 1차 캐시를 가지고 있다.

- persist를 하는 순간 pk값(id), 타입과 객체를 맵핑 하여 1차 캐시에 가지고 있음

- **한 트랜잭션 내에 1차 캐시에 이미 있는 값을 조회하는 경우 DB를 조회 하지 않고 1차 캐시에 있는 내용을 그대로 가져온다.**

- **단, 1차 캐시는 어플리케이션 전체 공유가 아닌 한 트랜잭션 내에서만 공유**

- 반면, 조회 했을 때 1차 캐시에 없다면 DB에서 가져와서 1차 캐시에 저장 후 반환


<br>

### 영속성 컨텍스트 - 쓰기 지연 SQL

---

![img_4.png](https://user-images.githubusercontent.com/74996516/206245655-58a7662a-e0f4-4f11-9dfb-dfeed3b2a78f.png)

- memberA를 persist 하는 순간, 
  1차 캐시에 넣고 쓰기 지연 SQL 저장소에 쿼리를 만들어 쌓는다.

- memberB도 persis하는 순간 동일한 과정을 거치며, commit 하는 순간 flush가 되면서 DB에 반영

- flush란 영속성 컨텍스트의 변경 내용을 DB에 반영하며, 1차 캐시를 지우지는 않는다.

- 트랜잭션이 commit이 될 때, 쓰기 지연 저장소에 쌓아 뒀던 쿼리를 한번에 데이터 베이스에 적용한다.

    - 예외적으로 쓰기 지연 저장소에 쌓아 두는 것이 아닌 바로 쿼리가 작동하는 경우가 존재한다.
  
    - 예: 엔티티를 save() 하고 나면 1차 캐시에 엔티티를 관리하기 위해 PK가 필요하다. 이 때 **쌓아 두는 것이 아닌 바로 쿼리가 작동한다.**

<br>

## JPA Dirty Checking

---

- 코드에서 엔티티의 값만 변경했을 뿐인데, 데이터 베이스 업데이트 쿼리가 발생한다?

- 이유는 Dirty Checking 덕분이며, Dirty란 상태의 변화가 생긴 정도를 의미한다.

- JPA에서 트랜잭션이 끝나는 시점에 변화가 있는 모든 entity 객체를 데이터 베이스에 자동으로 반영해 준다.

- 즉, Dirty Checking이란 entity 상태 변경 검사

<br>

### Dirty Checking 내부 구조

---

![img_5.png](https://user-images.githubusercontent.com/74996516/206245659-310646be-716d-4acf-bfa2-80407850d10d.png)

- JPA는 **commit 하는 순간 내부적으로 flush가 호출되고, 이때 엔티티와 스냅샷을 비교**

- **1차 캐시에는 처음 들어온 상태인 엔티티 스냅샷을 넣어두고
  commit 하는 순간 변경된 값이 있는지 비교하여 변경된 값이 있으면 update 쿼리를 쓰기 지연 SQL에 넣어둔다.**


### Dirty Checking 주의 사항

---

- 당연히 Dirty Checking은 영속성 컨텍스트가 관리하는 entity에만 적용된다.

- **영속성 컨텍스트에 처음 저장된 순간 스냅샷을 저장해놓고,
  트랜잭션이 끝나는 시점에 비교하여 변경된 부분을 쿼리로 생성하여 데이터 베이스로 반영**한다.

- **즉, 영속 상태가 아닐 경우 값을 변경해도 데이터베이스에 반영되지 않는다.**

- **트랜잭션이 없이 데이터 변경 반영이 일어나지 않는다.**
 

## 테스트 작성

### Service

```java
@Slf4j
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class PharmacyService {

    private final PharmacyRepository pharmacyRepository;

    @Transactional
    public void updateAddress(Long id, String address) {
        Pharmacy entity = pharmacyRepository.findById(id).orElse(null);

        if (Objects.isNull(entity)) {
            log.error("[PharmacyService updateAddress] not found id : {}", id);
            return;
        }

        entity.changePharmacyAddress(address);
    }
    
    // for test
    public void updateAddressWithoutTransaction(Long id, String address) {
        Pharmacy entity = pharmacyRepository.findById(id).orElse(null);

        if (Objects.isNull(entity)) {
            log.error("[PharmacyService updateAddress] not found id : {}", id);
            return;
        }

        entity.changePharmacyAddress(address);
    }
}
```

<br>

### Test

```java
package com.example.recommendationservice.domain.pharmacy.service

import com.example.recommendationservice.AbstractIntegrationContainerBaseTest
import com.example.recommendationservice.domain.pharmacy.entity.Pharmacy
import com.example.recommendationservice.domain.pharmacy.repository.PharmacyRepository
import org.springframework.beans.factory.annotation.Autowired

class PharmacyServiceTest extends AbstractIntegrationContainerBaseTest {

    @Autowired
    private PharmacyService pharmacyService

    @Autowired
    private PharmacyRepository pharmacyRepository

    def setup() {
        pharmacyRepository.deleteAll()
    }

    def "updateAddress: dirty Checking 성공 경우"() {
        given:
        String address = "서울특별시 성북구 보문동"
        String modifiedAddress = "서울특별시 서초구 잠원동"
        String name = "가까운 약국"
        double latitude = 37.58
        double longitude = 127.02

        def pharmacy = Pharmacy.builder()
                .pharmacyAddress(address)
                .pharmacyName(name)
                .latitude(latitude)
                .longitude(longitude)
                .build()

        when:
        // 트랜잭션 commit 되면서 insert 쿼리 발생
        def entity = pharmacyRepository.save(pharmacy)

        // 트랜잭션 commit 되면서 쿼리 발생
        pharmacyService.updateAddress(entity.getId(), modifiedAddress)

        // 다른 트랜잭션이기에 select 쿼리 발생
        println "==============================================================="
        def result = pharmacyRepository.findAll()
        println "==============================================================="

        then:
        result.get(0).getPharmacyAddress() == modifiedAddress
    }

    def "updateAddressWithoutTransaction: dirty Checking 실패 경우"() {
        given:
        String address = "서울특별시 성북구 보문동"
        String modifiedAddress = "서울특별시 서초구 잠원동"
        String name = "가까운 약국"
        double latitude = 37.58
        double longitude = 127.02

        def pharmacy = Pharmacy.builder()
                .pharmacyAddress(address)
                .pharmacyName(name)
                .latitude(latitude)
                .longitude(longitude)
                .build()

        when:
        // 트랜잭션 commit 되면서 insert 쿼리 발생
        def entity = pharmacyRepository.save(pharmacy)

        // 트랜잭션 commit 되면서 쿼리 발생
        pharmacyService.updateAddressWithoutTransaction(entity.getId(), modifiedAddress)

        // 다른 트랜잭션이기에 select 쿼리 발생
        println "==============================================================="
        def result = pharmacyRepository.findAll()
        println "==============================================================="

        then:
        result.get(0).getPharmacyAddress() == address
    }

}
```

### 테스트 결과

![img_6.png](https://user-images.githubusercontent.com/74996516/206245666-a3917876-b257-4ce9-b13f-67cdad64bf88.png)


## 정리 

`@Transactional`이 걸려져 있지 않으면 Dirty Checking, 즉 스냅샷과 비교해 update 쿼리가 작성되지 않는 것을 유의해야 한다.

**단, 1차 캐시는 어플리케이션 전체 공유가 아닌 한 트랜잭션 내에서만 공유**

`JPA에 대해 조심해서 사용해야 한다.`
