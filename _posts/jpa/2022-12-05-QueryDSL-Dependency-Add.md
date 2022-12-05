---
title: "[Spring JPA] QueryDSL Dependency 추가하는 방법"
subtitle: "Spring boot 2.6 이전과 이후 queryDSL 설정 방법 비교하기 / queryDSL를 이용한 인터페이스"
categories : JPA
date: 2022-12-05 20:28:00 +0900
---

## Spring boot 2.7 이전 QueryDSL 설정

##### build.gradle

---

```gradle
plugins {
    id 'org.springframework.boot' version '2.6.12'
    id 'io.spring.dependency-management' version '1.0.14.RELEASE'
    id 'java'
}

group = 'com.example'
version = 'v1.1'
sourceCompatibility = '17'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    // Querydsl
    implementation 'com.querydsl:querydsl-jpa'
    // Querydsl JPAAnnotationProcessor 사용 지정
    annotationProcessor "com.querydsl:querydsl-apt:5.0.0:jpa"
    // java.lang.NoClassDefFoundError(javax.annotation.Entity) 발생 대응
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
    // java.lang.NoClassDefFoundError(javax.annotation.Generated) 발생 대응
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
}

// clean task 실행시 QClass 삭제
clean {
    delete file('src/main/generated') // 인텔리제이 Annotation processor 생성물 생성 위치
}
```

## Spring boot 2.7 QueryDSL 설정

---

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.7.6'
    id 'io.spring.dependency-management' version '1.0.15.RELEASE'
}

group = 'com.example'
version = 'v1.1'
sourceCompatibility = '17'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    // queryDSL 설정
    implementation "com.querydsl:querydsl-jpa"
    implementation "com.querydsl:querydsl-core"
    implementation "com.querydsl:querydsl-collections"
    annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"
    // querydsl JPAAnnotationProcessor 사용 지정
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    // java.lang.NoClassDefFoundError (javax.annotation.Generated) 대응 코드
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
    // java.lang.NoClassDefFoundError (javax.annotation.Entity) 대응 코드
}

// Querydsl 설정부
def generated = 'src/main/generated'

// querydsl QClass 파일 생성 위치를 지정
tasks.withType(JavaCompile) {
    options.getGeneratedSourceOutputDirectory().set(file(generated))
}

// java source set 에 querydsl QClass 위치 추가
sourceSets {
    main.java.srcDirs += [generated]
}

// gradle clean 시에 QClass 디렉토리 삭제
clean {
    delete file(generated)
}

jar {
    enabled = false
}
```


