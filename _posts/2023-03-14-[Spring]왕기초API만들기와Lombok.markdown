---
layout: post
title:  "[Spring]왕기초API만들기와 Lombok으로 로그찍기"
date:   2023-03-14 11:19:37 +0900
categories: Study
---

# [Spring] 아주아주 간단한 API만들기와 Lombok으로 로그찍기

## 아주아주 간단한 API 만들기
저번에 API에 관한 포스팅을 작성했었다. 이번에는 직접 아주아주 간단한 API를 만들어보자!

> IntelliJ Ultimate, Springboot 2.7.9, Java 11, Gradle, Windows(슬래시 방향 유의)

### Spring boot 프로젝트 빌드하기

프로젝트 빌드는 두가지 방법으로 가능하다.

1. jar 빌드

만들어둔 스프링 부트 프로젝트의 `gradlew` 파일을 찾아서 `gradlew build` 해준다.

<img src='/assets/img/docs/0314_1.png' />

Spring boot gradle 플러그인 2.5버전부터는 별도의 설정이 없으면 빌드시 `프로젝트\build\libs` 디렉토리 내에 `.jar` 파일이 두가지 생성된다. 하나는 그냥 .jar, 다른 하나는 plain.jar이다.

plain.jar는 jar task로 생성되는데, **dependency를 포함하지 않고** 오로지 소스코드의 클래스 파일, 리소스 파일만 포함해 실행이 제대로 이루어지지 않는다고 한다. 따라서 plain archive라고 칭한다고 한다.

우리가 실행할 jar는 bootjar task로 생성되는데, executable archive라고 하고, **모든 dependency를 포함, 함께 빌드한다.** 

(추가로 archive는 빌드 결과물을 의미한다고 한다.)

