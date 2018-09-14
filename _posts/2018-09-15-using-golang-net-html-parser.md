---
title: "Golang HTML 파서 사용하기"
date: 2018-09-15 02:20:00 +0900
category: CS Golang
comments: true
---

Golang으로 애플리케이션 개발을 하면서 HTML 파서를 사용할 일이 가끔 있었습니다.
`golang.org/x/net/html` 파서는 Python의 `BeautifulSoup`처럼 사용하기 편하게 구성되어 있지 않아 사용법을 간단히 정리해 보았습니다.

# Codes
예제 코드의 실행 결과에는 아래 HTML 노드를 사용했습니다. 편의상 예제 코드에서는 생략하도록 하겠습니다.

*아래 코드는 HTML 문서 전체가 아닌 노드 일부입니다. 문서 전체가 아닌 일부만을 html.Parser 함수에 전달할 경우 파서가 누락된 노드들(`<html>`,`<head>`,`<body>`등)을 알아서 추가합니다.*

~~~html
<ul>
	<li>Item 1</li>
	<li>Item 2</li>
	<li><a href="http://www.example.com">Item 3</a></li>
</ul>
~~~

## 자식 노드 전체 탐색
HTML 파싱에는 일반적으로 이 코드를 사용합니다. 자식 노드 전체를 재귀적으로 탐색합니다.
### Code
~~~go
var f func(*html.Node)
f=func(n *html.Node){
	if n.Type == html.ElementNode{
		fmt.Println(n.Data)
	}
	for c:=n.FirstChild; c != nil; c = c.NextSibling{
		f(c)
	}
}
f(node)
~~~
### 결과
~~~
ul
li
li
li
a
~~~

## 특정 자식 노드의 Text 읽기
### Code
~~~go
var f func(*html.Node)
f=func(n *html.Node){
	if n.Type == html.ElementNode && n.Data=="li"{
		fmt.Println(n.FirstChild.Data)
	}
	for c:=n.FirstChild; c != nil; c = c.NextSibling{
		f(c)
	}
}
f(node)
~~~
### 결과
~~~
Item 1
Item 2
a
~~~
`Item 1`과 `Item 2`의 경우 해당 텍스트 노드가 `<li>` 태그의 첫 번째 자식 노드(FirstChild)이기 때문에 정상적으로 출력될 수 있었습니다. 그렇지만 `Item 3`의 경우 `<li>` 태그의 첫 번째 자식 노드가 텍스트 노드가 아닌 `<a>` 태그이기 때문에 `Item 3`가 아닌 `a`가 출력이 되었습니다. 이러한 경우를 방지하기 위해 저는 n.FirstChild.Data 대신 아래에 나오는 `renderNode` 함수를 사용합니다.
### Code (renderNode 함수 사용)
~~~go
var f func(*html.Node)
f=func(n *html.Node){
	if n.Type == html.ElementNode && n.Data=="li"{
		fmt.Println(renderNode(n.FirstChild))
	}
	for c:=n.FirstChild; c != nil; c = c.NextSibling{
		f(c)
	}
}
f(node)
~~~
### 결과
~~~
Item 1
Item 2
<a href="http://www.example.com">Item 3</a>
~~~
다만, renderNode를 사용할 경우 파싱 결과에 HTML이 포함됩니다. 파싱 결과에 HTML이 포함되어도 되는 경우에만 사용할 수 있겠습니다.

## HTML 노드 렌더링
### Code
[stackoverflow](https://stackoverflow.com/questions/30109061/)를 참고했습니다.
~~~go
func renderNode(n *html.Node) string {
	var buf bytes.Buffer
	w := io.Writer(&buf)
	html.Render(w, n)
	return buf.String()
}
~~~
### 결과
~~~HTML
<ul>
	<li>Item 1</li>
	<li>Item 2</li>
	<li><a href="http://www.example.com">Item 3</a></li>
</ul>
~~~

## 하위 노드 전체 렌더링하기
### Code
~~~go
var result ="" 
for d := n.FirstChild; d != nil; d = d.NextSibling {
	result += renderNode(d)
}
~~~
### 결과
~~~HTML
	<li>Item 1</li>
	<li>Item 2</li>
	<li><a href="http://www.example.com">Item 3</a></li>
~~~

## Attribute 값 읽기
### Code
~~~go
var f func(*html.Node)
f=func(n *html.Node){
	if n.Type == html.ElementNode && n.Data=="a"&& len(n.Attr)>0&&n.Attr[0].Key=="href"{
		fmt.Println(n.Attr[0].Val)
	}
	for c:=n.FirstChild; c != nil; c = c.NextSibling{
		f(c)
	}
}
f(node)
~~~
### 결과
~~~
http://www.example.com
~~~
`len(n.Attr)>0`을 추가하지 않을 경우 Attribute가 없는 태그(예시의 경우 `<a>`)를 만났을 때 `runtime error: index out of range` Panic이 발생합니다.

***

`net/html` 파서를 사용하기 불편하다면 [goquery](https://github.com/PuerkitoBio/goquery) 등의 다른 라이브러리를 사용하는 것도 좋겠습니다.