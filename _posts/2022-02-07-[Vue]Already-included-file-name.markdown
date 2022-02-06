---
layout: post
title:  "[Vue]Already included file name 에러"
date:   2022-01-05 15:44:13 +0900
categories: Devlife
---

# [Vue.js] Already included file name 오류
VS CODE로 Vue 작업 중 파일 이름을 남이 봐도 무슨 파일인 줄 알 수 있을만한 것으로 변경을 했는데 대뜸 오류가 출현했다 !! 
```
Already included file name 'C:/Users/.../파일이름1.vue' differs from file name 'C:/Users/.../파일이름2.vue' only in casing.
```

결국 파일명을 파일이름1로 이미 소스에 추가해뒀는데, 내가 파일명을 파일이름2로 변경해버렸고 그래서 미스매치가 벌어졌다는 뜻이다.

해결방법은 두가지가 있다.


# 해결법 1
 VS CODE를 종료했다가 재시작한다.
 알아서 변경사항(이번 경우는 파일명) 업데이트를 해준다.

 # 해결법 2
 'jsconfig.json' 파일에 들어가서 아래 코드를 추가해주자.
 ```
"forceConsistentCasingInFileNames": false
 ```

 여기서 혹시 또 에러가 난다면 'compilerOptions'항목 간 ','를 잊지 않았는지 체크하기.

 
 
출처 [stack overflow][stack-overflow]

 [stack-overflow] : https://stackoverflow.com/questions/51197940/file-name-differs-from-already-included-file-name-only-in-casing-on-relative-p#