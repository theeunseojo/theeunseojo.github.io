---
layout: post
title: "[JPA] 값 타입 컬렉션(@ElementCollection)의 List vs Set 동작 원리 및 쿼리 분석"
date: 2025-12-05
category: JPA , Spring Boot, TroulbeShooting
---

# 문제상황
- JPA 스터디 중 스터디원이 낸 값 타입 컬렉션 관련 문제에서 혼동이 생겼다.

- "자바 ORM 표준 JPA 프로그래밍"책에서는 값 타입 컬렉션은 데이터 변경 시 관련된 데이터를 모두 삭제하고, 남은 데이터를 다시 삽입한다는 내용이 나오기 때문이다.

```java

@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ElementCollection
    @CollectionTable(name = "favorite_food",
            joinColumns = @JoinColumn(name = "member_id"))
    @Column(name = "food_name")
    private Set<String> favoriteFoods = new HashSet<>();
}


// 문제 : SQL은 몇 번 실행되는가?(1차 캐시에 없다는 전제하)
favoriteFoods = { "Pizza", "Chicken", "Sushi" }   // 총 3개

Member member = em.find(Member.class, 1L);

member.getFavoriteFoods().remove("Chicken");

em.flush();
```

- 책에서 배운대로 전체 삭제 후 남은 데이터 재등록이 발생할 것이라는 가정하에 진행했다면
1. 멤버 조회 SELECT
2. 컬렉션 조회 SELECT
3. 전체 삭제 DELETE ALL
4. 데이터 삽입 INSERT
5. 데이터 삽입2 INSERT
- 총 5번이 나가야한다다.

- 하지만 문득 Set은 중복을 허용하지 않은 자료구조이고, 키값이 식별자 아닌가 하는 생각을 하게 되었다.


# 실험 및 검증 
- list와 set으로 했을 경우 실제 쿼리는 어떨까?를 테스트해봤다.

## List
<img src="blog/public/img/jpa_값타입컬렉션_list.png">

- 예상대로 전체 삭제를 진행 한 후, 남은 데이터를 삽입해준다. 

## Set
<img src="blog/public/img/jpa_값타입컬렉션_set.png">

- 반면 Set을 사용했을 때는 전체 삭제가 이루어지지 않았다. 정확히 해당데이터를 하나만 삭제했기 때문이다.

# 결론
- 값 타입 컬렉션 사용 시 발생하는 성능 이슈의 원인은 '식별자의 부재'때문이다.
- 그렇기 때문에 값타입 컬렉션을 사용하는 것 보다는 엔티티를 사용해서 식별자를 만들어줄 수 있도록 일대다 엔티티 관계를 사용한다고 한다. 
