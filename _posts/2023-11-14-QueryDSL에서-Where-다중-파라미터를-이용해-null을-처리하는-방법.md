---
title: "QueryDSL에서 Where 다중 파라미터를 이용해 null을 처리하는 방법"
date: 2023-11-14 20:27:00 +09:00
categories: [Jekyll]
author: scv1702
tags: [jekyll] # TAG names should always be lowercase
---

GET-P 프로젝트를 진행하던 중, 프론트엔드에서 요청된 쿼리스트링에 따라 필터링을 하는 기능을 구현해야 됐었다. 해당 기능을 구현하는데 있어서 겪은 경험에 대해 정리해보았다.

다음 코드는 GET-P 플랫폼에 등록된 프로젝트 중 인자로 넘어온 프로젝트 상태, 지원 마감일에 따라 데이터베이스로 쿼리를 보내는 `findFiltteredProjects` 메소드이다.

```java
public List<Project> findFilteredProjects(ProjectStatus projectStatus,
            Long applicationDeadlineOffset) {
        BooleanBuilder booleanBuilder = new BooleanBuilder();
        if (projectStatus != null) {
            booleanBuilder.and(project.status.eq(projectStatus));
        }
        if (applicationDeadlineOffset != null) {
            LocalDate applicationDeadline = LocalDate.now().plusDays(applicationDeadlineOffset);
            booleanBuilder.and(project.applicationDeadline.loe(applicationDeadline));
        }
        JPAQuery<Project> query = queryFactory.selectFrom(project).where(booleanBuilder);
        return query.fetch();
    }
```

처음에는 위와 같이 `BooleanBuilder`를 이용해 각 변수가 가지고 있는 값에 따라 동적으로 쿼리문을 만들어 내었다. 그러나 다만, 각 변수가 가지고 있는 값을 확인을 해야 하기 때문에 `if` 문이 반복될 수 밖에 없었다. 이는 코드의 가독성을 해치고, 메소드가 한 가지 일을 하는 것이 아닌 여러 가지 일을 하게 된다.

위 문제는 QueryDSL의 `Where` 문의 다중 파라미터를 이용해 해결할 수 있다. `Where` 문에 `null`이 들어가게 되면 해당 변수에 대한 쿼리는 무시된다는 것을 이용해, `null` 값을 확인하는 로직을 메소드로 분리하고 마지막 `selectFrom`의 `Where` 문에 메소드들의 결과값을 파라미터로 넘기는 것이다.

먼저 `projectStatus`가 `null` 인지 아닌지 판단하는 로직을 보자.
```java
if (projectStatus != null) {
    booleanBuilder.and(project.status.eq(projectStatus));
}
```
위 코드 중 `project.status.eq(projectStatus)`는 데이터베이스에 저장된 `Project` 엔티티들 중 `projectStatus`와 같은 값을 가지고 있는 `Project` 만을 가져올 수 있도록 하는 `status = :projectStatus`라는 불린 표현식을 만들어 낸다. 정리하자면, 위 코드는 `status = :projectStatus`라는 `BooleanExpression`을 만들어내어 `BooleanBuilder`에 담는 일을 한다.

이는 다음과 같이 변경할 수 있다.

`if` 문을 메소드로 분리해 `projectStatus`가 `null`인 경우 `null`을 반환하고, 그렇지 않은 경우 `BooleanExpression`을 반환하도록 하는 메소드인 `projectStatusEq` 메소드를 선언한다.

```java
private BooleanExpression projectStatusEq(ProjectStatus projectStatus) {
    return projectStatus != null ? project.status.eq(projectStatus) : null;
}
```
단순한 `null` 검사 만을 하기 때문에 삼항 연산자를 사용했지만, 추가적인 로직이 필요한 경우 삼항 연산자는 피해야 한다고 생각한다.

두 번째는 지원 마감일에 대한 쿼리인데, 현재 날짜를 기준으로 지원 마감일이 `applicationDeadlineOffset`과 같거나 적게 남은 `Project`를 검색하는 쿼리다.
```java
if (applicationDeadlineOffset != null) {
    LocalDate applicationDeadline = LocalDate.now().plusDays(applicationDeadlineOffset);
    booleanBuilder.and(project.applicationDeadline.loe(applicationDeadline));
}
```

이 또한 비슷한 방법으로 `applicationDeadlineLoe` 라는 메소드를 선언한다.

```java
private BooleanExpression applicationDeadlineLoe(Long applicationDeadlineOffset) {
    if (applicationDeadlineOffset == null) {
        return null;
    }
    LocalDate applicationDeadline = LocalDate.now().plusDays(applicationDeadlineOffset);
    return project.applicationDeadline.loe(applicationDeadline);
}
```

그리고 이들을 `queryFactory.selectFrom`의 `Where` 문의 파라미터로 넘긴다.

```java
JPAQuery<Project> query = queryFactory.selectFrom(project).where(
            projectStatusEq(projectStatus), applicationDeadlineLoe(applicationDeadlineOffset));
```

이를 최종적으로 정리하면 다음과 같다.

```java
private BooleanExpression projectStatusEq(ProjectStatus projectStatus) {
    return projectStatus != null ? project.status.eq(projectStatus) : null;
}

private BooleanExpression applicationDeadlineLoe(Long applicationDeadlineOffset) {
    if (applicationDeadlineOffset == null) {
        return null;
    }
    LocalDate applicationDeadline = LocalDate.now().plusDays(applicationDeadlineOffset);
    return project.applicationDeadline.loe(applicationDeadline);
}

public List<Project> findFilteredProjects(ProjectStatus projectStatus,
        Long applicationDeadlineOffset) {
    JPAQuery<Project> query = queryFactory.selectFrom(project).where(
            projectStatusEq(projectStatus), applicationDeadlineLoe(applicationDeadlineOffset));
    return query.fetch();
}
```

처음 코드에 비해 가독성이 훨씬 좋아졌고, `findFiltteredProjects` 메소드가 한 가지 일만 하게 되었다. 또한 다른 메소드에서 해당 변수들에 대한 쿼리가 있는 경우 메소드를 재활용할 수 있게 되었다.