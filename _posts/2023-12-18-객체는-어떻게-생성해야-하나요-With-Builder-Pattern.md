---
title: "객체는 어떻게 생성해야 하나요? With Builder Pattern"
date: 2023-12-18 13:55:00 +09:00
categories: [Java, 기본기, Builder]
author: wlgns12370
tags: [Java, 기본기, Builder] # TAG names should always be lowercase
---

Spring Boot를 사용하여 객체를 생성한다면 여러분들은 어떻게 생성하실 건가요? 요즘 시대에 적용되고 있는 방법들을 소개해 보겠습니다. 먼저 객체를 생성하기 전 **필드의 불변과 가변의 유무를 판단해야 합니다.**

> 💡 불변은 “변할 수 없는” 가변은 “변할 수 있는”으로 해석해 주시면 됩니다.
>

### 클래스 필드 분류

- 필드 중 불변 필드만 존재하는 경우
- 필드 중 불변 필드와 가변 필드가 모두 존재하는 경우
- 필드 중 가변 필드만 존재하는 경우

대부분 실제 개발을 하게 되면 **필드 중 불변 필드와 가변 필드가 모두 존재하는 경우**가 대다수입니다. 따라서 이 경우에 대해서 먼저 알아보겠습니다!

예를 들어 아래와 같은 요구사항이 존재하는 써브웨이 주문 시스템을 만들어 본다고 가정해 보겠습니다. 

- 주문 번호는 주문 한 순간부터 변경될 수 없다. (불변 필드)
- 메뉴 이름, 빵 종류, 토핑, 야채, 소스, 세트 유무는 주문 중간에 변경이 가능하다. (가변 필드)

```java
public class Subway {
    
    /* 
    불변 필드
    */

    // 써브웨이 주문 번호
    private Long id;

    /* 
    가변 필드
    */

    // 메뉴 이름
    private String menuName;
    
    // 빵 종류
    private String bread;
    
    // 토핑 종류
    private String topping;

    // 야채 종류
    private String vegetable;
    
    // 소스 종류
    private String sauce;

    // 세트 유무
    private String isSet;

    // 기본 생성자
    public Subway(Long id, String menuName, String bread, String topping, String vegetable, String sauce, boolean isSet) {
        this.id = id;
        this.menuName = menuName;
        this.bread = bread;
        this.topping = topping;
        this.vegetable = vegetable;
        this.sauce = sauce;
        this.isSet = isSet;
    }
}
```

해당 요구사항에 대하여 위 코드와 같이 필드를 세팅할 수 있습니다. 이후 자바에서 일반적으로 사용하는 기본 생성자를 선언하였습니다. 

## 점층적 생성자 패턴(**Telescoping Constructor Pattern)**

- 생성자를 필요한 매개변수의 개수만큼 만드는 패턴

```java
public Subway() {}

public Subway(Long id) {
    this.id = id;
}

public Subway(Long id, String menuName) {
    this(id);
    this.menuName = menuName;
}

public Subway(Long id, String menuName, String bread) {
    this(id, menuName);
    this.bread = bread;
}

public Subway(Long id, String menuName, String bread, String topping) {
    this(id, menuName, bread);
    this.topping = topping;
}

// ... (생략) ...

// 세트유무만 기본값인 생성자
public Subway(Long id, String menuName, String bread, String topping, String vegetable, String sauce) {
    this(id, menuName, bread, topping, vegetable);
    this.sauce = sauce;
}

public Subway(Long id, String menuName, String bread, String topping, String vegetable, String sauce, String isSet) {
    this(id, menuName, bread, topping, vegetable, sauce);
    this.isSet = isSet;
}
```

만약 써브웨이 객체를 생성할 때, **세트 유무만 기본값**으로 설정한다면 `세트 유무만 기본값인 생성자`를 사용해서 생성하면 됩니다. 그러나 사용자가 만약 **소스 종류만 기본값**으로 설정하려고 합니다.

```java
// 세트 유무만 기본값인 생성자
public Subway(Long id, String menuName, String bread, String topping, String vegetable, String sauce) {
    this(id, menuName, bread, topping, vegetable);
    this.sauce = sauce;
}

// 소스 종류만 기본값인 생성자
public Subway(Long id, String menuName, String bread, String topping, String vegetable, String isSet) {
    this(id, menuName, bread, topping, vegetable);
    this.isSet = isSet;
}
```

