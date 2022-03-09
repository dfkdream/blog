---
title: zsh-autosuggest와 zsh-syntax-highlighting 충돌 문제 해결
date: 2020-02-21 00:00:00 +0900
categories: Environment macOS ZSH
comments: true
---
[zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions)와 [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting)은 zsh 사용을 엄청 편리하게 해 주는 플러그인입니다. 그런데 macOS에서 이 플러그인들을 설치했더니 플러그인 간에 충돌이 발생해 자동완성된 명령어가 보이지 않거나 하이라이팅이 제대로 되지 않는 문제가 발생했습니다. 이 포스트에서는 이런 문제들을 해결해 보겠습니다.

# 테스트 환경
* macOS Big Sur
* iTerm2 Build 3.3.7
* oh-my-zsh

# 플러그인 설치
먼저 더 많은 명령어 자동 완성 목록을 사용하기 위해 [zsh-completions](https://github.com/zsh-users/zsh-completions)를 설치해 줍시다.
```zsh
% git clone https://github.com/zsh-users/zsh-completions ${ZSH_CUSTOM:=~/.oh-my-zsh/custom}/plugins/zsh-completions
```
다음으로 [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting
)과 [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions)를 설치해 주겠습니다.
```zsh
% git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
% git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```
설치가 모두 끝나면 `~/.zshrc` 파일을 다음과 같이 수정해 플러그인을 활성화해 주겠습니다.
```zsh
plugins=(
    [Plugins...]
    zsh-syntax-highlighting
    zsh-completions
    zsh-autosuggestions
)
...
autoload -U compinit && compinit
```
가장 마지막 줄의 `autoload...`는 zsh-completions 활성화를 위한 부분입니다. 파일 가장 마지막 줄에 추가해 줍시다.

설치가 모두 끝났습니다! 마지막으로 다음 명령어를 입력해 설정 파일을 로드해 주겠습니다.
```zsh
source ~/.zshrc
```

이러면 이제 자동완성과 문법 하이라이팅이 이쁘게 되어야 하는디...
# Troubles
이대로 설정하면 데모에서 본 것처럼 표시가 되지 않습니다. 제 경우에는
1. 자동완성된 부분이 입력한 부분과 구분이 되지 않음
2. 자동완성된 부분 글자가 하나만 보이고 나머지는 보이지 않음

이런 문제가 발생했습니다.

# Troubleshooting
## 자동완성된 부분이 입력한 부분과 구분이 되지 않음
먼저 1번 문제부터 고쳐 보겠습니다. 이 문제는 zsh-syntax-highlighting이 macOS Big Sur에 기본으로 설치된 `zsh 5.8 (x86_64-apple-darwin20.0)`을 인식하지 못해 발생하는 문제입니다. brew를 사용해 zsh를 업데이트(설치)해 줍시다.
```zsh
% brew install zsh
```
이제 터미널을 새로 열면 zsh가 업데이트된 것을 확인할 수 있습니다.
```zsh
% zsh --version
zsh 5.8 (x86_64-apple-darwin20.1.0)
```
(실행 시점과 OS버전 / 아키텍쳐에 따라 다를 수 있습니다.)

이 문제는 이제 해결!

## 자동완성된 부분 글자가 하나만 보이고 나머지는 보이지 않음
그런데.. 기본 테마가 아닌 `Solarized Dark` 등의 컬러 스킴을 사용하시는 경우 자동완성된 부분 글자가 제대로 보이지 않는 문제가 발생합니다. 이건 zsh-autosuggestions가 기본 자동완성 하이라이팅 색상으로 사용하는 `ANSI Bright Black` 색상이 Background 색상과 동일하게 설정되어 있어 생기는 문제입니다.

iTerm2 설정 (cmd+,) > Profiles > Colors > ANSI Colors > Black > Bright 색상을 다른 색으로 변경해 줍시다. 저의 경우 `#2a5965`로 설정했습니다. 

이러면 이제 진짜로 설정 끝!

# 참고문헌 / 출처
* [본격 macOS에 개발 환경 구축하기 - Subicura's Blog](https://subicura.com/2017/11/22/mac-os-development-environment-setup.html)
* [autosuggestion not working for oh-my-zsh #416 - zsh-users/zsh-autosuggestion](https://github.com/zsh-users/zsh-autosuggestions/issues/416#issuecomment-486516333)
* [Conflict with zsh-autosuggestions under zsh 5.8 (and only that version) #756 - zsh-users/zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting/issues/756)