---
layout: post
title:  "[Spring] 상품 관리 api"
date:   2023-03-18 13:48:27 +0900
categories: Study
published: False
---

# [Spring] 상품 관리 API 만들기

> Postman, IntelliJ Ultimate, Springboot 2.7.9, Java 11, Gradle, Windows(경로 설정 시 슬래시 방향 유의)

이번 스터디 과제는 요구사항을 만족하는 API를 만드는 거였다.
요구사항은 다음과 같다.

1. 상품 등록 API 만들기 
2. 상품 조회 API 만들기
3. 응답 인터페이스 구현하기

<br />
<br />

## 프로젝트 구조에 대한 고민 

'상품'이라는 구체적인 도메인에 대한 api를 만들다 보니 패키지 구성을 어떻게 해야할까에 대한 고민이 가장 먼저 들었다. 
우선 크게 레이어 계층형, 도메인형 두가지로 나뉜다고 한다.

계층형 구조는 각 계층(domain, controller, dto, service 등) 별로 패키지를 구성한다. 
도메인형의 경우는 각 도메인(본 포스팅의 경우 '상품') 별로 구성한다. 도메인 내에서의 응집도를 높일 수 있다고 한다.

개인적으로 도메인 내 응집도가 중요하다고 생각해 같은 도메인형을 따라 작성하였다.


```
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── example
    │   │           └── springclassdemo
    │   │               ├── SpringclassDemoApplication.java
    │   │               └── product
    │   │                   ├── controller
    │   │                   ├── domain
    │   │                   ├── base
    │   │                   ├── repository
    |   |                   ├── dto
    │   │                   └── service
    │   └── resources
    │       └── application.properties

```

<br />
<br />

## 상품 등록 API 만들기

Http의 POST method를 사용하여 json으로 상품명(name), 가격(price)를 전송하여 상품을 등록하는 api를 만들어보자.
DB는 아직 미정이라 `ProductRepository`라는 interface를 통해 임시로 `MemoryProductRepository`에 접근하여 저장하도록 구현하였다.

전체적인 POST 흐름은 다음과 같다.

`ProductController`의 `registerProduct` 메서드를 통해 RequestBody에 상품명, 가격을 담아 등록 요청을 보내면, `ProductService`측에서 동일한 이름의 상품이 없는지 검증 후, 등록 순서대로 id를 부여하여 product 객체로 `ProductRepository` 인터페이스를 통해 `MemeoryProductRepository`에 `save()` 메서드로 저장한다.

여기서 상품명은 고유하다!

여기서 해당 상품명 가진 상품이 이미 존재하는 경우, `ifSameProductNameExsistsException`으로 예외처리한다. 

여기서 또 고민을 한 것이 dto가 어느 계층까지 사용될 것인지였다. 
dto는 정의 그대로 data transfer object, 계층 간 데이터 전송을 위해 도메인 모델(entity) 대신 사용하는 객체인데, 목적은 캡슐화이다. 

dto-entity간 변환은 어느 곳에서 해야하는지가 가장 큰 의문이었다. 
찾아본 의견들의 결론은 결합도와 유지 보수 등 여러가지를 고려해서 결정해야 하므로 case-by-case였고, 결론적으로 이 api에서는 `repository`에서는 entity만을 처리하도록 했다. 

특히 `service`, `controller`중 변환을 어느곳에서 관장해야 할 지 잘 모르겠다...

