---
title: "scanf()의 format string에 줄바꿈을 사용하면 안 되는 이유"
date: 2018-09-12 12:18:00 +0900
categories: CS C
comments: true
---

```C
#include <stdio.h>

int main(){
    int i=0;
    scanf("%d\n",&i);
    printf("%d\n",i);
    scanf("%d",&i);
    printf("%d\n",i);
    return 0;
}
```

`scanf`는 문자가 매칭이 안 되고 다음 문자가 whitespace가 아닐 때 까지 읽고 문자열이 남을 경우 나머지를 버퍼에 남긴다.

`scanf`에 \n이 들어갈 경우 줄바꿈이 매칭되므로 `scanf`가 종료되지 않고 다음 readable character가 들어올 때 까지 대기한다.

이 예시의 경우 format string에 conversion specification이 하나밖에 없으므로 첫 readable character을 `i`에 쓰고 버퍼에 다음 character를 남긴다.
그래서 다음 `scanf()` 호출 시에 아무것도 입력하지 않아도 그 전에 입력했던 문자열을 버퍼에서 읽어와 출력하게 된다.

(K&K 2/E p.46 참고)

결론: scanf 끝에 줄바꿈 문자를 넣지 말자

특별한 상황에서는 쓸 수도 있겠지만... :)