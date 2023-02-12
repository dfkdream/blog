---
title: "Jekyll 블로그 Hugo로 이전한 후기"
date: 2023-02-13 01:06:00 +0900
tags: ["blog", "hugo"]
categories: ["블로그 관련"]
comments: true
---

Jekyll + Minimal Mistakes로 만들었던 블로그를 [Hugo](https://gohugo.io/)로 이전했습니다.

# 왜 Hugo인가?
* 가장 큰 이유는, Ruby보다는 Hugo가 사용하는 언어인 Go에 경험이 많았기 때문입니다. 사실 이 점이 블로그 글 작성에 큰 영향을 미치지는 않습니다. 실제 사용시에는 Hugo cli 명령어 정도만 사용하니까요. 하지만 혹시 모를 문제 발생시에 해결하기 훨씬 수월할 것 같다는 안정감이 들었습니다.
* Hugo를 사용한 정적 웹사이트 개발 프로젝트를 수행했던 경험이 있고, Hugo가 사용하는 템플릿 언어인 [text/template](https://pkg.go.dev/text/template) 을 자주 사용했습니다.
* macOS에서 빌드 속도가 Jekyll에 비해 매우 빠릅니다. Linux에서는 비슷한 성능을 보여줬지만, macOS의 경우 Jekyll은 체감 빌드 시간이 1분 정도 걸리는 데 비해 Hugo는 대략 3초 이내로 빌드가 완료됩니다.
* Hot Reload를 지원해 테마 개발이 편리합니다.
* Google Analytics, 목차 생성 기능 등 기본 제공되는 기능들이 풍부합니다.
* 테마를 git submodule로 관리해 소스트리가 깔끔합니다.

# 단점은?
* Github pages에서 기본적으로 지원하지 않습니다. Jekyll의 경우, 별도 설정 없이 페이지 빌드와 배포가 자동으로 수행되는 반면, Hugo의 경우 Github Actions를 수동으로 설정해 주어야 합니다.

단점이 별로 없는 것 같지만, 위 단점이 생각보다 많이 컸습니다. 그래서 사실 Go를 많이 좋아하고, 경험이 있는 것이 아니라면 그냥 Jekyll을 사용하는 것이 나을지도 모르겠네요.


Hugo를 사용한 또 다른 이유에는 [PaperMod](https://themes.gohugo.io/themes/hugo-papermod/) 테마를 사용하기 위한 것도 있었습니다. 미니멀한 디자인이고, 퍼지 검색 기능을 기본적으로 지원한다는 점이 마음에 들었어요.

# 이전 과정
## 1. Hugo 초기화 및 기존 블로그 정리
먼저, 기존 블로그 디렉토리에서 Hugo 사이트를 생성했습니다. 디렉토리가 비어 있지 않기 때문에 `--force`를 사용해 강제로 초기화합니다.
```zsh
hugo new site . --force
```
그런 다음, 기존 포스트 파일들을 `content/posts` 디렉토리로 이동하고, 파일명 앞에 붙어 있던 날짜를 제거해 주었습니다. Hugo가 permalink 생성에 파일명을 사용하기 때문인데, 개인적으로는 날짜를 붙일 수 있는 Jekyll 방식이 더 마음에 듭니다. 파일명 정렬이 쉬워지니까요. Hugo에서도 이 방식을 사용할 수 있는지 찾아보아야겠습니다. 혹시 아시는 분 있으시면 댓글에 알려주세요.
```zsh
for file in *; do mv $file $(echo $file | cut -c12-) ; done
```

루트 디렉터리에 두었던 `CNAME`, `favicon.ico` 파일 등은 `static` 디렉토리로 이동합니다.

## 2. 테마 및 Hugo 설정
`themes` 폴더 아래에 테마 리포지터리를 git submodule로 추가해 줍니다. 저는 PaperMod를 사용했습니다.
```zsh
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```
테마의 `config.yml`파일을 참고해 Hugo를 설정해 줍니다. 주요 수정 내용은 다음과 같습니다.
```yaml
# 읽기 시간 계산등에 한국어에 맞는 계산법 적용
hasCJKLanguage: true

# fuse 검색을 위해 JSON 인덱스 생성
outputs:
  home:
    - HTML
    - RSS
    - JSON

# SEO를 위해 Jekyll과 동일하게 permalink 생성
permalinks:
  posts: /:2006/:01/:02/:filename
```
## 3. Github Actions 설정
[Hugo 문서](https://gohugo.io/hosting-and-deployment/hosting-on-github/#build-hugo-with-github-action)를 참고해 `.github/workflows/gh-pages.yml` 파일을 작성했습니다. 주의할 점은 `on` 설정과 `Deploy` step의 `if`에 브랜치 이름이 `main`으로 하드코딩되어 있다는 점입니다. 제 블로그는 `master`브랜치를 사용하기 때문에, 문서에 있는 파일을 그대로 사용했을 때 제대로 빌드되지 않는 문제가 발생했습니다.


빌드가 완료되면 결과물이 `gh-pages` 브랜치에 푸시됩니다. 이 브랜치를 github pages 배포에 사용하도록 프로젝트를 설정합니다.
![github pages 설정](/posts/Pasted_image_20230213014815.png)