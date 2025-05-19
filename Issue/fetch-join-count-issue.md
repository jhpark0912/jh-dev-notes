# Paging 기능 사용 중, 전체 개수가 join으로 인해 일치하지 않은 문제

## 문제 상황

페이징 기능을 위해 total 값을 QueryDSL로 `workItem`과 `workAssignees`를 `LEFT JOIN` 하여 리스트를 조회하던 중,  
`fetch().size()`로 개수를 확인했을 때 예상보다 큰 값이 반환되는 이슈 발생.

```java
long count = queryFactory.selectDistinct(workItem.id)
    .from(workItem)
    .leftJoin(workAssignees).on(workItem.id.eq(workAssignees.workItemId))
    .where(
      조건
    )
    .fetch()
    .size();
```
