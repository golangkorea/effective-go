# 웹 서버
* 원문 : [A web server](https://golang.org/doc/effective_go.html#web_server)
* 번역자 : MinJae Kwon (@mingrammer)


완전한 Go 프로그램인 웹 서버로 마무리를 짓자. 이것은 실제로 웹 서버의 한 종류이다. 구글은 데이터를 자동으로 차트와 그래프로 만들어주는 <code>chart.apis.google.com</code> 서비스를 제공한다. 이것은 URL에 데이터를 실어서 쿼리로 만들 필요가 있기 때문에, 쉬워 보임에도 불구하고, 인터랙티브하게 사용하기가 어렵다. 여기서 주어진 프로그램은 데이터 폼 (이 예제에선 짧은 텍스트가 하나 주어졌다.)에 대해 텍스트를 인코딩한 행렬 형태의 QR코드를 생성하는 차트 서버를 호출하는 훌륭한 인터페이스를 제공한다.


여기에 완전한 프로그램이 있다. 설명을 따라가보자.

```go
package main

import (
    "flag"
    "html/template"
    "log"
    "net/http"
)

var addr = flag.String("addr", ":1718", "http service address") // Q=17, R=18

var templ = template.Must(template.New("qr").Parse(templateStr))

func main() {
    flag.Parse()
    http.Handle("/", http.HandlerFunc(QR))
    err := http.ListenAndServe(*addr, nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}

func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}

const templateStr = `
<html>
<head>
<title>QR Link Generator</title>
</head>
<body>
{{if .}}
<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
<br>
{{.}}
<br>
<br>
{{end}}
<form action="/" name=f method="GET"><input maxLength=1024 size=70
name=s value="" title="Text to QR Encode"><input type=submit
value="Show QR" name=qr>
</form>
</body>
</html>
`
```


메인 위 까지는 이해하기가 쉬울 것이다. 한 플래그는 서버의 기본 HTTP 포트를 세팅한다. 템플릿 변수인 `templ`를 사용하면서 재미가 생기기 시작한다. 이는 서버가 페이지를 보여주려고 할 때 HTML 템플릿을 제작한다. 자세한 건 잠시 후에 살펴보자.


메인 함수는 플래그를 파싱하고, 위에서 말한 메커니즘과 같이 `QR` 함수를 서버의 루트 경로에 바인딩한다. 그런 다음엔 서버를 시작하기 위해 `http.ListenAndServe`가 호출되며, 이는 서버가 실행되는 동안에는 블로킹된다.


`QR` 함수는 폼 데이터를 포함한 웹 요청만을 받으며, `s`라는 이름을 가진 폼데이터를 가지고 템플릿을 실행한다.


템플릿 패키지 `html/template`는 강력한 기능을 제공하기 때문에 이 프로그램에서 사용하기만 하면 된다. 사실, 이는 `templ.Execute`를 통해 전달된 데이터로부터 얻어지는 요소들을 대체함으로써 즉시 HTML 텍스트의 일부를 재작성한다. 이 예시에선 폼 데이터 값이 되겠다. 템플릿 텍스트 (`templateStr`)에서 이중 중괄호 부분은 템플릿 행동을 나타낸다. `{{if .}}`부터 `{{end}}`까지는 `. (dot)`라고 불리는 현재 데이터의 값이 빈 문자열이 아닐 때에만 실행된다. 만약 문자열이 비어있을 땐, 템플릿의 해당 부분은 무시될 것이다.


두 개의 `{{.}}`은 쿼리로 들어온 데이터를 템플릿에 표현하고 웹 페이지에서 보여주겠다고 말한다. HTML 템플릿 패키지는 텍스트가 안전하게 보여질 수 있도록 적절한 이스케이프를 제공한다.


템플릿 스트링의 나머지 부분들은 웹 페이지가 로드될 시 그냥 HTML로 보여진다. 설명이 부족하다면, <a href="https://golang.org/pkg/html/template/">템플릿 패키지 문서</a>를 보라.


당신은 데이터 주도 HTML 텍스트를 포함한 몇 줄 안되는 코드로 쓸만한 웹 서버를 갖게 되었다!. Go는 단 몇 줄 만으로 많은 것들을 할 수 있을 만큼 강력하다.
