---
layout: post
title:  "[Java/SpringBoot] 🛠️ LocalDate, LocalDateTime 커스텀 직렬화 및 역직렬화: 사용자 Locale 기반 포맷팅
"
date:   2024-08-01 00:00:00 +0900
categories: Devlife
published: false
---
# 🛠️ LocalDateTime 커스텀 직렬화 및 역직렬화: 로케일 기반 포맷 조정
## 개요
애플리케이션에서 LocalDateTime 객체를 JSON으로 직렬화하고 역직렬화할 때, 로케일에 맞춘 포맷을 적용해야 할 필요가 있었습니다. Jackson 라이브러리를 활용하여 LocalDateTime의 포맷을 로케일에 따라 조정하는 커스텀 직렬화기와 역직렬화기를 구현했습니다.

### Jackson 라이브러리 개요
Jackson은 Java 객체를 JSON으로 직렬화하거나 JSON을 Java 객체로 역직렬화하는 데 사용되는 라이브러리입니다. 주요 클래스 중 하나인 ObjectMapper는 JSON 데이터를 처리하는 핵심 클래스입니다. ObjectMapper는 직렬화 및 역직렬화에 대한 설정을 조정할 수 있는 다양한 기능을 제공합니다.

### 커스텀 직렬화기 및 역직렬화기 구현
#### DateTimeFormatterUtils 클래스

DateTimeFormatterUtils 클래스를 사용하여 로케일에 따라 DateTimeFormatter를 관리합니다. 이를 통해 다양한 로케일에 적합한 날짜와 시간 포맷을 설정할 수 있습니다.

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
LocalDateTime 객체를 JSON 문자열로 변환할 때 로케일에 맞춘 포맷을 적용합니다.
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

JSON 문자열을 LocalDateTime 객체로 변환할 때, 설정된 포맷을 적용하여 변환합니다.

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

ObjectMapper를 설정하여 커스텀 직렬화기와 역직렬화기를 등록합니다.

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
적용 결과
이러한 커스텀 직렬화기와 역직렬화기를 적용하여 LocalDateTime 객체를 로케일에 맞춘 포맷으로 처리할 수 있게 되었습니다. 이를 통해 JSON 응답에서 날짜와 시간 정보를 보다 일관되게 표시할 수 있으며, 다양한 로케일에 대응할 수 있는 유연한 시스템이 구축되었습니다. 데이터 포맷의 일관성을 유지하면서 사용자 경험을 크게 향상시킬 수 있었습니다.