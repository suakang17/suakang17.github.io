---
layout: post  
title:  "[LALALA] 플레이리스트 서버 리팩터링하기"  
date:   2024-04-01 00:00:00 +0900  
categories: Devlife  
published: true  
---

# LALALA 플레이리스트 서버 핵심 비즈니스 로직 리팩터링하기

> Java / SpringBoot / MongoDB / Kafka

## 0. 개요

이 포스트는 음원 사이트의 플레이리스트 서버 리팩터링 과정에 대한 기록이다. 주요 기술 스택으로는 Java, SpringBoot, MongoDB, Kafka를 사용하였고, 리팩터링을 통해 서버의 유지보수성, 성능, 확장성을 개선하는 것이 목표였다. 리팩터링 과정에서의 주요 고민과 적용한 기술적 접근 방식을 공유하고자 한다.

## 1. PlaylistService 리팩터링

### 리팩터링 전

기존 `PlaylistService`는 플레이리스트를 생성하고 캐싱하는 과정에서 여러 가지 책임을 지니고 있었다. 특히, 캐싱과 데이터베이스 작업을 직접 처리하는 코드가 혼재되어 있었고, 외부 서비스 호출을 `FeignClient`를 통해 직접 처리하고 있었다. 이런 방식은 다음과 같은 문제를 야기했다:

- **복잡성**: 서비스 클래스가 다양한 책임을 지게 되면서 코드가 복잡해졌다.
- **유지보수 어려움**: 캐싱 로직과 데이터베이스 로직이 혼합되어 있어 유지보수가 어려워졌다.

### 리팩터링 후

리팩터링 후 `PlaylistService`는 다음과 같은 구조로 개선되었다:

```java
@Service
@RequiredArgsConstructor
public class PlaylistService {
    private final PlaylistRepository playlistRepository;
    private final CacheService cacheService;
    private final PlaylistToDtoConverter converter;
    private final MusicService musicService;
    private final AsyncDatabaseService asyncDatabaseService;

    public PlaylistResponseDto createPlaylist(PlaylistRequestDto requestDto) {
        List<Music> musics = musicService.getMusicsFromIds(requestDto.getMusics());
        Playlist playlist = Playlist.createFrom(requestDto, musics);
        Playlist saved = playlistRepository.save(playlist);
        cacheService.cachePlaylist(saved);
        asyncDatabaseService.savePlaylistAsync(saved);
        return converter.convert(saved);
    }
}
```

이 리팩터링에서 중점을 둔 기술적 고려 사항은 다음과 같다:

- **책임 분리**: `CacheService`, `MusicService`, `AsyncDatabaseService`를 도입하여 각 클래스의 책임을 명확히 했다. 이를 통해 `PlaylistService`는 플레이리스트 생성과 관련된 로직만 담당하게 되었고, 캐싱, 비동기 처리, 외부 서비스 호출은 별도의 클래스로 분리되었다.
- **비동기 처리 도입**: `AsyncDatabaseService`를 통해 데이터베이스 작업을 비동기적으로 처리함으로써, 사용자 요청에 대한 응답 속도를 개선하고, 시스템의 전반적인 처리량을 향상시켰다.
- **도메인 로직 강화**: `Playlist.createFrom()` 정적 팩토리 메서드를 도입하여 객체 생성 로직을 도메인 객체 내부로 이동시켰다. 이로 인해 생성 로직의 캡슐화와 재사용성이 증가하였다.

## 2. MusicService 도입

### 리팩터링 전

기존에는 `PlaylistService` 내에서 외부 음악 서비스의 API를 호출하여 음악 데이터를 가져오고 있었다. 이는 다음과 같은 문제를 초래했다:

- **서비스 복잡성**: 음악 데이터 처리와 관련된 로직이 `PlaylistService`에 포함되어 복잡성이 증가했다.
- **변환 로직의 중복**: DTO에서 도메인 객체로의 변환 로직이 분산되어 있어 변경 시 여러 군데에서 수정을 해야 했다.

### 리팩터링 후

`MusicService`를 도입하여 음악 데이터와 관련된 책임을 분리하였다:

```java
@Service
@RequiredArgsConstructor
public class MusicService {
    private final FeignMusicClient musicClient;

    public List<Music> getMusicsFromIds(List<Long> musicIds) {
        BaseResponse<List<MusicDTO>> response = musicClient.getMusicFromIds(new MusicRetrieveRequestDTO(musicIds));
        return response.getData().stream()
                .map(this::convertToMusic)
                .toList();
    }

    private Music convertToMusic(MusicDTO dto) {
        return Music.builder()
                .id(dto.getId())
                .title(dto.getTitle())
                .artistId(dto.getArtist().getId())
                .artist(dto.getArtist().getName())
                .albumId(dto.getAlbum().getId())
                .album(dto.getAlbum().getTitle())
                .thumbnail(dto.getAlbum().getCoverUrl())
                .playtime(formatPlayTime(dto.getPlayTime()))
                .build();
    }

    private String formatPlayTime(Short playTime) {
        int min = playTime / 60;
        int sec = playTime % 60;
        return String.format("%02d:%02d", min, sec);
    }
}
```