[참고] [https://tecoble.techcourse.co.kr/post/2021-04-25-dto-layer-scope/](https://tecoble.techcourse.co.kr/post/2021-04-25-dto-layer-scope/)

<br />
<br />

응답 인터페이스 구현 전 기본적인 POST 기능만 수행하는 코드는 다음과 같다.

```
 @PostMapping("")
    public ProductDto registerProduct(@RequestBody ProductDto productDto) {
        productService.save(productDto);
        return productDto;
    }
```

postman 요청시 등록된 상품의 정보만 json으로 뜨고, 예외시 동일한 이름의 상품이 이미 존재한다는 문구가 ide 콘솔에만 나타난다.

<br />
<br />

## 상품 조회 API 만들기

GET method들은 다음과 같다.

`ProductController`의 `findAll()` 메서드를 통해 `/all`로 요청하면 전체 상품을 json 리스트로 반환한다.

`ProductController`의 `getProductById()` 메서드를 통해 `pathvariable`로 id값을 넣어 요청하면 `ProductService`에서 해당 id값을 가진 상품이 존재하는지 검증 후, `ProductRepository`를 통해 `MemoryProductRepository`의 `findById()` 메서드로 해당 상품명, 가격을 반환한다. 
여기서 해당 id값을 가진 상품이 없는 경우, `ifProductNoneExsistsException`으로 예외처리한다. 

`ProductController`의 `getProductByName()` 메서드를 통해 `requestParam`에 상품명(name)을 넣어 요청하면 `ProductService`에서 해당 이름을 가진 상품이 존재하는지 검증 후, `ProductRepository`를 통해 `MemoryProductRepository`의 `findByName()` 메서드로 해당 상품명, 가격을 반환한다. 상품명은 유니크하다는 설정이다.
여기서 해당 이름을 가진 상품이 없는 경우, `ifProductNoneExsistsException`으로 예외처리한다. 

응답 인터페이스 구현 전 기본적인 GET 기능만 수행하는 코드는 다음과 같다.

```
    // 상품 조회 api - all
    @GetMapping("/all")
    public List<Product> findAll() {
        return productService.findAll();
    }

    // 상품 조회 api - id
    @GetMapping("/{productId}")
    public ProductDto getProductById(@PathVariable int productId) {
        Product product = productService.findById(productId);
        ProductDto productInfo = new ProductDto(product.getId(), product.getName(), product.getPrice());
        return productInfo;
    }

    // 상품 조회 api - name
    @GetMapping("/name")
    public ProductDto getProductByName(@RequestParam(required = false, defaultValue = "") String name) {
        Product product = productService.findByName(name);
        ProductDto productInfo = new ProductDto(product.getId(), product.getName(), product.getPrice());
        return productInfo;
    }
```

postman 요청시 등록된 상품의 정보만 json으로 뜨고, 예외시 해당 상품이 존재하지 않는다는 문구가 ide 콘솔에만 나타난다.

<br />
<br />

## 응답 인터페이스 구현

응답을 `ResponseEntity`를 사용해서 구현했다. 
`BaseResponse` 클래스에서 기본적인 응답 구조를 `<T>`로 만들었다. 이 응답은 상태코드, 응답 메시지, 데이터 객체를 담는다. 

성공, 실패 두 경우 모두 `BaseResponse` 형식에 맞추어 응답한다.

최종적인 postman 응답은 다음과 같다.

- 상품 등록 성공

<img src='/assets/img/docs/상품조회api_1.png' />

- '노트북'이 이미 등록되어 있을 때, 동일 상품 등록으로 인한 상품 등록 실패

<img src='/assets/img/docs/상품조회api_2.png' />

- '노트북'이 이미 등록되어 있을 때, 새 상품 등록 성공

<img src='/assets/img/docs/상품조회api_3.png' />

<img src='/assets/img/docs/상품조회api_4.png' />

- 전체 상품 조회 성공

<img src='/assets/img/docs/상품조회api_5.png' />

- 등록되지 않은 상품 조회 실패 (id로 조회시)

<img src='/assets/img/docs/상품조회api_6.png' />

- 상품 조회 성공 (id로 조회)

<img src='/assets/img/docs/상품조회api_7.png' />

- 상품 조회 성공 (상품명으로 조회)

<img src='/assets/img/docs/상품조회api_8.png' />

<br />

전체 코드는 [여기](https://github.com/JSCODE-EDU/spring-class-Sua)서 확인할 수 있다.