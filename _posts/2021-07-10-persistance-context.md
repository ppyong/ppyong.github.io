---
layout: post
title: JPA 정리
---

## 영속성 컨텍스트

> 엔티티를 영구 저장하는 환경이라는 뜻. 엔티티 매니저를 생성할 때 하나 만들어지고, 매니저를 통해서 영속성 컨텍스트에 접근할 수 있다. 

## 엔티티 생명 주기 

- 비영속(new/transient)

>  엔티티 객체를 생성 후 아직 저장하지 않은 상태. 영속성 컨텍스트나 데이터베이스와는 전혀 관련이 없음

- 영속(managed)

> 엔티티 매니저를 통해서 엔티티를 영속성 컨텍스트에 저장했다. 영속성 컨텍스트가 관리하는 상태
> em.find() 나 JPQL을 사용해 조회한 엔티티도 영속성 컨텍스트가 관리하는 영속 상태 
> 영속 상태는 식별자 값이 반드시 있어야 한다 

- 준영속(detached)

> 영속성 컨텍스트가 관리하던 영속 상태의 엔티티를 영속성 컨텍스트가 관리하지 않으면 준영속 상태가 된다. 
> em.close(), em.detach(), em.clear() 를 호출해서 영속성 컨텍스트를 초기화해도 영속성 컨테스트가 관라헌 영속 상태의 엔티티는 준영속 상태가 된다. 

> - 거의 영속화 상태에 가깝다.    
> \- 1차 캐시, 쓰기 지연, 변경 감지, 지연 로딩 등 영속성 컨텍스트가 제공하는 어떠한 기능도 동작하지 않는다.
> - 식별자 값을 가지고 있다.     
> \- 비영속 상태는 식별자 값이 없을 수도 있지만 준영속 상태는 이미 한번 영속 상태였으므로 반드시 식별자 값을 가지고 있다. 
> - 지연 로딩을 할 수 없다.     
> \- 준영속 상태는 영속성 컨텍스트가 더는 관리하지 않으므로 지연 로딩 시 문제가 발생한다. 

- 삭제 (removed)

> 엔티티를 영속성 컨텍스트와 데이터베이스에서 삭제한다. 

## 일대다 단방향 [1:N]

이 매핑은 반대쪽 테이블에 있는 외래키를 관리한다. 일대다 관계에서 외래 키는 항상 다쪽 테이블에 있다. 하지만 다 쪽인 Member 엔티티에는 외래 키를 매핑할 수 있는 참조 필드가 없다.

```java
@Entity 
public class Team {
    @Id @GeneratedValue
    @Column(name="TEAM_ID")
    private Long id;

    private String name; 

    @OneToMany
    @JoinColumn(name="TEAM_ID")
    private List<Member> mebers = new ArrayList<>();
}

@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name="MEMBER_ID")
    private Long id;
    
    private String username;
}
```

- 일대다 단방향 매핑의 단점    
\- 매핑한 객체가 관리하는 외래 키가 다른 테이블에 있다는 점이다. 본인 테이블에 외래 키가 있으면 엔티티의 저장과 연관 관계 처리를 INSERT SQL 한 번으로 끝낼 수 있지만, 다른 테이블에 외래 키가 있으면 연관 관계 처리를 위해 UPDATE SQL을 추가로 실행해야 한다.

## 식별 관계

- 식별 관계     
\- 식별 관계는 부모 테이블의 기본 키를 내려받아서 자식 테이블의 기본 키 + 외래 키로 사용하는 관계다.

<img src="https://ppyong.github.io/assets/img/identifying.jpg" width="60%">

- 비식별 관계    
\- 비식별 관계는 부모 테이블의 기본 키를 받아서 자식 테이블의 외래 키로만 사용하는 관계다.

<img src="https://ppyong.github.io/assets/img/non-identify.jpg" width="60%">

## 복합 키

- @IdClass   

```java
@IdClass(ParentId.class)
@Entity
public class Parent { 
    @Id
    @Column(name="PARENT_ID1")
    private String id1; 

    @Id
    @Column(name="PARENT_ID2")
    private String id2;

    private String name;
}

public class ParentId implements Serializable { 

    private String id1;


    private String id2;
}
```    

> @IdClass 를 사용할 때 식별자 클래스는 아래 조건을 만족해야 한다.    
>- 식별자 클래스의 속성명과 엔티티에서 사용하는 식별자의 속성명이 같아야 한다.    
>- Serializable 인터페이스를 구현해야 한다.     
>- equals, hashCode를 구현해야 한다.    
>- 기본 생성자가 있어야 한다.     
>- 식별자 클래스는 public이어야 한다.    

&nbsp; 

- @EmbaeddedId   

```java
@Entity
public class Parent { 
    @EmbaddedId
    private ParentId id; 

    private String name;
}

@Embeddable
public class ParentId implements Serializable { 
    @Column(name="PARENT_ID1")
    private String id1;

    @Column(name="PARENT_ID2")
    private String id2;
}
```    
    
> @EmbaddedId를 적용한 식별자 클래스는 아래 조건을 만족해야 한다.    
>- 식별자 클래스의 속성명과 엔티티에서 사용하는 식별자의 속성명이 같아야 한다.   
>- Serializable 인터페이스를 구현해야 한다.     
>- equals, hashCode를 구현해야 한다.    
>- 기본 생성자가 있어야 한다.     
>- 식별자 클래스는 public이어야 한다.    

&nbsp; 