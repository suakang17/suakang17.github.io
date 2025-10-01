---
layout: post
title:  "[Spring] 객체 매핑 라이브러리 비교하기"
date:   2025-06-13 00:00:00 +0900
categories: Devlife
---

# [Spring] 객체 매핑 라이브러리 비교하기

## TL;DR

- **MapStruct**: 컴파일 타임 코드 생성, 성능 최강 (대규모 서비스)
- **BeanUtils**: 간단하지만 느림 (소규모 어드민 등)
- **ModelMapper**: 중간 성능, 설정 복잡
- 대규모 환경에서 MapStruct가 200배 이상 빠른 이유는 리플렉션 대신 순수 getter/setter를 사용하기 때문이다

---

## 왜 객체 매핑이 필요한가

Spring 기반 백엔드 개발에서는 아래와 같은 코드를 반복적으로 작성하게 된다:

```java
// Controller
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    UserEntity entity = userService.findById(id);
    
    // 매번 이렇게 변환해야 한다
    UserResponse response = new UserResponse();
    response.setId(entity.getId());
    response.setName(entity.getUserName());
    response.setEmail(entity.getEmail());
    response.setCreatedAt(entity.getCreatedDate());
    // ... 50줄 반복
    
    return response;
}
```

Entity를 그대로 노출하면 보안 이슈가 발생하고, 필요한 데이터만 가공해서 내려줘야 하므로 DTO 변환은 필수다. 문제는 이를 어떻게 **효율적으로** 처리하느냐다.

---

## 주로 사용하는 매핑 4가지 비교

### 1. 수동 매핑 (직접 구현)

```java
public class UserMapper {
    public static UserDto toDto(UserEntity entity) {
        UserDto dto = new UserDto();
        dto.setId(entity.getId());
        dto.setName(entity.getUserName());
        dto.setEmail(entity.getEmail());
        return dto;
    }
}
```

**장점:**
- 가장 빠르다 (순수 Java 메서드 호출)
- 명확한 로직
- 디버깅이 쉽다

**단점:**
- 보일러플레이트 코드가 많다
- 필드 추가 시마다 수정이 필요하다
- 실수로 필드를 누락할 수 있다

**평가:** 필드가 서너개라면 나쁘지 않을 수도 있지만, 그 이상은 유지보수가 어렵다.

---

### 2. Apache Commons BeanUtils / Spring BeanUtils

```java
import org.springframework.beans.BeanUtils;

UserDto dto = new UserDto();
BeanUtils.copyProperties(entity, dto);
```

**장점:**
- 한 줄로 처리 가능

**단점:**
- **리플렉션 사용으로 매우 느리다**
- 타입 불일치 시 런타임 에러가 발생한다
- 필드명이 다르면 매핑할 수 없다

**성능 테스트 결과:**
```
10,000번 매핑 기준
BeanUtils: 1,200ms
수동 매핑: 6ms
→ 200배 차이
```

**평가:** 사내 어드민처럼 TPS가 낮은 곳에서만 사용하고, 고객용 API에서는 사용하지 않는 것이 좋을 것 같다.

---

### 3. ModelMapper

```java
ModelMapper modelMapper = new ModelMapper();
UserDto dto = modelMapper.map(entity, UserDto.class);

// 복잡한 매핑도 가능
modelMapper.typeMap(UserEntity.class, UserDto.class)
    .addMapping(UserEntity::getUserName, UserDto::setName);
```

**장점:**
- 중첩 객체도 자동 매핑한다
- BeanUtils보다 다양한 기능을 제공한다
- 유연한 설정이 가능하다

**단점:**
- 여전히 리플렉션 기반이라 느리다
- 설정이 복잡해지면 디버깅이 어렵다
- 예상치 못한 매핑이 발생할 수 있다

**성능:**
```
ModelMapper: 800ms (캐싱 적용 시)
```

**평가:** BeanUtils보다는 낫지만, 대규모 환경에는 부적합하다.

