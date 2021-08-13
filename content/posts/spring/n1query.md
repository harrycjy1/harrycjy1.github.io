---
date: '2021-08-12T11:00:18.061Z'
draft: false
title: n1query
---
> Kotlin, Spring Boot 2
- ## N+1 Query?
연관관계가 설정 되어있는 엔티티를 조회할 때 나오는 문제로써 \
조회된 엔티티의 개수 만큼(n) 엮여 있는 엔티티를 추가로 조회해오는 문제

### Example
- OneToMany로 연관관계가 맺어진 Team, Member Entity
- DB에는 3개의 Team이 존재하고 각각 6명의 Member를 가지고 있는상태

```kotlin
@Entity
class Team(
    @Id
    @GeneratedValue
    val id: Long = 0,
    @OneToMany(
        mappedBy = "team",
        orphanRemoval = true,
        cascade = [CascadeType.MERGE, CascadeType.PERSIST]
    )
    val member : List<Member> = listOf()
)

@Entity
class Member(
    @Id
    @GeneratedValue
    val id: Long = 0,

    @ManyToOne
    @JoinColumn(name = "team_id")
    val team: Team
)
```

Team을 모두 조회하는 쿼리를 실행한 결과 (JpaRepository에서 제공하는 findAll() 사용)
```
Hibernate: 
    select
        team0_.id as id1_1_ 
    from
        dev.team team0_
Hibernate: 
    select
        member0_.team_id as team_id2_0_0_,
        member0_.id as id1_0_0_,
        member0_.id as id1_0_1_,
        member0_.team_id as team_id2_0_1_ 
    from
        dev.member member0_ 
    where
        member0_.team_id=?
Hibernate: 
    select
        member0_.team_id as team_id2_0_0_,
        member0_.id as id1_0_0_,
        member0_.id as id1_0_1_,
        member0_.team_id as team_id2_0_1_ 
    from
        dev.member member0_ 
    where
        member0_.team_id=?
Hibernate: 
    select
        member0_.team_id as team_id2_0_0_,
        member0_.id as id1_0_0_,
        member0_.id as id1_0_1_,
        member0_.team_id as team_id2_0_1_ 
    from
        dev.member member0_ 
    where
        member0_.team_id=?
```

로그를 보면 team을 조회하고 member를 조회하는 쿼리가 n번(조회된 team의 개수만큼(3번)) \
즉, join을 활용한다면 한번으로 충분할 조회 쿼리가 3번이나 더 날아가고 있는걸 확인 할 수 있다.

- #### 이유?
jpaRepository내 에서 사용하는 JPQL은 엔티티와 필드이름만을 가지고 쿼리를 실행하도록 추상화 되어 있다. \
즉 [select * from entity] 만을 실행하기 때문에 연관관계인 엔티티가 필요할 경우에도 [select * from (entity)]로 호출해온다 \
(호출하는 시점은 FetchType.LAZY, EAGER에 따라 사용하는시점, 쿼리를 호출한 시점으로 나뉜다.) \
(OneToMany의 경우 default는 LAZY이지만 코드에서 member를 호출하는 내용을 추가하였다)

> 결국 조회하는 Team의 개수가 늘어날때, 연관관계가 추가될때 마다 실행하는 쿼리의 수가 기하급수적으로 늘어날 수 있다.

- #### 해결방법

1. Fetch Join을 사용한다. JPQL에서 제공하는 join fetch를 사용하면 쿼리를 최적화 할 수 있다.

(Query 작성은 Querydsl을 사용하였다.)
```kotlin
    override fun solveN1QueryProblem(): MutableList<Team> {
        return jpaQueryFactory.selectDistinct(team)
            .from(team)
            .leftJoin(team.member, member).fetchJoin()
            .fetch()
    }
    
     
```
fetch join 쿼리를 실행한 결과
```
Hibernate: 
    select
        team0_.id as id1_2_0_,
        member1_.id as id1_1_1_,
        member1_.team_id as team_id2_1_1_,
        member1_.team_id as team_id2_1_0__,
        member1_.id as id1_1_0__ 
    from
        dev.team team0_ 
    left outer join
        dev.member member1_ 
            on team0_.id=member1_.team_id
```
조인을 통해 1번의 쿼리만이 실행된걸 알 수 있다 😁

 > select distinct를 사용한 이유
: distinct를 제외하고 select 쿼리를 실행시킨 결과를 찍어보면 총 18개(3 * 6)의 team이 리턴되는데 \
이는 join fetch는 Cartesian product(카테시안 곱)이 발생하여 member의 수만큼 team이 복사되기 때문이다. \
> - Cartesian Product Link: https://byul91oh.tistory.com/26










