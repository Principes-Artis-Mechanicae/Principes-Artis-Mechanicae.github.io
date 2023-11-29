---
title: "Spring Data JPA와 QueryDSL에서 페이지네이션 최적화하기"
date: 2023-11-21 16:20:00 +09:00
categories: [QueryDSL, Java, JPA, Spring]
author: wlgns12370
tags: [querydsl, java, jpa, spring] # TAG names should always be lowercase
---

GET-P 백엔드를 개발하던 중, Spring Boot에 Pagination이 필요했습니다. 해당 기능을 구현하다 겪은 경험에 대해 정리해 보겠습니다.

먼저 Pagination(Paging)은 Client가 Server로 API 요청을 할 때, 데이터를 나누어 받아오는 것입니다. 크게 Size와 Offset으로 나눌 수 있습니다.

`/?page={PageData}&size={SizeData}` : 쿼리 형태

`Limit` or `PageSize` : 한 페이지에 보여줄 데이터 수

`offset`(index) : 데이터가 시작하는 위치

다음 코드는 GET-P 플랫폼에 등록된 프로젝트 중 프로젝트의 조건에 따라 데이터베이스로 쿼리를 보내는데, 이때 데이터를 필요한 만큼 받아오는 `findFilteredProjectPage` 메소드 입니다.

```java
public Page<Project> findFilteredProjectPage(ProjectStatus projectStatus,
        Long applicationDeadlineOffset, ProjectOrder projectOrder, Pageable pageable) {

    // 검색 조건에 맞는 프로젝트를 가져옵니다.
    List<Project> content = queryFactory.selectFrom(project)
            .where(projectStatusEq(projectStatus), applicationDeadlineLoe(applicationDeadlineOffset))
            .orderBy(projectOrderSpecifier(projectOrder))
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .fetch();

    // 검색 조건에 맞는 프로젝트의 총 개수를 가져옵니다.
    long total = queryFactory.selectFrom(project)
            .where(projectStatusEq(projectStatus), applicationDeadlineLoe(applicationDeadlineOffset))
            .fetchCount();

    // Page 객체를 생성하여 반환합니다.
    return new PageImpl<>(content, pageable, total);
}
```

처음에는 위와 같이 `PageImpl` 을 사용해 페이징 처리를 했습니다. 그러나 

- **i) 첫 페이지가 마지막 페이지인 경우(ContentSize < Total)**
- **ii) 마지막 페이지의 데이터를 조회한 경우**
i와 ii의 경우 총개수를 구하는 쿼리를 실행할 필요가 없습니다.

위 문제는 `PageableExecutionUtils`를 통해 해결할 수 있습니다. `PageableExecutionUtils` 은 i와 ii의 경우에는 프로젝트의 총개수를 구하는 쿼리를 실행하지 않습니다.

다음 터미널은 GET-P의 프로젝트를 조회하던 중 ii 경우의 예시입니다. 전체 데이터가 20개인데 `/projects?page=1&size=10` 를 요청한 상황입니다. `page`는 0부터 시작하기 때문에 `page=1` 일 때, 마지막 페이지라는 것을 알 수 있습니다. 하지만 PageImpl을 사용했기 때문에 다음과 같은 카운트 쿼리가 발생하였습니다.

![Untitled](https://github-production-user-asset-6210df.s3.amazonaws.com/30788586/284499474-63f5f5f7-4298-4495-8131-5b8de09bdd65.png)

`PageImpl` 을 `PageableExecutionUtils`으로 변경한 코드입니다. 변경 이후 `findFilteredProjectPage` 에 content와 countQuery의 가독성을 높이기 위해 메서드로 분리하였습니다.

```java

private List<Project> getProjectContent(ProjectStatus projectStatus,
        Long applicationDeadlineOffset, ProjectOrder projectOrder, Pageable pageable) {
    return queryFactory.selectFrom(project)
            .where(projectStatusEq(projectStatus), applicationDeadlineLoe(applicationDeadlineOffset))
            .orderBy(projectOrderSpecifier(projectOrder)).offset(pageable.getOffset())
            .limit(pageable.getPageSize()).fetch();
}

private JPAQuery<Long> getProjectCountQuery(ProjectStatus projectStatus, Long applicationDeadlineOffset) {
    return queryFactory.select(project.count()).from(project).where(projectStatusEq(projectStatus),applicationDeadlineLoe(applicationDeadlineOffset));
}

public Page<Project> findFilteredProjectPage(ProjectStatus projectStatus,
        Long applicationDeadlineOffset, ProjectOrder projectOrder, Pageable pageable) {
    List<Project> content = getProjectContent(projectStatus, applicationDeadlineOffset, projectOrder, pageable);
    JPAQuery<Long> countQuery = getProjectCountQuery(projectStatus, applicationDeadlineOffset);
    return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
}

```

다음 터미널은 `PageImpl` 을 `PageableExecutionUtils`으로 변경한 뒤, ii 경우의 예시입니다. 위와 같이 `/projects?page=1&size=10` 를 요청하였는데 카운트 쿼리 없이 마지막 content를 반환합니다.

![Untitled](https://github-production-user-asset-6210df.s3.amazonaws.com/30788586/284499482-bf3af2f4-68a5-41d8-bdb8-04740bc7d691.PNG)

결론적으로,**`PageableExecutionUtils`** 클래스를 사용하면 count 쿼리를 최적화할 수 있었습니다. 이를 통해, 애플리케이션의 성능을 향상시킬 수 있습니다.