---

### 4. MapStruct

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDto toDto(UserEntity entity);
    
    @Mapping(source = "userName", target = "name")
    @Mapping(target = "password", ignore = true)
    UserDto toDtoCustom(UserEntity entity);
}
```

**장점:**
- **컴파일 타임에 실제 Java 코드를 생성한다**
- 수동 매핑 수준의 성능을 보인다
- 타입 안전성을 보장한다
- 생성된 코드를 확인할 수 있다

**단점:**
- 초기 설정이 필요하다
- 어노테이션 프로세서에 대한 이해가 필요하다

**생성되는 코드 확인:**
```java
// build/generated/sources/annotationProcessor/java/main 에 생성됨

@Component
public class UserMapperImpl implements UserMapper {
    @Override
    public UserDto toDto(UserEntity entity) {
        if (entity == null) {
            return null;
        }
        
        UserDto dto = new UserDto();
        dto.setId(entity.getId());
        dto.setName(entity.getUserName());  // 자동 매핑
        dto.setEmail(entity.getEmail());
        
        return dto;
    }
}
```

생성된 코드를 보면 수동으로 작성하는 코드와 동일하다는 것을 알 수 있다. 차이는 **컴파일러가 대신 작성**해준다는 점이다.

**성능:**
```
MapStruct: 5ms
→ BeanUtils 대비 240배 빠름
→ 수동 매핑과 거의 동일
```

---

## MapStruct가 성능이 좋은 이유

### 리플렉션의 숨겨진 비용

BeanUtils가 내부적으로 수행하는 작업:

```java
// 1. 클래스 메타정보 조회
Field[] fields = sourceClass.getDeclaredFields();

// 2. 각 필드마다 반복
for (Field field : fields) {
    field.setAccessible(true);  // private 접근 허용
    
    // 3. 메서드 이름으로 getter/setter 찾기 (문자열 검색)
    Method getter = sourceClass.getMethod("get" + capitalize(field.getName()));
    Method setter = targetClass.getMethod("set" + capitalize(field.getName()));
    
    // 4. 리플렉션으로 값 가져와서 설정
    Object value = getter.invoke(source);
    setter.invoke(target, value);
}
```

**문제점:**
- 메서드 룩업 비용이 발생한다 (HashMap 검색)
- invoke() 호출 오버헤드가 존재한다
- 타입 체킹을 런타임에 수행한다
- JIT 컴파일러 최적화가 불가능하다

### MapStruct의 처리 방식

```java
// 생성된 코드 (순수 Java)
dto.setName(entity.getUserName());
```

단순히 메서드를 호출할 뿐이다. JIT 컴파일러가 인라이닝까지 수행한다.

### 성능 비교 시나리오

```
초당 10,000 요청 처리 시나리오

BeanUtils:
- 요청당 0.12ms 소요
- 10,000 * 0.12ms = 1,200ms
- 단일 스레드로 처리 불가 → 병목 발생

MapStruct:
- 요청당 0.0005ms 소요  
- 10,000 * 0.0005ms = 5ms
- 여유롭게 처리 가능
```

톰캣 기본 스레드 200개 기준으로, BeanUtils를 사용하면 매핑 작업만으로 스레드 대부분을 소진하게 된다.

---

## MapStruct 설정 방법

### Gradle 설정

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
}

dependencies {
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
    
    // Lombok 같이 사용할 때
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
}

tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
    options.compilerArgs.addAll([
        "-Amapstruct.defaultComponentModel=spring",
        "-Amapstruct.unmappedTargetPolicy=WARN"
    ])
}
```

### 컴파일러 옵션 해부

#### 1. `-Amapstruct.defaultComponentModel=spring`

```java
// 이 옵션이 없으면
@Mapper(componentModel = "spring")  // 매번 명시해야 한다
public interface UserMapper { }

// 이 옵션이 있으면
@Mapper  // 이것만 작성해도 Spring Bean으로 등록된다
public interface UserMapper { }
```

