---
title: macOS에 pyenv 설치, 사용하기
date: 2022-03-08 00:00:00 +0900
tags: ["Environment", "macOS", "Python"]
comments: true
---

pyenv, pyenv-virtualenv 설치 방법과 자주 사용되는 명령어들을 모아 보았습니다.

# 테스트 환경
* macOS Monterey
* iTerm2
* ZSH

# pyenv 설치

```zsh
brew install pyenv

echo 'eval "$(pyenv init --path)"' >> ~/.zprofile

echo 'eval "$(pyenv init -)"' >> ~/.zshrc
```

# 특정 버전의 python 설치

```zsh
pyenv install <python-version>
```

ex)

```zsh
pyenv install 3.8.12
```

# pyenv-virtualenv 설치

```zsh
brew install pyenv-virtualenv

echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.zshrc
```

# virtualenv 생성

```zsh
pyenv virtualenv <python-version> <venv-name>
```

ex)

```zsh
pyenv virtualenv 3.8.12 venv
```

# virtualenv 적용

```zsh
pyenv activate <venv-name>
```

ex)

```zsh
pyenv activate venv
```

# 특정 폴더에 virtualenv 자동 적용

```zsh
pyenv local <venv-name>
```

ex)

```zsh
pyenv local venv
```

실행 시 현재 디렉토리에 `.python-version` 파일이 생성되고 pyenv가 이 파일을 자동으로 인식해 가상 환경을 적용해 줍니다.

# virtualenv 삭제

```zsh
pyenv uninstall <venv-name>
```

ex)
```zsh
pyenv uninstall venv
```