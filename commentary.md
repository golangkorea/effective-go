# 주석(Commentary)

* 원문: [Commentary](https://golang.org/doc/effective_go.html#commentary)
* 번역자: Jeongbu Yoon (@coma333ryu)


Go언어는 C언어 스타일의 /\** \**/ 블럭주석과 C++스타일의 // 한줄(line) 주석을 제공한다. 한줄주석은 일반적으로 사용되고, 블럭주석은 대부분의 패키지(package)주석에 나타난다. 하지만 표현식 안이나 많은 코드를 주석처리하는데 유용하다.

프로그램 및 웹서버이기도 한 godoc은 패키지의 내용에 대한 문서를 추출하도록 Go 소스 파일을 처리한다. 최상위 선언문 이전에 끼어드는 줄바꿈 없이 주석이 나타나면 그 선언문과 함께 추출되어 해당 항목의 설명으로 제공된다. 이러한 주석의 스타일과 유형은 godoc이 만들어내는 문서의 질을 결정하게 된다.

모든 패키지(package)는 패키지 구문 이전에 블럭주석형태의 패키지 주석이 있어야 한다. 여러 파일로 구성된 패키지의 경우, 패키지 주석은 어느 파일이든 상관없이 하나의 파일에 존재하면 되고 그것이 사용된다. 패키지 주석은 패키지를 소개해야하고, 전체 패키지에 관련된 정보를 제공해야한다. 패키지 주석은 godoc 문서의 처음에 나타나게 되니 이후의 자세한 사항도 작성해야 한다.

```go
/*
Package regexp implements a simple library for regular expressions.

The syntax of the regular expressions accepted is:

    regexp:
        concatenation { '|' concatenation }
    concatenation:
        { closure }
    closure:
        term [ '*' | '+' | '?' ]
    term:
        '^'
        '$'
        '.'
        character
        '[' [ '^' ] character-ranges ']'
        '(' regexp ')'
*/
package regexp
```

만약 패키지가 단순하다면, 패키지 주석 또한 간단할 수 있다.

```
// Package path implements utility routines for
// manipulating slash-separated filename paths.
```

별로 줄을 그어 쓰는 지나친 포맷은 주석에 필요없다. 심지어 생성된 출력이 고정폭 폰트로 주어지지 않을 수도 있으므로, 스페이스나 정렬등에 의존하지 말라. 그런 것들은 gofmt과 마찬가지로 godoc이 처리한다. 주석은 해석되지 않는 일반 텍스트이다. 그래서 HTML이나 `_this_` 같은 주석은 작성된 그대로 나타날것이다. 그러므로 사용하지 않는것이 좋다. godoc이 수정하는 한 가지는 들여쓰기된 텍스트를 고정폭 폰트로 보여주는 것으로, 프로그램 코드조각 같은 것에 적합하다. [fmt package](https://golang.org/pkg/fmt/)의 패키지주석은 좋은 예이다.

상황에 따라, godoc은 주석을 재변경 하지 않을 수 있다. 그래서 확실하게 보기좋게 만들어야 한다. 정확한 철자, 구두법, 문장구조를 사용하고 긴문장을 줄여야한다.

패키지에서 최상위 선언의 바로 앞에있는 주석이 그 선언의 문서주석으로 처리된다.
패키지 내부에서 최상위 선언 바로 이전의 주석은 그 선언을 위한 문서주석이다. 프로그램에서 모든 외부로 노출되는 (대문자로 시작되는) 이름은 문서주석이 필요하다.

문서 주석은 매우 다양한 자동화된 표현들을 가능케 하는 완전한 문장으로 작성될 때 가장 효과가 좋다. 첫 문장은 선언된 이름으로 시작하는 한 줄짜리 문장으로 요약되어야 한다.

```go
// Compile parses a regular expression and returns, if successful,
// a Regexp that can be used to match against text.
func Compile(str string) (*Regexp, error) {
```

만약 모든 문서의 주석이 그 주석이 서술하는 항목의 이름으로 시작한다면, godoc의 결과는 grep문을 이용하는데 유용한 형태로 나오게 될 것이다. 만약 당신이 정규표현식의 파싱 함수를 찾고 있는데 "Compile"이라는 이름을 기억하지 못해 다음의 명령어를 실행했다고 상상해보자,

```go
$ godoc regexp | grep parse
```

만약 패키지 내부의 모든 문서주석이 "This function.."으로 시작된다면, grep명령은 당신이 원하는 결과를 보여줄 수 없을것이다. 그러나 패키지는 각각의 문서 주석을 패키지명과 함께 시작하기 때문에, 아래와 같이 당신이 찾고 있던 단어를 상기시키는 결과를 볼 수 있을 것이다.

```go
$ godoc regexp | grep parse
    Compile parses a regular expression and returns, if successful, a Regexp
    parsed. It simplifies safe initialization of global variables holding
    cannot be parsed. It simplifies safe initialization of global variables
$
```

Go언어의 선언구문은 그룹화가 가능하다. 하나의 문서주석은 관련된 상수 또는 변수의 그룹에 대해 설명할 수 있다. 하지만 이러한 주석은 선언 전체에 나타나므로 형식적일 수 있다.

```go
// Error codes returned by failures to parse an expression.
var (
    ErrInternal      = errors.New("regexp: internal error")
    ErrUnmatchedLpar = errors.New("regexp: unmatched '('")
    ErrUnmatchedRpar = errors.New("regexp: unmatched ')'")
    ...
)
```

그룹화는 항목 간의 관련성을 나타낼 수 있다. 예를 들어 아래 변수들의 그룹은 mutex에 의해 보호되고 있음을 보여준다.

```go
var (
    countLock   sync.Mutex
    inputCount  uint32
    outputCount uint32
    errorCount  uint32
)
```
