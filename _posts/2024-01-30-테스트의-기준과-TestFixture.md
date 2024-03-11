---
title: "테스트의 기준과 TestFixture"
date: 2024-01-11 12:30:00 +09:00
categories: [Java, Test, TestFixture]
author: wlgns12370
tags: [Java, Test, TestFixture]
---

GET-P 서버 테스트를 하기 위해서 메서드마다 테스트하는 **단위테스트**로 JUnit을 사용하고 있었습니다. 저에게 서버라 함은 “견고함”이 중요했습니다. 하지만 어느정도로 테스트를 해야할까?라는 의문은 지속 되었습니다. 주변에 쿠팡 백엔드 개발자인 분에게 질문을 해보았습니다. 예전 우아한형제들 포비님께 “테스트는 마음의 안정감이 드는데 까지”라는 말씀을 해주셨다고 합니다. 이후 서핑을 통해 논문에서도 70%~80%정도 커버하는게 일반적인 사실임을 알게 되었습니다.

## 테스트의 기준

---

“70% ~ 80%를 커버하자”는 기준은 기본적으로 CRUD에 모든 메서드는 테스팅을 하고, 핵심기능을 책임지는 클래스 위주로 테스팅 케이스를 많이 작성하자!라는 기준을 세웠습니다.

## 테스트 픽스쳐 개념

---

