---
layout: post
title:  "[git] 계층형 이름의 branch push 이슈"
date:   2023-10-25 23:34:10 +0900
categories: Devlife
---
# [git] 계층형 이름의 branch push 이슈
**src refspec feat/auth does not match any, cannot lock ref branch, cannot create branch 이슈**

## 발단

- remote: `main` → `feat/auth` (main에서 새로 생성한 브랜치)
- local: `feat/auth/signup` (pulled from remote `feat/auth`)

`feat/auth/signup` 에서 작업 후 origin `feat/auth` or origin `feat/auth/signup`으로 push 하려고 하니 아래 에러가 발생

```
error: src refspec feat/auth does not match any
error: failed to push some refs to 'https://github.com/team-waff1e/Backend.git'
```

```
To https://github.com/team-waff1e/Backend.git
 ! [remote rejected] feat/auth/signup -> feat/auth/signup (cannot lock ref 'refs/heads/feat/auth/signup': 'refs/heads/feat/auth' exists; cannot create 'refs/heads/feat/auth/signup')
error: failed to push some refs to 'https://github.com/team-waff1e/Backend.git'
```


## 원인

git branch naming 문제로 upstream branch랑 비슷한 (폴더형 - 계층형) 구조의 이름은 쓸 수 없다고 한다.

브랜치는 분기이므로 분기 `ex`가 존재하는 경우 `ex/ex1` 같은의 하위 분기는 생성할 수 없다고 한다. 

**해당 명칭의 브랜치를 로컬에 갖고 있던 말던 그게 remote에 이미 존재하는게 문제**라고 한다. 

> It's not a *folder* that exists, it's a *branch*. (Well, there may be a folder/directory involved somewhere—or maybe not, as references get "packed" and stop existing as files within directories.) <br> - **If branch `b` exists, no branch named `b/anything` can be created.** <br> - **Likewise, if branch `dev/b` exists, `dev/b/c` cannot be created.** <br> [출처: stackoverflow](https://stackoverflow.com/a/22630664)

그럼 로컬에서는 왜 되었냐 지금까지

- remote : `refs/remotes/origin/branchname/~~`
- origin : `refs/heads/branchname/~~`

브랜치 분기가 이런식으로 갈리기 때문이다.

## 해결

문제는 네이밍이니 브랜치 이름을 바꿔주어서 해결했다. 
```
$ git branch -m [NEW_BRANCH]
```