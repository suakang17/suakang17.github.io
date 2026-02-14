---
layout: post
title:  "[Java] List.subList 사용 시 발생하는 Memory Leak과 GC 동작 원리"
date:   2026-01-03 00:00:00 +0900
categories: Devlife
---

# [Java] List.subList 사용 시 발생하는 Memory Leak과 GC 동작 원리

최근 코드 리뷰 중, List.subList의 내부 동작 방식에 대한 오해로 인해 자칫하면 심각한 메모리 누수를 유발할 뻔한 로직을 발견했다.

일반적으로 더 이상 사용하지 않는 객체 변수에 null을 할당하면 GC가 해당 메모리를 수거해 갈 것이라고 기대한다. 하지만 자바의 메모리 관리 구조, 특히 뷰와 참조의 개념을 명확히 이해하지 못하면 트래픽이 몰릴 때 OOM상황을 맞이할 수 있다.

## 1. '복사'가 아닌 '뷰(View)'의 의미

메모리 누수가 발생하는 핵심 원인은 subList가 데이터를 복사해서 새로운 리스트를 만드는 것이 아니라, 원본 데이터를 바라보는 창문(View) 역할만 수행한다는 점에 있다.

### Copy vs View

Copy: 데이터를 잘라내어 새로운 메모리 공간에 할당한다. 원본과 독립적인 객체가 된다.

View: 새로운 메모리를 할당하지 않고, "원본 배열의 인덱스 0번부터 9번까지"라는 메타 정보(Offset, Size)만 가진다. 즉, 원본 배열을 내부적으로 공유한다.

## 2. 참조 해제(null)가 예상대로 동작하지 않는 이유

1GB 크기의 대용량 리스트를 로드한 뒤 앞부분 10개만 필요하여 자르는 시나리오를 가정해 보자.

```java
// 1. 100만 개 데이터 로딩 (약 1GB 가정)
List<String> hugeList = loadHugeData(); 

// 2. 앞 10개만 필요해서 자름 (View 생성)
List<String> smallList = hugeList.subList(0, 10);

// 3. 거대한 리스트는 이제 필요 없으니 참조 해제
hugeList = null; 

// 4. smallList를 캐시나 전역 변수에 저장해서 계속 사용
return smallList;
```

코드상으로는 hugeList에 null을 할당했으므로 원본 데이터가 GC 대상이 될 것처럼 보인다. 하지만 실제로는 1GB 메모리가 힙(Heap) 영역에 그대로 남게 된다.

그 이유는 ArrayList.subList가 생성하는 객체의 내부 구현을 살펴보면 알 수 있다.

```java
// 실제 Java ArrayList 내부의 SubList 클래스 (단순화)
private class SubList extends AbstractList<E> {
    private final AbstractList<E> parent; // 원본 리스트를 필드로 참조함

    SubList(AbstractList<E> parent, ...) {
        this.parent = parent; // 여기서 강한 참조(Strong Reference) 발생
    }
}
```

subList로 만들어진 smallList는 내부적으로 원본 리스트(parent)를 강하게 참조(Strong Reference)하고 있다. 따라서 smallList가 살아있는 한, 원본인 hugeList의 데이터 역시 GC에 의해 수거되지 않는다. 변수(hugeList)와의 연결만 끊어졌을 뿐, 객체 간의 연결은 유지되고 있기 때문이다.

### 해결책: 방어적 복사 

이 문제를 해결하려면 뷰(View)가 아닌 실제 데이터의 복사본을 만들어 원본과의 참조 고리를 끊어야 한다.

```java
// 원본과의 연결 고리를 끊고, 새로운 리스트 생성 (Deep Copy)
List<String> smallList = new ArrayList<>(hugeList.subList(0, 10));

hugeList = null; // 이제 원본 데이터는 참조가 사라져 GC 대상이 된다.
```

## 3. GC의 수거 기준: Reachability

이 현상을 정확히 이해하려면 자바 GC가 객체의 생존 여부를 판단하는 기준인 Reachability(도달 가능성)를 알아야 한다.

Root Set: 스택 변수, 정적(Static) 변수 등 현재 실행 중인 코드에서 유효한 참조들이다.

Reachable: Root Set에서 시작하여 참조를 타고 도달할 수 있는 모든 객체는 살아있는 것으로 간주한다.

위 예제에서 smallList가 애플리케이션 내에서 계속 사용된다면(Reachable), smallList가 참조하고 있는 parent(원본 1GB 데이터) 역시 Reachable 상태로 판정된다. GC는 도달 불가능한(Unreachable) 객체만 수거하기 때문에 원본 데이터는 메모리에 계속 상주하게 된다.

## 4. WeakHashMap 사용 시 주의할 점

메모리 관리를 위해 WeakHashMap을 캐시로 사용하는 경우가 종종 보이기도 한다. 하지만 이 자료구조 역시 정확한 동작 방식을 이해하고 사용해야 한다.

WeakHashMap은 Key에 대한 참조가 Weak인 맵이다. 즉, Key 객체가 더 이상 다른 곳에서 참조되지 않을 때 해당 엔트리를 삭제한다. Value의 메모리 점유 상태와는 무관하다.

만약 캐시의 Key로 String 리터럴이나 Integer 같은 상수를 사용한다면 주의가 필요하다. 자바의 String Pool 등으로 인해 해당 객체들은 프로그램 종료 시까지 강한 참조로 남아있을 가능성이 높기 때문이다. Key가 수거되지 않으면 Value에 저장된 데이터 역시 영원히 메모리에 남게 되어 누수의 원인이 될 수 있다.

따라서 범용적인 데이터 캐싱이 목적이라면 Caffeine이나 Ehcache와 같이 LRU 알고리즘 등을 지원하는 검증된 캐시 라이브러리를 사용하는 것이 적합하다.

## 요약

List.subList는 원본 리스트의 뷰이므로 원본 객체에 대한 참조를 유지한다.

변수에 null을 할당하더라도, 다른 객체(View)가 원본을 참조하고 있다면 GC 대상이 되지 않는다.

대용량 데이터의 일부분만 사용할 때는 반드시 방어적 복사(new ArrayList<>(...))를 통해 원본과의 참조를 끊어주는 것이 안전하다.
