---
title: "[Java] 기본 자료형과 참조 자료형의 차이"
categories : Java
tags : java
date: 2022-12-06 15:56:00 +0900
---

> 기본 자료형과 참조 자료형의 차이

<br>

## 자료형의 이름 규칙

---

`int, long, float, double ... 모두 소문자로 시작한다.`  
`반면, 참조 자료형의 이름은 모두 대문자(String, Long, Integer...)로 시작한다.`

<br><br>

## 실제 데이터 값의 저장 위치

---
<img width="500" src="https://user-images.githubusercontent.com/74996516/205845764-3d1625fa-293f-460b-adae-f55346a6fa39.png">


- 기본 자료형은 스택 메모리에 생성된 공간에 실제 변숫값을 저장한다.

    - 기본 자료형에 속하는 변수 a는 공간이 스택 메모리에 만들어지고, 이 공간 안에 실제 데이터 값(3)이 저장된다.

- 참조 자료형은 실제 데이터값은 힙 메모리에 저장하고, 스택 메모리에 변수 공간에는 실제 변수값이 저장된 힙 메모리의 위치값을 저장한다.

    - 참조 자료형 변수 b에서 실제 데이터 값("안녕")은 힙 메모리에 저장하고, 스택 메모리에 있는 b의 공간에는 힙 메모리에 실제 데이터값의 위치를 저장한다.

    - 메서드에서 참조 자료형을 복사할 때도 `모두 스택 메모리의 변수 공간에 값을 복사해서 전달한다.`

<br>

> 자바는 힙 메모리에 직접 접근할 수가 없으므로 반드시 위칫값을 기준으로 저장하고 있는 참조 변수 b가 필요하다.

<br><br>

## 기본 자료형 간의 타입 변환 (추가 내용)

boolean을 제외한 기본 자료형 7개는 자료형을 서로 변환할 수 있는데, 이를 `타입 변환(type casting)`이라고 한다.  
자바는 항상 대입 연산자(`=`)를 중심으로 왼쪽과 오른쪽 자료형을 일시시켜야 하므로 타입 변환을 수행해야 할 때가 있다.

`타입 변환 방법은 단순히 변환 대상 앞에 (*자료형*)만 표기하면 된다.`

```java
public class TypeCasting {
    public static void main(String[] args) {
        
        int value1 = (int) 5.3;
        long value2 = (long) 11;
        long value3 = 11L;
        
        float value4 = 5.8F;
        double value5 = (double) 126;
    }
}
```

<br><br>

### 자동 타입 변환과 수동 타입 변환

---

##### 자동 타입 변환

---

타입 변환에는 컴파일러가 자동으로 수행하는 `자동 타입 변환`과 개발자가 직접 타입 변환을 수행해야 하는 `수동 타입 변환`이 있다.  
먼저 크기(범위)가 작은 자료형을 큰 자료형에 대입 하는 경우는 `어떠한 데이터 손실도 발생하지 않는다.`   
따라서 작은 자료형을 큰 자료형에 담으면 개발자가 타입 변환 코드를 넣어 주지 않더라도 컴파일러가 자동 적으로 타입 변환을 실행하는데, 이를 `업캐스팅`이라고 한다.

이와 더불어 업캐스팅이 아닌데도 자동 타입 변환이 적용되는 때가 있다. 사실 모든 정수 리터럴값은 int 자료형으로 인식된다.  
하지만 byte 및 shotr 자료형에 저장할 수 있는 범위 내의 정수 리터럴값이 대입될 때는 자동 타입 변환이 각각의 자료형으로 수행된다.

<br>

##### 수동 타입 변환

---

큰 자료형을 작은 자료형에 대입하는 행위를 `다운 캐스팅`이라고 한다. 이때는 `데이터 손실이 발생할 수 있으므로 컴파일러에 따른 자동 타입 변환은 일어 나지 않는다.`  
즉, 개발자가 직접 명시적으로 타입 변화을 수행해야 한다.

> 자료형의 크기: byte < short/char < int < long < float < double

```java
public class TypeCasting {
    public static void main(String[] args) {
        
        // 자동 타입 변환 
        float value1 = 7;  // int -> float(업캐스팅)
        long value2 = 5;  // int -> long(업캐스팅)
        double value3 = 1;  // int -> long(업캐스팅)
        byte value4 = 8;  // int -> byte
        short value5 = 11;  // int -> short
        
        // 수동 타입 변환 
        int value6 = (int) 5.3;
        long value7 = (long) 11;
        long value8 = (float) 7.5;
        byte value9 = (byte)128;  // int -> byte(다운캐스팅) -> (-128 ~ 127)
        
        // 데이터 손실 발생
        System.out.print(value6);  // 5
        System.out.print(value9);  // -128
        
        double a = 7.000000005;
        System.out.print(a);  // 7.000000005
        float b = (float) 7.000000005;
        System.out.print(a);  // 7.0
    }   
}
```

> float의 정밀도: 대략 소수 7자리
>
> double의 정밀도: 대략 소수 15자리

<br>

### 기본 자료형 간의 연산

|       연산        |   결과   |
|:---------------:|:------:|
|   byte + byte   |  int   |
|  short + short  |  int   |
|    int + int    |  int   |
|   long + long   |  long  |
|  float + float  | float  |
| double + double | double |
|   int + float   | float  |
|  long + float   | float  |
| float + double  | double |

