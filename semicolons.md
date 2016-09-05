# 세미콜론(Semicolons)

Like C, Go's formal grammar uses semicolons to terminate statements, but unlike in C, those semicolons do not appear in the source. Instead the lexer uses a simple rule to insert semicolons automatically as it scans, so the input text is mostly free of them.

C언어 처럼, Go의 정식문법은 구문을 종료하기 위하여 세미콜론을 사용한다. 하지만 C언어와는 달리, 세미콜론은 소스상에 나타나지 않는다. 대신 구문분석기(lexer)는 스캔하는것처럼 자동으로 세미콜론을 추가하기 위해 단순한 규칙을 사용한다. 그래서 소스작성시 대부분 세미콜론을 사용하지 않는다.

The rule is this. If the last token before a newline is an identifier (which includes words like int and float64), a basic literal such as a number or string constant, or one of the tokens

규칙은 다음과 같다. 만약 신규라인 이전의 마지막 토큰이 식별자(int와 float64같은 단어들을 포함한 숫자나 문자상수 혹은 토큰중의 하나같은)이면,

```go
break continue fallthrough return ++ -- ) }
```

the lexer always inserts a semicolon after the token. This could be summarized as, “if the newline comes after a token that could end a statement, insert a semicolon”.

구문분석기(lexer)는 항상 토큰뒤에 세미콜론을 추가한다.

A semicolon can also be omitted immediately before a closing brace, so a statement such as

    go func() { for { dst <- <-src } }()
needs no semicolons. Idiomatic Go programs have semicolons only in places such as for loop clauses, to separate the initializer, condition, and continuation elements. They are also necessary to separate multiple statements on a line, should you write code that way.

One consequence of the semicolon insertion rules is that you cannot put the opening brace of a control structure (if, for, switch, or select) on the next line. If you do, a semicolon will be inserted before the brace, which could cause unwanted effects. Write them like this

```go
if i < f() {
    g()
}
```

not like this

```go
if i < f()  // wrong!
{           // wrong!
    g()
}
```