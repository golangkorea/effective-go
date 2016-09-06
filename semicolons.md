# 세미콜론(Semicolons)

* 원문: [Semicolons](https://golang.org/doc/effective_go.html#semicolons)
* 번역자: Jeongbu Yoon (@coma333ryu)

Like C, Go's formal grammar uses semicolons to terminate statements, but unlike in C, those semicolons do not appear in the source. Instead the lexer uses a simple rule to insert semicolons automatically as it scans, so the input text is mostly free of them.

C언어 처럼, Go의 정식문법은 구문을 종료하기 위하여 세미콜론을 사용한다. 하지만 C언어와는 달리, 세미콜론은 소스상에 나타나지 않는다. 대신 구문분석기(lexer)는 스캔하는것처럼 자동으로 세미콜론을 추가하기 위해 단순한 규칙을 사용한다. 그래서 소스작성시 대부분 세미콜론을 사용하지 않는다.

The rule is this. If the last token before a newline is an identifier (which includes words like int and float64), a basic literal such as a number or string constant, or one of the tokens

규칙은 다음과 같다. 만약 새로운 라인앞의 마지막 토큰이 숫자나 문자상수 혹은 토큰같은 기본 문자의 식별자(int와 float64같은 단어들을 포함한)라면, 구문분석기(lexer)는 항상 토큰 다음에 세미콜론을 추가한다.

```go
break continue fallthrough return ++ -- ) }
```

the lexer always inserts a semicolon after the token. This could be summarized as, “if the newline comes after a token that could end a statement, insert a semicolon”.

이것은 "만약 새로운 라인이 토큰뒤에 온다면, 구문이 끝났고, 세미콜론을 추가해라." 와 같이 요약해서 설명할 수 있다.

A semicolon can also be omitted immediately before a closing brace, so a statement such as

세미콜론은 또한 닫는 중괄호(}) 바로 앞에서 생략할 수 있다. 예를들어 아래의 구문은 세미콜론이 필요하지 않는다.

    go func() { for { dst <- <-src } }()

needs no semicolons. Idiomatic Go programs have semicolons only in places such as for loop clauses, to separate the initializer, condition, and continuation elements. They are also necessary to separate multiple statements on a line, should you write code that way.

Go 프로그램에서는 세미콜론을 for loop 구문에서 변수 초기화와 조건, 그리고 진행 변수를 구분할때에만 사용 한다. 또한 세미콜론은 한 라인에서 여러문장을 구분하기 위해 필요하고, 이런 방법으로 코드를 작성해야한다.

One consequence of the semicolon insertion rules is that you cannot put the opening brace of a control structure (if, for, switch, or select) on the next line. If you do, a semicolon will be inserted before the brace, which could cause unwanted effects. Write them like this

세미콜론 입력규칙의 중요한 한가지는 제어문(if, for, switch, 혹은 select)의 여는 중괄호({)를 다음 라인에 사용하지 말아야 한다는 것이다. 만약 그렇게 사용하게 되면, 세미콜론은 중괄호({) 앞에 추가될것이고, 예상하지 못한 영향을 발생기킬 것이다. 다음과 같이 작성하라.

```go
if i < f() {
    g()
}
```

not like this

다음과 같이 사용하지 마라.

```go
if i < f()  // wrong!
{           // wrong!
    g()
}
```