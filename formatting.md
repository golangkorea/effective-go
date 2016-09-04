# 포맷팅 (Formatting)
* 원문 : [Formatting](https://golang.org/doc/effective_go.html#formatting)
* 번역자 : MinJae Kwon (@mingrammer)

`Formatting issues are the most contentious but the least consequential. People can adapt to different formatting styles but it's better if they don't have to, and less time is devoted to the topic if everyone adheres to the same style. The problem is how to approach this Utopia without a long prescriptive style guide.`

포맷팅 (Formatting) 이슈는 가장 논쟁거리이긴 하지만 필수적이다. 사람들은 각자 다른 포맷팅 스타일을 적용할 수도 있지만, 모든 사람들이 같은 스타일을 고수하여 더 이상 그럴 필요가 없어지고, 포맷팅 주제에 덜 신경을 쓰게된다면 더 좋을 것이다. 문제는 길고 규정적인 스타일 가이드 없이 어떻게 이 유토피아에 접근할 수 있는 가이다.

`With Go we take an unusual approach and let the machine take care of most formatting issues. The gofmt program (also available as go fmt, which operates at the package level rather than source file level) reads a Go program and emits the source in a standard style of indentation and vertical alignment, retaining and if necessary reformatting comments. If you want to know how to handle some new layout situation, run gofmt; if the answer doesn't seem right, rearrange your program (or file a bug about gofmt), don't work around it.`

Go에서 우리는 새로운 접근법을 택하며, 머신에게 대다수의 포맷팅 이슈를 처리하도록 할 수 있다. `gofmt` 프로그램 (`go fmt`로도 사용할 수 있으며, 이는 소스 파일이 아닌 패키지 레벨에서 실행된다)은 Go 프로그램을 읽은 뒤, 표준 스타일의 들여쓰기와 수직정렬, 유지 그리고 필요시 주석을 재포맷팅한 소스를 내놓는다.

`As an example, there's no need to spend time lining up the comments on the fields of a structure. Gofmt will do that for you. Given the declaration`

예를 하나 들면, Go에서는 구조체의 필드에 적힌 주석을 정렬하는데 신경을 쓸 필요가 없다. `Gofmt`가 대신해 줄 것이다. 아래를 보자.

```go
type T struct {
    name string // name of the object
    value int // its value
}
```

`gofmt will line up the columns:`

`gofmt`는 각 열을 다음과 같이 정렬할 것이다

```go
type T struct {
    name    string // name of the object
    value   int    // its value
}
```

`All Go code in the standard packages has been formatted with gofmt.`

표준 패키지들에 있는 모든 Go 코드는 `gofmt`로 포맷팅이 되어있다.

`Some formatting details remain. Very briefly:`

몇 가지 포맷팅에 대한 상세한 내용이 남아있는데, 이를 매우 간단하게 요악해보면 다음과 같다.

`Indentation`

들여쓰기

> `We use tabs for indentation and gofmt emits them by default. Use spaces only if you must.`

> 들여쓰기를 위해 탭(tabs)을 사용하며, `gofmt`는 기본값으로 탭을 사용한다. 만약 꼭 써야하는 경우에만 스페이스(spaces)를 사용하라.

`Line length`

한 줄 길이

> `Go has no line length limit. Don't worry about overflowing a punched card. If a line feels too long, wrap it and indent with an extra tab.`

> Go는 한 줄 길이에 제한이 없다. 길이가 길어지는것에 대해 걱정하지 마라. 만약 라인 길이가 너무 길게 느껴진다면, 별도의 탭을 가지고 들여쓰기를하여 감싸라

`Parentheses`

괄호

> `Go needs fewer parentheses than C and Java: control structures (if, for, switch) do not have parentheses in their syntax. Also, the operator precedence hierarchy is shorter and clearer, so`

> Go는 C와 Java에 비해 적은 수의 괄호가 필요하다. 제어 구조들(`if`, `for`, `switch`)의 문법엔 괄호가 없다. 또한 연산자 우선순위 계층이 간단하며 명확하다. 아래를 보자.

```go
x<<8 + y<<16
```

`means what the spacing implies, unlike in the other languages.`

스페이스 간격은 다른 언어들과는 달리, 암묵적으로 어떤 의미를 가진다. 
