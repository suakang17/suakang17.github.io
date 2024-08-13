---
layout: post
title:  "[Java/SpringBoot] 🛠️ LocalDate, LocalDateTime 커스텀 직렬화 및 역직렬화: 사용자 Locale 기반 포맷팅 유틸 만들기"
date:   2024-07-01 00:00:00 +0900
categories: Devlife
---

# 🛠️ LocalDate/LocalDateTime 커스텀 직렬화 및 역직렬화: 로케일 기반 포맷 조정

## 개요
진행 중인 프로젝트에서 LocalDate, LocalDateTime 객체를 JSON으로 직렬화하고 역직렬화할 때, 로케일에 맞춘 포맷을 적용해야 할 필요가 생겼다. Jackson 라이브러리와 Map을 활용하여 LocalDate, LocalDateTime의 포맷을 로케일에 따라 조정하는 커스텀 직렬화기와 역직렬화기를 구현해보았다. 

### Jackson 라이브러리 개요
Jackson은 Java 객체를 JSON으로 직렬화하거나 JSON을 Java 객체로 역직렬화하는 데 사용되는 라이브러리로, 주요 클래스 중 하나인 ObjectMapper는 JSON 데이터를 처리하는 핵심 클래스이다. ObjectMapper는 직렬화 및 역직렬화에 대한 설정을 조정할 수 있는 다양한 기능을 제공하며, 커스텀 직렬화기와 역직렬화기를 쉽게 등록하고 관리할 수 있다.

### 커스텀 직렬화기 및 역직렬화기 구현

#### DateTimeFormatterUtils 클래스

`DateTimeFormatterUtils` 클래스는 다양한 로케일에 맞춘 `DateTimeFormatter`를 관리한다. 이 클래스를 통해 애플리케이션의 국제화 요구 사항에 맞게 날짜와 시간 포맷을 동적으로 조정하고자 했다.

```java
public class DateTimeFormatterUtils {
    private DateTimeFormatterUtils() { throw new IllegalStateException("Utility class");}

    public static final Map<Locale, DateTimeFormatter> dateTimeFormatterMap =
            Map.ofEntries(
                Map.entry(Locale.KOREA, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")),
                Map.entry(Locale.US, DateTimeFormatter.ofPattern("MM/dd/yyyy HH:mm:ss")),
                Map.entry(Locale.UK, DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss")),
                Map.entry(Locale.FRANCE, DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss")),
                Map.entry(Locale.GERMANY, DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm:ss")),
                Map.entry(Locale.JAPAN, DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss")),
                Map.entry(Locale.CHINA, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))
            );

    public static DateTimeFormatter getDateTimeFormatter(Locale locale) {
        return dateTimeFormatterMap.getOrDefault(locale, dateTimeFormatterMap.get(Locale.KOREA));
    }
}
```

#### LocalDateTimeSerializer 클래스
`LocalDateTimeSerializer` 클래스는 `LocalDateTime` 객체를 JSON 문자열로 변환할 때, 로케일에 맞춘 포맷을 적용한다. 이를 통해 사용자별로 적절한 포맷을 제공할 수 있다. 포맷 설정에 실패할 경우, 각자 적절한 예외를 처리한다.

```java
public class LocalDateTimeSerializer extends StdSerializer<LocalDateTime> {

    public LocalDateTimeSerializer() {
        super(LocalDateTime.class);
    }

    @Override
    public void serialize(LocalDateTime value, JsonGenerator gen, SerializerProvider provider)
            throws IOException {
        Locale locale = LocaleContextHolder.getLocale();
        DateTimeFormatter formatter = DateTimeFormatterUtils.getDateTimeFormatter(locale);

        try {
            gen.writeString(value.format(formatter));
        } catch (Exception e) {
            throw new IOException("The format of the response for the datetime field is invalid.", e);
        }
    }
}
```

#### LocalDateTimeDeserializer 클래스
`LocalDateTimeDeserializer` 클래스는 JSON 문자열을 `LocalDateTime` 객체로 변환할 때, 설정된 포맷을 적용하여 변환한다. 이를 통해 서버에서 클라이언트로 전송된 데이터가 유효한 형식인지 확인할 수 있다.

```java
public class LocalDateTimeDeserializer extends StdDeserializer<LocalDateTime> {

    public LocalDateTimeDeserializer() {
        super(LocalDateTime.class);
    }

    @Override
    public LocalDateTime deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
        String dateTime = p.getText();
        Locale locale = LocaleContextHolder.getLocale();
        DateTimeFormatter formatter = DateTimeFormatterUtils.getDateTimeFormatter(locale);

        try {
            return LocalDateTime.parse(dateTime, formatter);
        } catch (DateTimeParseException e) {
            throw new InvalidRequestException("msg.valid.LocalDateTimeRange.field", "");
        }
    }
}
```

#### JacksonConfig 클래스
`JacksonConfig` 클래스는 `ObjectMapper`를 설정하여 커스텀 직렬화기와 역직렬화기를 등록한다. 이를 통해 모든 JSON 변환 작업에서 일관된 날짜와 시간 포맷을 적용할 수 있다.

```java
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper(Jackson2ObjectMapperBuilder builder) {
        ObjectMapper objectMapper = builder.createXmlMapper(false).build();

        SimpleModule module = new SimpleModule();
        module.addSerializer(LocalDate.class, new LocalDateSerializer());
        module.addDeserializer(LocalDate.class, new LocalDateDeserializer());

        module.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer());
        module.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer());

        objectMapper.registerModule(new JavaTimeModule());
        objectMapper.registerModule(module);
        objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);

        return objectMapper;
    }
}
```

#### 적용 결과
`LocalDateTime`, `LocalDate` 객체를 로케일에 맞춘 포맷으로 처리할 수 있게 되었다.

1. **포맷 일관성 유지**: 다양한 로케일에 맞는 포맷을 자동으로 적용함으로써 데이터의 일관성을 유지할 수 있다. 지금 프로젝트처럼 다양한 Locale 사용자를 가진 애플리케이션에서 유용할 것 같다.

2. **프론트엔드의 데이터 처리 간소화**: JSON 응답에서 날짜와 시간 정보를 일관되게 표시함으로써, 프론트엔드 개발자들이 데이터를 파싱할 필요가 없어져 좋아했다. ㅎㅎ

3. **유지보수 용이성**: 포맷 변경이 필요할 때, 이 파일 하낙로 변경을 관리하면 된다.

LocalDate, Time Formatter도 동일한 로직으로 작성했다!
유틸성 클래스를 직접 작성하고 반영한 건 또 처음인 것 같은데 나중에 프로젝트에 더 많은 로케일을 추가하거나 포맷을 변경해야 할 경우에도 유연하게 대처할 수 있지 않을까 싶다.
