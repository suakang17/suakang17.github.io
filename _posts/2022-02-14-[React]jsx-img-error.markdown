---
layout: post
title:  "[React]jsx에서 이미지가 엑박으로 나오는 에러"
date:   2022-02-05 15:44:13 +0900
categories: Devlife
---
# [React] jsx에서 이미지가 엑박으로 나오는 경우
## jsx파일 이미지 로드 안됨

리엑트에서 이미지를 띄우려고 했는데 결과를 확인하니 로드되지 않는 에러가 발생했다.
당시 나는 아래와 같이 작성했다.
```
<img className='header-logo' src='./img/header_logo.svg' />
```
처음 뜬 메시지는 다음과 같다.
```
img elements must have an alt prop, either with meaningful text, or an empty string for decorative images  jsx-a11y/alt-text
```
img는 alt prop을 반드시 가져야한다는 오류라, 우선 alt prop을 추가해주었다.
```
<img className='header-logo' src='./img/header_logo.svg' alt='headerlogo' />
```

##  alt 속성이란?






alt속성을 추가했는데도 엑박이 사라지지 않았다.

기존에 리액트가 아닌 일반 html을 작성시에 이런 에러가 뜨는 경우 항상 경로의 문제였기에 경로부터 재차 확인했는데 경로에는 문제가 없었다.

## require
```
<img className='header-logo' src={require('./img/header_logo.svg')} alt='headerlogo' /> 
```

## .default
```
<img className='header-logo' src={require('./img/header_logo.svg').default} alt='headerlogo' /> 
```