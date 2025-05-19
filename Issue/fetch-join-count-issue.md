# Paging 기능 사용 중, 전체 개수가 join으로 인해 일치하지 않은 문제

## 문제 상황

페이징 기능을 위해 total 값을 QueryDSL로 `workItem`과 `workAssignees`를 `LEFT JOIN` 하여 리스트를 조회하던 중,  
`fetch().size()`로 개수를 확인했을 때 예상보다 큰 값이 반환되는 이슈 발생.

```java
long count = queryFactory.selectDistinct(workItem)
    .from(workItem)
    .leftJoin(workAssignees).on(workItem.id.eq(workAssignees.workItemId))
    .where(
      조건
    )
    .fetch()
    .size();
```

## 문제 발생 이유

이 부분은 `selectDistinct`를 해서 가져오면 id를 기준으로 `distinct` 처리할 것으로 생각
그런데 `left join`을 통한 `id` 중복이 처리되지 않으면서 개수가 중복된 개수가 처리됨

## 해결 방안

- `selectDistinct` 가 아닌 `countDistinct`를 활용하는 것으로 변경 진행
- 이 부분에 대해서 찾아본 결과 `selectDistinct`의 경우에는 데이터 전체를 가져온 뒤에 개수를 세면서 비효율적
- 대신 `countDistinct`를 이용하는 것을 통해 DB 내부에서 id 값을 이용해 `count`처리를 직접하기 때문에 효율적

### 변경 코드

```java
long count = queryFactory.select(workItem.id.countDistinct())
    .from(workItem)
    .leftJoin(workAssignees).on(workItem.id.eq(workAssignees.workItemId))
    .where(
            조건
    )
    .fetchOne();
```