단위 테스트 코드를 작성하던 중, 테스트 데이터가 중복되어 매우 불편했습니다.  개발을 할때, DTO 자체에 정적 팩토리 메서드를 만들어 Entity를 만드는 책임을 `Service`에서 `DTO`로 위임 하는 방식이 떠올랐습니다.  이 방법은 객체를 생성하는 중복되는 코드나 래퍼 메서드를 줄여주어서 너무 매력적으로 느꼈습니다.  이후 테스트에도 똑같이 적용해도 되지 않을까? 라는 생각에 인터넷 서핑을 하다가 [Junit에서 언급하는 Fixture에 대한 개념 글](https://junit.org/junit4/cookbook.html)을 읽었습니다.

> 💡 Tests need to run against the background of a known set of objects. This set of objects is called a **test fixture**. When you are writing tests you will often find that you spend more time writing the code to set up the fixture than you do in actually testing values.
> 

Test Fixture는 테스트 데이터 셋이 중복될때, 하나의 변수(집합)로 묶어주는 역할을 합니다. 이 또한 중복되는 코드를 줄이는데 매력적인 방법이라고 느껴 공부해보았고, GET-P 서버에 적용한 부분을 설명하면서 더 자세하게 알아보겠습니다.

### Test Fixture 사용 전 TestCode

```java
class test {
    Client testClient = Client.builder()
        .name("겟피")
        .email("getp@princip.es")
        .phoneNumber("010-1234-5678")
        .profileImageUri("https://he.princip.es/img.jpg")
        .address("대구광역시")
        .accountNumber("3332-112-12-12")
        .member(testMember)
        .build();

    CreateClientRequest testCreateClientRequest = new CreateClientRequest(
            "겟피",
            "getp@princip.es",
            "010-1234-5678",
            "https://he.princip.es/img.jpg",
            "대구광역시",
            "3332-112-12-12"
        );

    UpdateClientRequest testUpdateClientRequest = new UpdateClientRequest(
            "겟피.Update", ...
        );

    void testCreate() {
        when(clientRepository.save(any(Client.class))).thenReturn(testClient);

        Client createdClient = clientService.create(testMember, testCreateClientRequest);

        assertSoftly(softly -> {
            ...
        });
    }

    void testUpdate() {
        when(clientRepository.save(any(Client.class))).thenReturn(any(Client.class));
        clientService.create(testMember, testCreateClientRequest);
        when(clientRepository.findByMember_MemberId(testMember.getMemberId())).thenReturn(Optional.of(testClient));

        Client updatedClient = clientService.update(testMember.getMemberId(), testUpdateClientRequest);

        assertSoftly(softly -> {
            ...
        });
    }
}

class anotherTest {
    Client testClient = Client.builder()
        .name("겟피")
        .email("getp@princip.es")
        .phoneNumber("010-1234-5678")
        .profileImageUri("https://he.princip.es/img.jpg")
        .address("대구광역시")
        .accountNumber("3332-112-12-12")
        .member(testMember)
        .build();

		...
}
```

CRUD 많은 복잡한 메서드 중 간단하게 Create와 Update 메서드로 설명하겠습니다. 먼저 `Client`, `CreateClientRequest`, `UpdateClientRequest` 라는 테스트 데이터가 필요합니다. `Client`와 `CreateClientRequest`는 `member`객체의 유무만 다릅니다. 나머지 필드는 중복되기 때문에 **static 변수**로 묶을 수 있습니다. 또한 `anotherTestClass`에서 현재 `testClass`의 테스트 데이터가 겹쳤습니다. 현재는 설명하기 위해 2개의 메서드만 적어두어서 그렇지 실제로 수십개의 메서드가 만들어지면 중복되는 코드수가 엄청났습니다.

## 테스트 픽스쳐 적용

---

Test Fixture를 적용하는 방식에는 크게 두가지가 있습니다.

- **@BeforeEach**
- **FixtureClass 생성**

두 가지의 방법 중 위에서 언급한 클래스마다 중복되는 코드를 줄여 재사용성을 높이려면, `**FixtureClass**`를 권장합니다. 

### ClientFixture

```jsx
public class ClientFixture {
    public static String NAME = "겟피";
    public static String EMAIL = "getp@princip.es";
    public static String PHONE_NUMBER = "010-1234-5678";
    public static String PROFILE_IMAGE_URI = "https://he.princip.es/img.jpg";
    public static String ADDRESS = "대구광역시 북구";
    public static String ACCOUNT_NUMBER = "3332-112-12-12";
    public static String UPDATED_NAME = "겟피.Update";

    public static CreateClientRequest createClientRequest() {
        return new CreateClientRequest(NAME, EMAIL, PHONE_NUMBER, NAME, EMAIL, ACCOUNT_NUMBER);
    }

    public static UpdateClientRequest updateClientRequest() {
        return new UpdateClientRequest(UPDATED_NAME, EMAIL, PHONE_NUMBER, NAME, EMAIL, ACCOUNT_NUMBER);
    }

    public static Client createClientByMember(Member member) {
        return Client.builder()
                        .name(NAME)
                        .email(EMAIL)
                        .phoneNumber(PHONE_NUMBER)
                        .profileImageUri(PROFILE_IMAGE_URI)
                        .address(ADDRESS)
                        .accountNumber(ACCOUNT_NUMBER)
                        .member(member)
                    .build();
    }
}
```

`FixtureClass`를 잘 활용하는 방법은 **사용 되어야 할 값들을 static 변수로 선언해줍니다.** 클래스 내부에서 메서드 간 중복되는 데이터를 한번에 수정하기 위함입니다. 다음으로 **정적 팩토리 메서드로 필요한 생성 메서드를 구현합니다.** 이 방법은 `ClientFixture` 인스턴스를 생성한 뒤, 테스트 데이터 생성 메서드 호출할 필요가 없고, 즉각적으로 테스트 데이터 생성 메서드를 호출할 수 있어 장점을 가집니다.

### Test Fixture 사용 후 TestCode

```java
class test {
    private final Member testMember = MemberFixture.createMember();
    private final CreateClientRequest testCreateClientRequest = ClientFixture.createClientRequest();
    private final Client testClient = ClientFixture.createClientByMember(testMember);

    void testCreate() {
        when(clientRepository.save(any(Client.class))).thenReturn(testClient);

        Client createdClient = clientService.create(testMember, testCreateClientRequest);

        assertSoftly(softly -> {
            ...
        });
    }

    void testUpdate() {
		private final UpdateClientRequest testUpdateClientRequest = ClientFixture.updateClientRequest();
        when(clientRepository.save(any(Client.class))).thenReturn(any(Client.class));
        clientService.create(testMember, testCreateClientRequest);
        when(clientRepository.findByMember_MemberId(testMember.getMemberId())).thenReturn(Optional.of(testClient));

        Client updatedClient = clientService.update(testMember.getMemberId(), testUpdateClientRequest);

        assertSoftly(softly -> {
            ...
        });
    }
}

class anotherTest{
    private final Member testMember = MemberFixture.createMember();
    private final CreateClientRequest testCreateClientRequest = ClientFixture.createClientRequest();
    private final Client testClient = ClientFixture.createClientByMember(testMember);

    ...
}
```

`Test Fixture`에서 필요한 부분을 테스트가 필요한 `Class`에 넣어서 사용할 수 있습니다. 이전 코드보다 훨씬 깔끔해졌고, 재사용성을 높였습니다. 

## 느낀점

---

이번에 `Test Fixture`을 공부하면서 느낀 점은 **책임의 분리의 중요성입니다.** 항상 개발을 하다가 중복되는 코드를 리팩터링 하는 작업에서 고난을 겪고 새로운 패턴을 학습하게 되었는데, 정적 팩토리 메서드와 ****`Class`의 책임을 누구에게 위임하는가에 따라 **좋은 코드와 나쁜 코드**로 나누어진다고 느꼈습니다. 이후 개발을 하게 되면 30분이 걸리던 일이 10분 만에 완벽하게 해결되고 팀원과 **코드 리뷰를 하는 과정에서도 가독성이 높아져** Fixture에 소중함을 알게 된 경험이었습니다.