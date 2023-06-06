---
layout: post
title:  "[Project] LightUp 트러블 슈팅"
date:   2023-05-31 20:51:36 +0900
categories: Projects
---

# [Project] LightUp 트러블 슈팅

Light Up 백엔드를 구축하며 마주했던 문제들을 정리한다.  
지속적으로 추가할 예정!

| Spring Boot 3.0.6, MySQL 8.0

<!-- **목차** 
</br>
[JPA 연관관계 무한참조 hotfix](#jpa-연관관계-무한참조-hotfix) 
</br>
[JPA 컬렉션 형태의 컬럼 처리](#jpa-컬렉션-형태의-컬럼-처리) 
</br>
[연관관계 갖는 엔티티의 dto 저장](#연관관계-갖는-엔티티의-dto-저장) 
</br> -->

## JPA 연관관계 무한참조 hotfix

### 개요

image를 업로드할 때 caption 필드 제외하고 나머지 image정보들만 업로드하는 방식을 우선 구현했다. 

다음으로

1. 등록된 image의 idx로 image 객체 찾아와 caption을 추후에 add
2. image를 처음 업로드 할 때 caption도 같이 저장

을 구현하는데 포스트맨에 json을 작성하다보니 서로 무한참조가 되겠더라.

현재 image post 리퀘를 날릴 때 다음 imageDto를 사용한다.

보면 caption필드가 List<Caption> captions; 로 되어있다. caption 객체들의 리스트인거다. 

이제보니 심지어 dto는 만들어놓고 안쓰고 날것의 caption 객체로 해뒀다.  

```java
@Getter
@NoArgsConstructor
public class ImageDto {

    private Long idx;
    private Long memberIdx;
    private String savedPath;
    private List<Caption> captions;
    private Gps gps;

    @Builder
    public ImageDto(Long idx, Long memberIdx, String savedPath, List<Caption> captions, Gps gps) {
        this.idx = idx;
        this.memberIdx = memberIdx;
        this.savedPath = savedPath;
        this.captions = captions;
        this.gps = gps;
    }
    
    // 사진만 미리 저장해두고 후에 캡션 추가하는 경우
    @Builder ImageDto(Long idx, Long memberIdx, String savedPath, Gps gps) {
        this.idx = idx;
        this.memberIdx = memberIdx;
        this.savedPath = savedPath;
        this.gps = gps;
    }
    
    // update시 수정할 요소가 뭐가 있을지에 대한 고민 필요 - 관리자 - 캡션, 경로 ...
    @Builder
    public ImageDto(List<Caption> captions) {
        this.captions = captions;
    }

    public Image toEntity(Member member) {

        return Image.builder()
                .member(member)
                .gps(gps)
                .savedPath(savedPath)
                .captions(captions)
                .build();
    }
}
```

caption은 아래와 같이 생겼다.

```java
public class Caption {

    @Id
    @Column(name = "CAPTION_IDX")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idx;

    @ManyToOne
    @JoinColumn(name = "IMAGE_IDX")  // img 외래키를 가짐 -> 연관관계 주인
    private Image image;

    @Column(nullable = false)
    private String originalCaption;

    @Column(nullable = false)
    @ElementCollection
    private Set<String> dangerFactor;  // 중복될 일 없으니 set로 지정 // !! danger Factor field 생성 안되는 이슈 !!

    public void update(String originalCaption, Set<String> dangerFactor) {
        this.originalCaption = originalCaption;
        this.dangerFactor = dangerFactor;
    }
} 
```

그럼 json image POST request를 날릴 때 서로 무한참조를 한다.

### 해결

~~사실 만들어둔 Dto로만 바꿔줘도 ..? 어찌보면 typo로 인한 오류..~~  

List<CaptionDto>로 사용하는 로직 → 클라에서 리퀘시 CaptionDto 각 필드들 다 입력해줘야함 → 비효율

ImageDto에 List<CaptionDto> 말고 List<String> 으로 처리시 ImageDto를 toEntity할 때 각각의 리스트 구성요소를 CaptionEntity로 toEntity하는 로직 추가 필요 

1. ImageDto로 post 요청이 들어올 때 다음 필드들을 넘겨준다.
    - Long idx → 자동생성이므로 제외
    - Long memberIdx
    - String savedPath
    - List<String> captions !!
    - Gps gps.latitude, gps.longitude
    
    → 이 때, 캡션을 동시에 넘겨받는 경우, 캡션 없이 이미지만 등록하는 경우로 나뉜다. 후자는 이미 구현했으므로 패스.
    
2. 캡션을 이미지와 동시에 받아와 Post하는 경우, api의 imageCreate → service의 save로직을 따른다. save(imageDto) → 우선 imageDto.getMemberIdx로 어느 회원이 업로드한 이미지인지 파악, 인덱스로 회원 객체 member를 만들어 imageDto.toEntity(member)로 넘겨주어 image 객체를 db에 저장한다. 
3. 추가할 로직은 
1. service단에서 save로직 중 List<String> captions를 분리해 → imageDto.getCaptions() 구현 필요
2. getCaptions가 반환한 리스트에서 각각의 데이터 (캡션 하나하나) 를 Caption 엔티티(Dto??하지만 굳이?)로 만들어 동시에 저장하는 로직 필요  → caption 객체든 Dto든 imageIdx가 필드에 들어가는데 해당하는 이미지가 등록되기 전이다.. 어쩌지? 이미지만 먼저 등록해서 imageIdx 만들어두고 뒤에 추가하는 방식? 

### 믿고 있었다 JPA

[[Java] Spring Boot - JPA 여러 Insert를 하나로 묶어 한번에 처리하는 방법](https://m.blog.naver.com/seek316/222012420626)

- 결론적으로 위에 추가할 로직들을 stream으로 처리했다.

## JPA 컬렉션 형태의 컬럼 처리

### 개요

Caption 도메인은 dangerFactor 라는 Set 타입의 컬럼을 갖는다.

JPA 매핑하려 `@Column`을 붙여줬더니 안되었다. (에러가 뭐였는지는 기억 안난다.)

### 해결

DB는 컬렉션과 같은 형태의 데이터를 컬럼에 저장할 수 없다고한다. 

별도의 테이블을 생성하여 컬렉션을 관리해야한다.

이때 컬렉션 객체임을 JPA에게 알려주는 어노테이션이 `@ElementCollection`

이다. 즉, `@Entity`가 아닌 Basic Type이나 Embeddable Class로 정의된 컬렉션을 매핑하는 방법이다. 

그럼 여기서 `@OneToMany` 와 `@ElementCollection` 은 뭐가 다르냐, `@ElementCollection` 의 경우 CASCADE = ALL 로 생각하면 된다. 

그러니까 JPA에서 entity는 각자의 독립적인 life cycle을 가진다.

근데 `@ElementCollection` 을 붙인 항목은 entity가 아닌 값 타입(Basic Type)의 모음이라고 선언되는 것이다. 따라서 CASCADE = ALL 이 되는거다.

반면, `@OneToMany` 를 통해서는 다른 객체와 일대다 연관관계를 맺는다. 이걸 붙인 항목 또한 entity이기에 별도의 life cycle을 가지고 있는 것이다.

## 연관관계 갖는 엔티티의 DTO 저장

### 개요

Caption 엔티티는 이렇게 생겼다. Image 엔티티와 일대다 연관관계를 갖는다.

```java
public class Caption {

    @Id
    @Column(name = "CAPTION_IDX")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idx;

    @ManyToOne
    @JoinColumn(name = "IMAGE_IDX") 
    private Image image;

    @Column(nullable = false)
    private String originalCaption;

    @Column(nullable = false)
    @ElementCollection
    private Set<String> dangerFactor;  
}
```

Caption 엔티티를 저장할 때 CaptionDto를 이렇게 작성했었다. 

```java
public class CaptionDto {

    private Long idx;
    private Image image;
    private String originalCaption;
    private Set<String> dangerFactor; 
}
```

그런데 이 경우 postman으로 리퀘를 날리기 위해서는 Caption과 연관관계 매핑이 되어있는 Image 엔티티 값들까지 모두 입력해야한다. 

이러면 비효율적이고 클라에서 불필요하게 추가로 API를 호출해야할 가능성이 생길 여지가 있다고 생각해서 Image 식별자인 idx값만 입력해 리퀘를 보내도록 수정했다.

### 해결

우선 dto의 Image를 imageIdx로 수정했다.

```java
public class CaptionDto {

    private Long idx;
    private Long imageIdx;  // 전체 객체 요청을 하므로 전체 img 객체가 아닌 id값만 가져와 매핑하도록
    private String originalCaption;
    private Set<String> dangerFactor;

		public Caption toEntity(Image image) {

        return Caption.builder()
                .image(image)
                .originalCaption(originalCaption)
                .dangerFactor(dangerFactor)
                .build();
    }
}
```

그리고 리퀘에 들어온 imageIdx을 기반으로 service단에서 해당 image 객체를 찾아오고 그 객체를 직접 파라미터로 넣어서 caption을 엔티티화하는 toEntity 메서드도 만들어줬다.

그러려면 captionService에서 imageRepository에 접근해야하므로 서비스단에 해당 리포를 선언해주었다. 최종 수정된 서비스 save로직은 아래와 같다.