이 리팩터링의 주요 기술적 고려 사항은 다음과 같다:

- **책임 분리**: 음악 데이터 처리와 변환 로직을 `MusicService`로 분리함으로써 `PlaylistService`의 복잡도를 줄였다.
- **변환 로직 캡슐화**: DTO에서 도메인 객체로의 변환 로직을 `convertToMusic` 메서드로 캡슐화하여 재사용성과 유지보수성을 높였다.
- **포맷팅 로직 분리**: 시간 포맷팅 로직을 별도의 메서드로 분리하여 일관성을 유지하고 코드의 변경 용이성을 높였다.

## 3. CacheService - 캐싱

### 리팩터링 전

기존의 캐싱 로직은 `PlaylistService` 내부에 직접 포함되어 있었다. 이 접근 방식은 다음과 같은 문제를 야기했다:

- **코드 중복**: 캐싱 로직이 여러 군데에 중복되어 있었고, 변경 시 모든 관련 부분을 수정해야 했다.
- **유지보수 어려움**: 캐싱 구현이 `PlaylistService`와 혼합되어 있어 유지보수가 어려웠다.

### 리팩터링 후

`CacheService`를 도입하여 캐싱 로직을 중앙화하였다:

```java
@Service
@RequiredArgsConstructor
public class CacheService {
    private final RedisTemplate<String, Playlist> redisTemplate;
    private final PlaylistToDtoConverter converter;

    public void cachePlaylist(Playlist playlist) {
        String key = KeyGenerator.playlistKeyGenerate(playlist.getId());
        redisTemplate.opsForValue().set(key, playlist, Duration.ofMinutes(10));
    }

    public Optional<PlaylistResponseDto> getCachedPlaylist(String id) {
        String key = KeyGenerator.playlistKeyGenerate(id);
        Playlist cachedPlaylist = redisTemplate.opsForValue().get(key);
        return Optional.ofNullable(cachedPlaylist).map(converter::convert);
    }

    public void evictPlaylist(String id) {
        String key = KeyGenerator.playlistKeyGenerate(id);
        redisTemplate.delete(key);
    }
}
```

이 리팩터링의 주요 기술적 고려 사항은 다음과 같다:

- **캐싱 로직 중앙화**: 모든 캐싱 관련 로직을 `CacheService`로 중앙화하여 코드의 중복을 줄이고, 캐시 구현 변경 시 영향 범위를 최소화하였다.
- **Optional 사용**: `getCachedPlaylist` 메서드에서 `Optional`을 사용하여 null 체크를 명시적으로 처리하고, `NullPointerException` 발생 가능성을 줄였다.
- **TTL 설정**: 캐시된 데이터를 10분 동안 유지하도록 TTL(Time To Live)을 설정하여 데이터의 신선도를 유지하고, 메모리 사용을 효율적으로 관리하였다.

## 4. AsyncDatabaseService 도입

### 리팩터링 전

기존 데이터베이스 작업은 동기적으로 처리되었으며, 대량의 요청 처리 시 성능 병목이 발생할 수 있었다. 

### 리팩터링 후

`AsyncDatabaseService`를 도입하여 데이터베이스 작업을 비동기적으로 처리하였다:

```java
@Service
@RequiredArgsConstructor
public class AsyncDatabaseService {
    private final PlaylistRepository playlistRepository;

    @Async
    @Transactional
    public CompletableFuture<Playlist> savePlaylistAsync(Playlist playlist) {
        Playlist savedPlaylist = playlistRepository.save(playlist);
        return CompletableFuture.completedFuture(savedPlaylist);
    }

    @Async
    @Transactional
    public CompletableFuture<Void> deletePlaylistAsync(String id) {
        playlistRepository.deleteById(id);
        return CompletableFuture.completedFuture(null);
    }
}
```

이 리팩터링의 주요 기술적 고려 사항은 다음과 같다:

- **비동기 처리**: `@Async` 어노테이션을 사용하여 비동기적으로 데이터베이스 작업을 처리함으로써, 응답 지연을 줄이고 시스템의 전반적인 처리량을 개선하였다.
- **트랜잭션 관리**: `@Transactional` 어노테이션을 통해 트랜잭션을 관리하고, 데이터 일관성을 보장하며 예외 발생 시 롤백을 자동으로 처리하였다.
- **CompletableFuture 사용**: 비동기 작업의 결과를 `CompletableFuture`로 반환하여 비동기 작업의 완료를 추적하고,

 결과를 처리할 수 있게 하였다.

리팩터링 전의 코드는 봐주기 어려운 "기능만 돌아가는 코드"였는데, 리팩터링을 통해 이것 저것 생각도 많이해봤고, 코드만 봐도 꽤 깔끔해진 것 같다.
입사 전에 최대한 이런 부분 고민하는 연습을 많이 해야겠다.