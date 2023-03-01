---
title: "[프로젝트] Obsidian으로 작성된 문서 Markdown으로 변환하기"
date: 2023-02-16 15:00:00 +0900
tags: ["blog", "hugo"]
categories: ["프로젝트 (진행 중)"]
comments: true
---

프로젝트 링크: [https://github.com/dfkdream/obsidian_to_hugo](https://github.com/dfkdream/obsidian_to_hugo)

# 개요
Obsidian은 내부 링크에 WikiLink 문법을 사용하는데, Hugo가 사용하는 Markdown 문법과 호환되지 않는다. Obsidian으로 작성된 문서를 바로 Hugo에 배포하고 싶어 이 프로젝트를 기획하게 되었다.

# 목표

## 주 목표
WikiLink가 사용된 문서를 파싱해 Markdown 링크로 변환해 주는 Github Actions 개발

## 부가 목표
매우 괜찮다고 생각하는 Obsidian 기능인 Graph View를 웹사이트 상에서도 사용할 수 있었으면 좋겠다. 문서 그래프를 작성해 json 형식으로 내보낼 수 있게 하기.

# 분석

### WikiLink 문법
Obsidian 자체도 [markdown link로 변환 기능](https://help.obsidian.md/Linking+notes+and+files/Internal+links) 을 제공하기는 한다. 하지만 Hugo로 빌드하는 경우 제대로 된 링크가 걸리지 않는 문제가 있다. Obsidian은 링크 생성 시 Hugo Permalink 설정을 참조하지 않기 때문이다. 이 부분이 주요 과제가 될 듯 하다.

#### Link To File
* WikiLink
```
[[file name.ext|alt]]
```
* Markdown
```
[alt](file-name.ext)
```
* 기본 변환 기능 작동 여부: N

#### Link To Image
* WikiLink
~~~
![[filename.ext|alt]]
~~~
* Markdown
~~~ 
![alt](filename.ext)
~~~
* 기본 변환 기능 작동 여부: N

#### Link To Heading in a note
* WikiLink
~~~
[[file name.ext#heading name|alt]]
~~~
* Markdown
~~~ 
[alt](file-name.ext#heading-name)
~~~
* 기본 변환 기능 작동 여부: N

#### Link To Block in a note
* WikiLink
~~~
[[file name.ext#^uid|alt]]
~~~
* Markdown: 지원하지 않음
* 기본 변환 기능 작동 여부: N
* Note: 사용 시 지정된 블럭 마지막 단어 뒤에 uid가 생성됨 (`^uid`). 

#### External Link
* WikiLink: 지원하지 않음
* Markdown: [[#Link To File]]과 동일

# 구현 계획
1. [X] Filename - Permalink Key-Value 페어 생성기 구현
2. [ ] Wikilink, Markdown 파서 구현
3. TBD