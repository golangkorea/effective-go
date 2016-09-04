# Introduction

Go is a new language. Although it borrows ideas from existing languages, it has unusual properties that make effective Go programs different in character from programs written in its relatives. A straightforward translation of a C++ or Java program into Go is unlikely to produce a satisfactory result—Java programs are written in Java, not Go. On the other hand, thinking about the problem from a Go perspective could produce a successful but quite different program. In other words, to write Go well, it's important to understand its properties and idioms. It's also important to know the established conventions for programming in Go, such as naming, formatting, program construction, and so on, so that programs you write will be easy for other Go programmers to understand.

Go는 새로운 언어입니다. 비록 기존 언어의 아이디어들을 차용했지만, 독특한 속성을 지니고 있으며 이는 동일한 속성들로 쓰여진 프로그램들과는 성격이 다른 것으로 Go 프로그램을 효과적으로 만들어줍니다. C++ 또는 Java 프로그램을 바로 Go로 변환하면 만족스러운 결과를 만들어내기는 어려울 것입니다. 왜냐하면 Java 프로그램은 Go가 아닌 Java로 쓰여졌기 때문입니다. 반면에, Go 관점에서 문제를 생각하면 만족스러운 결과를 만들 수 있습니다. 다시 말하면, Go를 잘 작성하기 위해서는 Go의 속성들과 Go 언어다운 코드들을 이해하는 것이 중요합니다. 또한 다른 Go 프로그래머들이 여러분들이 작성한 Go 프로그래밍을 쉽게 이해할 수 있도록 네이밍, 포맷팅, 프로그램 구조 등과 같은 Go 프로그래밍을 위한 정해진 컨벤션들을 아는 것도 중요합니다.

This document gives tips for writing clear, idiomatic Go code. It augments the [language specification](https://golang.org/ref/spec), the [Tour of Go](https://tour.golang.org), and How to [Write Go Code](https://golang.org/doc/code.html), all of which you should read first.

이 문서는 명확하고 Go언어다운 Go코드를 작성하는 팁을 전해 드리며 [언어 명세](https://golang.org/ref/spec), [Go 살펴보기](https://tour.golang.org), 그리고 [Go 코드 작성하는 방법](https://golang.org/doc/code.html) 여러분들이 먼저 읽어야 모든 지식을 향상시킬 것입니다.

## Examples

The [Go package sources](https://golang.org/src/) are intended to serve not only as the core library but also as examples of how to use the language. Moreover, many of the packages contain working, self-contained executable examples you can run directly from the [golang.org](https://golang.org/) web site, such as [this one](https://golang.org/pkg/strings/#example_Map) (if necessary, click on the word "Example" to open it up). If you have a question about how to approach a problem or how something might be implemented, the documentation, code and examples in the library can provide answers, ideas and background.

[Go 패키지 소스](https://golang.org/src/)는 코어 라이브러리로써 뿐만 아니라 언어를 사용하는 예제로써도 제공될 수 있도록 만들어졌습니다. 더군다나, [example Map](https://golang.org/pkg/strings/#exmaple_Map) 처럼 많은 패키지들이 동작하는, 즉 독립적으로 실행가능한 예제들을 포함하고 있으며 [golang.org](https://golang.org/) 웹사이트에서 돌려볼 수 있습니다. (필요할 경우, "예제"라는 단어를 클릭해서 열어보세요) 문제에 어떻게 접근해야하는지 혹은 어떻게 구현되어 있는지 궁금하다면 라이브러리의 문서, 코드 그리고 예제들이 해결책이나 아이디어, 그리고 배경지식을 알려줄 수 있을 것입니다.
