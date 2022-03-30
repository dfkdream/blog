---
title: 만료된 GPG 키 갱신하기 (유효기간 연장)
date: 2022-03-30 00:00:00 +0900
categories: Environment Git GPG
comments: true
---

Github 등 Git 서버에서는 커미터가 자신임을 증명하기 위해 GPG 서명을 사용할 수 있습니다.
이때 사용되는 GPG 키는 보안을 위해 유효기간을 지정하는 것이 권장되는데, 유효기간이 지나면 해당 키로 커밋을 할 수 없는 등 문제가 발생합니다.

이런 문제가 발생했을 때 GPG 키를 갱신하는 방법입니다.

# 테스트 환경
* macOS Monterey
* gpg (GnuPG) 2.3.4

# 저장된 키 목록 확인

```zsh
gpg --list-keys
```

```
<keybox 파일 경로>
----------------------------------
pub   rsa4096 <생성일> [SC] [expires: <만료일>]
      <Key ID>
uid           [ultimate] <username> <<email>
sub   rsa4096 <생성일> [E] [expires: <만료일>]
```

`Key ID`는 유효기간 연장을 위해 필요하니 잘 확인하시기 바랍니다.

# 키 유효기간 연장

```zsh
gpg --edit-key <Key ID>
```
아무것도 입력하지 않으면 ` Primary key`를 수정하게 됩니다.
```
gpg> expire
Changing expiration time for the primary key.
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 
```

n+단위를 입력해 만료일을 오늘 날짜로부터 n만큼 연장시킵니다.

`Subkey`의 유효기간도 연장시켜 줍시다.

```
gpg> key 1
gpg> expire
```

이후의 명령은 `Primary key`와 동일합니다.

마지막으로 수정 내용을 저장합니다.

```
gpg> save
```

# 공개키 내보내기
```zsh
gpg --armor --export <Key ID>
```
지정된 키를 텍스트 형식으로 내보냅니다. 표준 출력으로 키 내용이 출력되니 리디렉션을 사용해 클립보드에 바로 복사할 수도 있습니다.

이제 Git 서버 관리 페이지에서 기존 공개키를 삭제하고 내보낸 키를 다시 등록하면 됩니다.