다음과 같이 plain jar를 생성하지 않는 옵션을 줄 수가 있다.
```
# /build.gradle 

plugins {
    ...
}
...
jar {
    enabled = false
}
```
[참고] [https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/htmlsingle/#packaging-executable](https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/htmlsingle/#packaging-executable)

<br />

어쨌건! 빌드로 돌아와서 두가지 중 executable jar를 `java -jar 파일명.jar` 를 통해 실행해주면 Spring이 실행되는 것을 볼 수 있다.

<img src='/assets/img/docs/0314_2.png' />

~~위의 오타난 라인은 무시하자.~~

<img src='/assets/img/docs/0314_7.png' />

만들어둔 초간단 api가 잘 동작하고 있다.

\+ 추가로 기존에 이미 빌드했던 것이 있다면 `gradlew clean`으로 삭제하고 다시 빌드 할 수 있다. 

<br />

2. IDE 사용하여 빌드하기 ([spring initializr를 사용하는 방법도 있다.](https://suakang17.github.io/study/2022-05-11-Spring-%EC%8A%A4%ED%94%84%EB%A7%81%EA%B3%B5%EB%B6%801/)) 

[공식문서](https://www.jetbrains.com/help/idea/spring-boot.html)나 검색하면 나오는 많은 블로그들이 Intellij로 Spring boot 프로젝트를 생성, 빌드하는 과정을 상세히 알려주니 참고하자.

내 `build.gradle`은 다음과 같다.  
Java와 sdk버전 맞춰주는 것을 유의하자! 
```
# /build.gradle

plugins {
    id 'java'
    id 'org.springframework.boot' version '2.7.9'
    id 'io.spring.dependency-management' version '1.1.0'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    implementation 'org.projectlombok:lombok'
}

tasks.named('test') {
    useJUnitPlatform()
}

```
<img src='/assets/img/docs/0314_3.png' />

빌드 후 생성된 main함수를 찾아서 실행하면 콘솔에 뜨는 해당하는 포트로 접속해보자.  
내 경우는 `localhost:8080`이었다.

### 초간단 API

REST API를 만들어줄 위치로 `controller`라는 패키지를 생성해주었다.  
MVC 패턴의 Controller이다.


<img src='/assets/img/docs/0314_4.png' />

문자열 `test`를 반환하는 문자열 메소드이다. 
`@GetMapping`으로 지정한 URL로 접속시 정해진 리턴값을 반환한다. 

`@Controller`를 사용하면 데이터를 반환해야하는 경우마다 `@ResponseBody`를 붙여줘야해서 메서드의 수가 많아질수록 번거로워진다.  

그래서 이 둘을 결합한 `@RestController`를 붙여주면 그런 번거로움이 사라진다고 한다.  


그럼 query parameter로 입력된 값을 리턴하는 API도 만들어보자.  

<img src='/assets/img/docs/0314_5.png' />


컨트롤러에서 `@RequestParam`을 통해 쿼리 파라미터를 받아올 수 있다.  

형식은 다음과 같다.  
```
@RequestParam(value = "가져올 데이터의 이름") 데이터타입 가져온데이터를 담을 변수
```

위의 예시에서는 `/localhost:8080/name?name=sua`로 접속해서 파라미터의 값, 이름을 함께 전달했다. 보통 검색엔진에서 검색할 때 이런 형식을 볼 수 있다. 

나는 전달한 파라미터를 name이라는 변수에 넣고 그 값을 return했다. 따라서 view에 저장된 sua라는 문자열이 뜬다. 

\+ 파라미터 이름, 변수명이 같으면 다음과 같이 파라미터 이름을 생략할 수 있다.  
`@RequestParam String name`

이 때, 데이터값을 함께 넣지 않는 경우 404 에러가 발생하므로, 방지를 위해 `required = false, defaultValue = ""` 를 통해 default로 들어갈 값을 설정해주자. 

## lombok으로 로그 찍기

### 필요성

그동안 따로 로깅을 해본 경험은 없고, 정 필요한 경우에 `System.out.println()`으로 값을 확인해보는정도만 했었다. 

로깅에 System.out.println()을 사용하면 안되는 이유는 다음과 같다고 한다.
1. 저장되지 않는다.
2. 단순히 전달한 인자만 출력되고, 디버깅시 필요한 시각, 위치 등의 정보는 기록되지 않는다.
3. 성능 저하
4. 로그 출력 레벨 지정 불가

자세한 내용을 [여기서](https://hudi.blog/do-not-use-system-out-println-for-logging/) 너무 잘 정리해주셔서 도움이 되었다.

### Spring boot에서 로깅하기

Spring boot에서는 Slf4j 인터페이스를 통해 로깅한다고 한다. 인터페이스이기에 추후 로깅 라이브러리를 변경해도 소스코드를 수정할 필요는 없다.

위의 `build.gradle`의 dependency 부분에 보면 `implementation 'org.projectlombok:lombok'`이 있다.  
추가해주고 이제 lombok을 사용해보자!


보통은 다음과 같이 Logger를 클래스마다 생성했었다고 한다.
```
// Logger를 직접 클래스 변수로 선언하여 쓰는 방식

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class logSample {
    private static final Logger log = LoggerFactory.getLogger(logSample.class);

    public static void main(String[] args) {
        log.info("------------Log test------------");
    }
}
```
요즘엔 `@Slf4j` 어노테이션으로 직접 Logger를 생성하지 않아도 Logger 객체를 생성할 수 있다고 한다.  
다만 변수명이 log로 고정된다.
```
// @Slf4j 사용

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class Slf4jSample {

    public static void main(String[] args) {
        log.info("---------- Log 테스트 ---------");
    }

}
```

결과는 동일하다.
```
13:16:54.379 [main] INFO com.example.springclassdemo.logSample.logSample - ------------Log test------------
```

그러면 위에서 만든 이름을 쿼리 파라미터로 받는 API에 로그를 찍어보자.

별도로 logSample 패키지를 만들어주고, name 변수에 담긴 쿼리 파라미터 값이 뭔지 보여주는 `namelogger`라는 메서드를 만들어주었다.

```
# /logSample/Slf4jSample

@Slf4j
public class Slf4jSample {

    public static void namelogger(String s) {
        log.info("---------- Query Parameter Log ---------");
        log.info("name: {}", s);
        log.info("---------- Log END ---------");
    }
}
```
그리고 위의 이름 반환 API를 다음과 같이 변경해주었다.
```
# /controller/stringController

import com.example.springclassdemo.logSample.Slf4jSample;


@GetMapping("/name")
    public String viewName(@RequestParam(value = "name", required = false, defaultValue = "") String name) {
        Slf4jSample logger = new Slf4jSample();
        logger.namelogger(name);

        return name;
    }
```

후에 localhost에 이름으로 쿼리 파라미터를 넣어주면 콘솔에 이쁘게 찍힌다! 
```
2023-03-14 17:43:34.006  INFO 15288 --- [nio-8080-exec-1] c.e.s.logSample.Slf4jSample              : ---------- Query Parameter Log ---------
2023-03-14 17:43:34.006  INFO 15288 --- [nio-8080-exec-1] c.e.s.logSample.Slf4jSample              : name: sua
2023-03-14 17:43:34.013  INFO 15288 --- [nio-8080-exec-1] c.e.s.logSample.Slf4jSample              : ---------- Log END ---------
```

### error: cannot find symbol 이슈  

사실 중간에 이슈가 하나 있었다.  
빌드가 안되고 `error: cannot find symbol`이 발생해서 구글링한 결과 `annotationProcessor` 설정을 해주어야 한다고 해서 우선 Intellij 상에서 annotationProcessor 설정을 변경하였으나 안됐다.  
그래서 다음으로 build.gradle에 의존성을 추가해주었다. 여기서 해결이 됐다.

```
# /build.gradle - dependency

compileOnly 'org.projectlombok:lombok:1.18.8'
annotationProcessor 'org.projectlombok:lombok:1.18.8'
```

[해결 방안 출처] [https://stackoverflow.com/questions/35236104/gradle-build-fails-on-lombok-annotated-classes](https://stackoverflow.com/questions/35236104/gradle-build-fails-on-lombok-annotated-classes)