생성되는 코드:
```java
@Component  // 자동으로 추가된다
public class UserMapperImpl implements UserMapper {
    // Spring이 자동으로 빈을 관리한다
}
```

**활용 방식:**
```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserMapper mapper;  // 자동 주입된다
    
    public UserDto getUser(Long id) {
        UserEntity entity = repository.findById(id);
        return mapper.toDto(entity);
    }
}
```

#### 2. `-Amapstruct.unmappedTargetPolicy=WARN`

```java
public class UserEntity {
    private Long id;
    private String userName;
    private String password;
    private String address; 
}

public class UserDto {
    private Long id;
    private String name;
    private String address; 
}

@Mapper
public interface UserMapper {
    UserDto toDto(UserEntity entity);
}
```

**WARN (권장):**
```
컴파일 시:
Warning: Unmapped target property: "address"
→ 누락된 필드를 발견할 수 있다
```

**IGNORE:**
```
아무 경고 없이 넘어간다
→ 실수를 놓칠 수 있다
```

**ERROR:**
```
컴파일 에러가 발생한다
→ 모든 필드를 명시적으로 처리하도록 강제한다 (가장 안전)
```

**실무 추천 설정:**
```groovy
// 신규 프로젝트
"-Amapstruct.unmappedTargetPolicy=ERROR"

// 레거시 마이그레이션
"-Amapstruct.unmappedTargetPolicy=WARN"
```

---

## MapStruct vs MyBatis @Mapper 구분

두 라이브러리 모두 `@Mapper`를 사용하지만 완전히 다른 목적을 가진다:

### MapStruct @Mapper

```java
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring")
public interface UserDtoMapper {
    // Java 객체 → Java 객체
    UserDto toDto(UserEntity entity);
}
```

- **역할:** 객체 간 변환
- **처리 시점:** 컴파일 타임
- **생성물:** 실제 Java 클래스

### MyBatis @Mapper

```java
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface UserRepository {
    // SQL → Java 객체
    @Select("SELECT * FROM users WHERE id = #{id}")
    UserEntity findById(Long id);
}
```

- **역할:** DB 접근 (ORM)
- **처리 시점:** 런타임 (동적 프록시)
- **생성물:** 없음 (런타임 프록시)

### 실무 네이밍 컨벤션

```java
// 명확한 구분
@org.mapstruct.Mapper(componentModel = "spring")
public interface UserDtoMapper {  // 접미사로 구분
    UserDto toDto(UserEntity entity);
}

@org.apache.ibatis.annotations.Mapper
public interface UserRepository {  // 역할로 구분
    UserEntity findById(Long id);
}

// 사용 예시
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;    // DB 접근
    private final UserDtoMapper userDtoMapper;      // 객체 변환
    
    public UserDto getUser(Long id) {
        UserEntity entity = userRepository.findById(id);  // SQL 실행
        return userDtoMapper.toDto(entity);                // 객체 변환
    }
}
```

---

## MapStruct 실전 활용 패턴

### 1. 기본 매핑

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDto toDto(UserEntity entity);
    List<UserDto> toDtoList(List<UserEntity> entities);  // 리스트도 자동 처리
}
```

### 2. 필드명이 다를 때

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    @Mapping(source = "userName", target = "name")
    @Mapping(source = "userEmail", target = "email")
    UserDto toDto(UserEntity entity);
}
```

### 3. 특정 필드 제외

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    @Mapping(target = "password", ignore = true)
    @Mapping(target = "socialSecurityNumber", ignore = true)
    UserDto toDto(UserEntity entity);
}
```

### 4. 커스텀 로직 적용

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    @Mapping(target = "fullName", expression = "java(entity.getFirstName() + \" \" + entity.getLastName())")
    @Mapping(target = "age", source = "birthDate", qualifiedByName = "calculateAge")
    UserDto toDto(UserEntity entity);
    
    @Named("calculateAge")
    default int calculateAge(LocalDate birthDate) {
        return Period.between(birthDate, LocalDate.now()).getYears();
    }
}
```