위 코드와 같이 `소스 종류만 기본값인 생성자`를 사용하여 생성할 수 있습니다. 하지만 `세트 유무만 기본값인 생성자` , `소스 종류만 기본값인 생성자` 가 동시에 존재할 수 없습니다. 즉 **매개변수의 개수가 같은 생성자**는 서로 중복되어 같이 존재할 수 없기 때문에, **하나의 생성자만 존재합니다.**

또한 매개변수가 많아질수록 코드를 읽기 어렵고, 파라미터의 순서를 잘못 입력할 가능성이 있습니다.

### 장점

- 객체의 불변성을 유지할 수 있다.
- 구현이 간단하다.

### 단점

- 매개변수가 많을수록 가독성이 떨어진다.
- 매개변수가 많아질수록 생성자의 수가 많아지고 클래스가 복잡해진다.
- 매개변수의 개수가 같은 다양한 생성자를 만들 수 없다.

## 자바 빈즈 패턴(Java Beans Pattern)

- 빈 객체를 생성한 후, `setter` 메서드를 통해 필요한 필드를 설정하는 패턴

```java
public class JavaBeansSubway {
    /* 
    불변 필드
    */

    // 써브웨이 주문 번호
    private Long id;

    /* 
    가변 필드
    */

    // 메뉴 이름
    private String menuName;
    
    // ... (생략) ...

    // 파라미터가 없는 빈 객체
    public JavaBeansSubway() {}

    public void setId(Long id) {
        this.id = id;
    }

    public void setMenuName(String menuName) {
        this.menuName = menuName;
    }

    // ... (생략) ...
}
```

위 코드와 같이 자바 빈즈 패턴은 `JavaBeansSubway`를 통해 비어있는 객체를 생성한뒤, `setter` 메서드를 호출하여 객체를 채워 넣습니다.

```java
public class Application {
    /*
    자바 빈즈 패턴
    */
    JavaBeansSubway javaBeansSubway = new JavaBeansSubway();

    // 원하는 매개변수의 값 설정
    javaBeansSubway.setId(id);
    javaBeansSubway.setMenuName(menu);
    javaBeansSubway.setBread(bread);
    javaBeansSubway.setTopping(topping);
    javaBeansSubway.setVegetable(vegetable);
    javaBeansSubway.setSauce(sauce);
    javaBeansSubway.setIsSet(isSet);
}
```

자바 빈즈 패턴을 사용하면 원하는 **매개변수의 값을 설정**하여 객체를 생성하게 되었습니다. 또한 많은 메서드가 있지만 인스턴스 생성이 쉽고, **가독성이 좋아졌습니다.** 

하지만 자바 빈즈 패턴은 `setter`로 모든 값을 생성 및 변경이 가능하기 때문에, **불변의 객체를 만들 수 없습니다.**

### 자바 빈즈 패턴- 레이스 컨디션

```java
public class RaceConditionSubway extends Thread{
    private JavaBeansSubway subway;

    public RaceConditionSubway(JavaBeansSubway subway) {
        this.subway = subway;
    }

    @Override
    public void run() {
        subway.setMenuName("Italian B.M.T.");
    }

    public static void main(String[] args) {
        JavaBeansSubway subway = new JavaBeansSubway();
        subway.setMenuName("B.L.T.");

        RaceConditionSubway thread1 = new RaceConditionSubway(subway);
        RaceConditionSubway thread2 = new RaceConditionSubway(subway);

        thread1.start();
        thread2.start();
    }
}
```

자바 빈즈 패턴은 **멀티 스레드 환경에서** 레이스 컨디션, 데드락 등 스레드로 부터 **안전하다고 하기 힘듭니다.** 레이스 컨디션이 발생할 수 있는 상황을 예로 들어 보겠습니다.

