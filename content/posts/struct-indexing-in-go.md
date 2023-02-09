---
title: "[Golang] 구조체 필드를 배열처럼 접근하기"
date: 2019-04-05 02:25:00 +0900
tags: ["CS", "Golang"]
comments: true
---

리플렉션 (reflect 패키지) 를 사용하면 구조체를 배열처럼 인덱스로 접근할 수 있습니다.

~~~go
import(
    "fmt"
    "reflect"
    "strconv"
)

type test struct{
    A int
    B int
    C string
}
~~~

## 값 읽기
~~~go
t := test{1,2,"3"}

fmt.Println(reflect.ValueOf(t).Field(0).Int())
fmt.Println(reflect.ValueOf(t).Field(1).Int())
fmt.Println(reflect.ValueOf(t).Field(2).String())
~~~

### 결과
~~~
1
2
3
~~~

자료형이 일치하지 않을 경우 패닉이 발생합니다.
~~~go
t := test{1,2,"3"}

fmt.Println(reflect.ValueOf(t).Field(0).Int())
fmt.Println(reflect.ValueOf(t).Field(1).Int())
fmt.Println(reflect.ValueOf(t).Field(2).Int())
~~~

### 결과

~~~
1
2
panic: reflect: call of reflect.Value.Int on string Value
~~~

그러므로 값을 읽기 전에는 `Kind()` 함수를 이용해 타입 검사를 해 주어야 합니다.
~~~go
t := test{1,2,"3"}

for i:=0;i<3;i++{
    if reflect.ValueOf(t).Field(i).Kind()==reflect.String{
        fmt.Println(reflect.ValueOf(t).Field(i).String())
    }else{
        fmt.Println(reflect.ValueOf(t).Field(i).Int())
    }
}
~~~
### 결과
~~~
1
2
3
~~~

## 값 쓰기
~~~go
var t test

reflect.ValueOf(&t).Elem().Field(0).SetInt(int64(1))
reflect.ValueOf(&t).Elem().Field(1).SetInt(int64(2))
reflect.ValueOf(&t).Elem().Field(2).SetString("3")

fmt.Println(t)
~~~

### 결과
~~~
{1 2 3}
~~~

값을 읽는 코드와 비슷합니다. 다만 구조체 필드의 값을 변경하기 위해 `ValueOf` 함수의 파라미터로 포인터를 넘겼고 역참조를 위해 `Elem()` 함수를 사용하고 있습니다.

값 쓰기의 경우에도 읽기와 마찬가지로 자료형이 일치하지 않을 경우 패닉이 발생합니다.
~~~go
var t test

reflect.ValueOf(&t).Elem().Field(0).SetInt(int64(1))
reflect.ValueOf(&t).Elem().Field(1).SetInt(int64(2))
reflect.ValueOf(&t).Elem().Field(2).SetInt(int64(3))

fmt.Println(t)
~~~

### 결과
~~~
panic: reflect: call of reflect.Value.SetInt on string Value
~~~

그러므로 `Kind()` 함수를 이용해 타입 검사를 해 주어야 합니다.

~~~go
var t test

a:=[]int{1,2,3}

for i,n:=range a{
    if reflect.ValueOf(t).Field(i).Kind()==reflect.String{
        reflect.ValueOf(&t).Elem().Field(i).SetString(strconv.Itoa(n))
    }else{
        reflect.ValueOf(&t).Elem().Field(i).SetInt(int64(n))
    }
}

fmt.Println(t)
~~~

### 결과
~~~
{1 2 3}
~~~

쓰기의 경우 읽기와는 다르게 구조체 필드가 내보내기(Export)되지 않았을 경우(=앞 글자가 대문자가 아닐 경우) 패닉이 발생하므로 주의해야 합니다.