### 5. 중첩 객체 처리

```java
public class OrderEntity {
    private Long id;
    private UserEntity user;  // 중첩 객체
    private List<OrderItemEntity> items;
}

@Mapper(componentModel = "spring", uses = {UserMapper.class, OrderItemMapper.class})
public interface OrderMapper {
    OrderDto toDto(OrderEntity entity);
    // user, items도 자동으로 각각의 Mapper를 사용해서 변환된다
}
```

### 6. 양방향 매핑 및 업데이트

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDto toDto(UserEntity entity);
    UserEntity toEntity(UserDto dto);
    
    // 업데이트용
    @MappingTarget
    void updateEntity(UserDto dto, @MappingTarget UserEntity entity);
}

// 사용 예시
UserEntity entity = repository.findById(id);
userMapper.updateEntity(updateDto, entity);  // 기존 entity를 업데이트한다
repository.save(entity);
```

---

## 성능 최적화 팁

### 1. 불필요한 매핑 제거

```java
// 비효율적인 방식
@GetMapping("/users")
public List<UserDto> getUsers() {
    List<UserEntity> entities = repository.findAll();  // 1000건
    return entities.stream()
        .map(mapper::toDto)  // 1000번 매핑
        .collect(Collectors.toList());
}

// 효율적인 방식
@GetMapping("/users")
public List<UserDto> getUsers() {
    List<UserEntity> entities = repository.findAll();
    return mapper.toDtoList(entities);  // 배치 처리 최적화
}
```

### 2. 조건부 매핑 활용

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    @Mapping(target = "sensitiveData", 
             expression = "java(includePrivate ? entity.getSensitiveData() : null)")
    UserDto toDto(UserEntity entity, boolean includePrivate);
}
```

### 3. 페이징 처리

```java
@GetMapping("/users")
public Page<UserDto> getUsers(Pageable pageable) {
    Page<UserEntity> page = repository.findAll(pageable);
    return page.map(mapper::toDto);  // Page도 매핑을 지원한다
}
```

---

### 실제 마이그레이션 사례

```
Before (BeanUtils):
- 평균 응답시간: 350ms
- 매핑 소요 시간: 120ms (34%)

After (MapStruct):
- 평균 응답시간: 235ms
- 매핑 소요 시간: 5ms (2%)

→ 응답시간 33% 개선
→ 서버 증설 없이 처리량 50% 증가
```

---

## 트러블슈팅

### 1. Mapper 빈 주입이 안 되는 경우

```
Error: Could not autowire. No beans of 'UserMapper' type found.
```

**해결 방법:**
```groovy
// build.gradle 확인
annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'

// 또는 componentModel 확인
@Mapper(componentModel = "spring")
```

### 2. Lombok과 충돌하는 경우

```groovy
dependencies {
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'  // 필수
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
}
```

### 3. 생성된 코드가 보이지 않는 경우

생성 위치:
```
build/generated/sources/annotationProcessor/java/main
```
IntelliJ에서: `Build > Rebuild Project` 실행

---

## 결론

객체 매핑은 백엔드 개발에서 피할 수 없는 작업이다. 규모가 작을 때는 BeanUtils로도 충분하지만, 서비스가 커지면 성능 병목이 명확하게 체감된다.

**MapStruct 도입 시 얻는 이점:**
- 성능 200배 향상
- 타입 안전성 보장
- 유지보수성 개선
- 리팩토링 부담 감소

초기 설정만 제대로 해두면, 이후에는 인터페이스만 정의하면 된다. 컴파일러가 알아서 최적화된 코드를 생성해준다.

---

**참고 자료:**
- MapStruct 공식 문서: https://mapstruct.org/
- 성능 벤치마크: https://www.baeldung.com/java-performance-mapping-frameworks
- GitHub: https://github.com/mapstruct/mapstruct