![Untitled](https://github.com/Principes-Artis-Mechanicae/Principes-Artis-Mechanicae.github.io/assets/30788586/5646b106-c40c-43e3-b099-d5527e520ab7)

위 코드의 `JavaBeansSubway` 객체의 `menuName` 속성을 **여러 스레드가 동시에 접근하고 변경**할 때, `thread1`과 `thread2`가 **경쟁**하여 `menuName` 속성을 `Italian B.M.T.`로 변경하려고 합니다. 하지만 **어떤 스레드가 먼저 실행되고 완료될지 예측하기 어렵**기 때문에, `menuName`은 어떤 값으로 설정될지 알 수 없게 됩니다. 이처럼 자바빈즈 패턴은 객체의 **일관성(Consistency)**이 깨져 **불변성(immuta)**을 보장할 수 없다는 단점이 있습니다.

### 장점

- 점층적 생성자 패턴과 다르게 매개변수에 의미를 담아 가독성이 향상된다.

### 단점

- 객체를 완성하는 동안 객체의 일관성이 보장되지 않는다.
    - 빈 객체에서 모든 `setter` 를 실행하기 전까지 완성된 객체가 아니다
    - 일관성이 무너지면 `SIP(Single Responsibility Principle)`가 위배된다.
- 객체의 불변성을 보장할 수 없다.
    - 처음 생성된 이후 언제던 변경 가능하기 때문에, 불변성을 보장할 수 없다
    - 대안으로 `freeze` 를 사용하여 객체를 동결하는 방법이 있다.
- 필드의 개수가 많을 때, 객체 생성 시 메서드 호출의 수가 많아진다.

## 빌더 패턴

- 점층적 생성자 패턴의 **일관성**과 자바 빈즈 패턴의 **가독성**, 두 패턴의 장점을 합친 패턴

```java
public class Subway {
    private Long id;
    private String menuName;
    private String bread;
    private String topping;
    private String vegetable;
    private String sauce;
    private String isSet;
}
```

이제 써브웨이 주문을 생성할 때, 불변성도 지키면서 가독성까지 뛰어난 패턴에 대해서 알아봅시다. 먼저 `Subway` 클래스 필드와 **구성이 똑같은** `SubwayBuilder`를 만듭니다.

```java
public class SubwayBuilder {
    private Long id;
    private String menu;
    private String bread;
    private String topping;
    private String vegetable;
    private String sauce;
    private String isSet;
	
    public SubwayBuilder(Long id) {
        this.id = id;
    }

    public SubwayBuilder menu(String menu) {
        this.menu = menu;
        return this;
    }

    public SubwayBuilder bread(String bread) {
        this.bread = bread;
        return this;
    }

    public SubwayBuilder topping(String topping) {
        this.topping = topping;
        return this;
    }

    public SubwayBuilder vegetable(String vegetable) {
        this.vegetable = vegetable;
        return this;
    }

    public SubwayBuilder sauce(String sauce) {
        this.sauce = sauce;
        return this;
    }

    public SubwayBuilder set(String isSet) {
        this.isSet = isSet;
        return this;
    }

    public Subway build() {
        return new Subway(id, menu, bread, topping, vegetable, sauce, isSet);
    }
}
```

써브웨이 주문 번호(id)는 **불변 필드이기 때문에 생성자**로 입력받았습니다. 이후 **가변 필드는 메서드 체이닝**을 통해 값을 받아오고 `build()` 메서드로 `Subway` 객체를 반환합니다.

> 💡 메서드 체이닝(Method Chaining) : `return this`를 반환 즉 객체 자기 자신을 반환하여, 메서드 호출을 순차적으로 연결한 후 실행하게 해주는 프로그래밍 기법입니다. 호출 시점에서 코드의 가독성 향상이 되고 필요한 메서드를 선택적으로 호출할 수 있습니다.
> 

```java
public class Application {
    /*
    빌더 패턴
    */
    Subway subway = new SubwayBuilder(id)
                        .menu(menu)
                        .bread(bread)
                        .topping(topping)
                        .vegetable(vegetable)
                        .sauce(sauce)
                        .set(isSet)
                        .build();
}
```

빌더 패턴으로 객체를 생성하는 시점입니다. 이렇게 호출을 하게 되면 몇 번째 인자에 어떤 값이 들어가는지 명확해져 가독성이 향상됩니다. 또한 `build()`를 실행하기 전까지 `subway` 객체가 생성되는 것이 아니기 때문에 객체의 일관성이 보장됩니다.

> 💡 Builder 패턴 네이밍 : 빌더 패턴에서 필드 설정 메서드 명은 `1.필드명` , `2.set필드명` , `3.with필드명` 등이 있다. 보통은 `1.필드명` 을 사용한다.
> 

## 결론

---

- 필드 중 불변 필드만 존재하는 경우
    - 생성자
    - 빌더 패턴
- 필드 중 불변 필드와 가변 필드가 같이 존재하는 경우
    - 빌더 패턴
- 필드 중 가변 필드만 존재하는 경우
    - 빌더 패턴

결론적으로 모든 경우에 객체를 생성하려면 빌더 패턴을 사용하면 됩니다! 하지만 필드 중 불변 필드만 존재하는 경우에서는 필드의 개수가 **적으면 일반 생성자**, **많을 경우 빌더 패턴**이 적합합니다. 최근에 `JDK14` 이상일 경우 `Record`를 통해 생성하는 것도 좋은 방법으로 알려지고 있습니다. 

## 출처

---

[SUBWAY KOREA](https://www.subway.co.kr/utilizationSubway#none)