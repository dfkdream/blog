---
title: "Python3 CLI 스크립트에서 표 그리기"
date: 2018-09-19 00:45:00 +0900
tags: ["CS", "Python"]
comments: true
---
데이터와 행 이름을 리스트로 넣으면 표를 출력해 주는 함수입니다.

한글을 포맷팅하면 전각 한글을 한 글자로 계산해 표가 제대로 출력되지 않는데, 해당 문제를 해결해 두었습니다.
다른 전각 문자를 사용하려면 `reKo` 부분에 해당 문자 구간을 추가하면 됩니다.

제어 문자열 `"SEP"`를 데이터 리스트에 포함시키면 행 구분을 출력합니다. 

## Source Code
~~~python
import re

def drawTable(identifiers,datas):
    reKo=re.compile("[가-힣]")
    maxLength=[len(str(x))+len(reKo.findall(str(x))) for x in identifiers]
    for data in datas:
        for index,d in enumerate(data):
            if d=="SEP": continue
            if maxLength[index]<len(str(d))+len(reKo.findall(str(d))):
                maxLength[index]=len(str(d))+len(reKo.findall(str(d)))
    divider="-"*(sum(maxLength)+(len(maxLength)+1)+len(maxLength)*2)
    print(divider)
    formatString="|"+"".join([" {0}{1}:^{2}{3} |".format("{",index,length-len(reKo.findall(str(identifiers[index]))),"}") for index,length in enumerate(maxLength)])
    print(formatString.format(*identifiers))
    print(divider)
    for data in datas:
        if data=="SEP":
            print(divider)
            continue
        formatString="|"+"".join(" {0}{1}:^{2}{3} |".format("{",index,length-len(reKo.findall(str(data[index]))),"}") for index,length in enumerate(maxLength))
        print(formatString.format(*data))
    print(divider)
~~~

### 실행 예시
~~~python
drawTable(["a","b","합계"],[[1,2,3],[4,5,9],"SEP",[5000,5000,10000]])
~~~

### Output
**주의: 고정폭 폰트를 사용해야 표가 제대로 표시됩니다.**
~~~
-----------------------
|  a   |  b   | 합계  |
-----------------------
|  1   |  2   |   3   |
|  4   |  5   |   9   |
-----------------------
| 5000 | 5000 | 10000 |
-----------------------
~~~