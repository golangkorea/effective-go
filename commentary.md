# 주석(Commentary)

* 원문: [Commentary](https://golang.org/doc/effective_go.html#commentary)
* 번역자: Jeongbu Yoon (@coma333ryu)

Go provides C-style /* */ block comments and C++-style // line comments. Line comments are the norm; block comments appear mostly as package comments, but are useful within an expression or to disable large swaths of code.

Go언어는 C언어 스타일의 "/* */" 블럭주석과 C++스타일의 "//" 한줄(line) 주석을 제공한다. 한줄주석은 일반적이며 블럭주석은 대부분의 패키지(package)주석에 나타난다. 하지만 표현식안이나 많은 코드를 주석처리하는데 유용하다.

The program—and web server—godoc processes Go source files to extract documentation about the contents of the package. Comments that appear before top-level declarations, with no intervening newlines, are extracted along with the declaration to serve as explanatory text for the item. The nature and style of these comments determines the quality of the documentation godoc produces.

웹 서버로 구동되는 프로그램 godoc([The program—and web server—godoc](https://godoc.org/golang.org/x/tools/cmd/godoc))은 패키지의 내용에 대한 문서를 추출하도록 Go 소스 파일을 처리한다. 최상위 선언문 이전에 줄바꿈없이 나타나는 주석은 현재 소스에 대한 설명을 제공하기 위하여 최상위 선언문과 함께 발췌된다. 이러한 주석의 스타일과 유형은 godoc이 만들어내는 문서의 질을 결정하게 된다.

Every package should have a package comment, a block comment preceding the package clause. For multi-file packages, the package comment only needs to be present in one file, and any one will do. The package comment should introduce the package and provide information relevant to the package as a whole. It will appear first on the godoc page and should set up the detailed documentation that follows.

모든 패키지(package)는 패키지(package)구문 이전에 블럭주석형태의 패키지 주석이 있어야 한다. 여러파일의 패키지들의 경우, 패키지 주석은 어느 파일이든 상관없이 하나의 파일에 존재하면 된다. 패키지 주석은 패키지를 소개해야하고, 전체 패키지에 관련된 정보를 제공해야한다. 패키지 주석은 godoc문서의 처음에 나타나게 될것이고, 반듯이 작성되어야할 참고사항은 아래와 같다.

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

If the package is simple, the package comment can be brief.
만약 패키지가 단순하다면, 패키지주석은 간단히 작성할 수 있다.

```
// Package path implements utility routines for
// manipulating slash-separated filename paths.
```

Comments do not need extra formatting such as banners of stars. The generated output may not even be presented in a fixed-width font, so don't depend on spacing for alignment—godoc, like `gofmt`, takes care of that. The comments are uninterpreted plain text, so HTML and other annotations such as _this_ will reproduce verbatim and should not be used. One adjustment godoc does do is to display indented text in a fixed-width font, suitable for program snippets. The package comment for the [fmt package](https://golang.org/pkg/fmt/) uses this to good effect.

주석은 추가적인 형식은 필요하지 않는다. 생성된 결과는 고정폭의 폰트(font)로 표시되지 않을 수 있다. 그래서 godoc정렬(alignment—godoc)을 위해 "gofmt"처럼 스페이스(spacing)에 의존하지 마라. 주석은 실행되지 않는 일반적인글이다. 그래서 HTML이나 "_this_"같은 다른 형태의 주석은 작성된 그대로 나타날것이고 그래서 사용하면 안된다. godoc이 수정하는 한 가지는 들여쓰기된 텍스트를 고정폭 폰트로 보여주는 것으로, 프로그램 코드조각 같은 것에 적합하다.
[fmt package](https://golang.org/pkg/fmt/)의 패키지주석은 좋은 예이다.

Depending on the context, godoc might not even reformat comments, so make sure they look good straight up: use correct spelling, punctuation, and sentence structure, fold long lines, and so on.

상황에 따라, godoc은 주석을 변경하지 못할 수도 있다. 그래서 확실하게 보기좋게 만들어야 한다. 정확한 철자, 구두법, 문장구조를 사용하고 긴문장을 줄여야한다.

Inside a package, any comment immediately preceding a top-level declaration serves as a doc comment for that declaration. Every exported (capitalized) name in a program should have a doc comment.

패키지 내부 최상위 선언 바로 이전의 어떤 주석은 

Doc comments work best as complete sentences, which allow a wide variety of automated presentations. The first sentence should be a one-sentence summary that starts with the name being declared.

```go
// Compile parses a regular expression and returns, if successful,
// a Regexp that can be used to match against text.
func Compile(str string) (*Regexp, error) {
```

If the name always begins the comment, the output of godoc can usefully be run through grep. Imagine you couldn't remember the name "Compile" but were looking for the parsing function for regular expressions, so you ran the command,

```go
$ godoc regexp | grep parse
```

If all the doc comments in the package began, "This function...", grep wouldn't help you remember the name. But because the package starts each doc comment with the name, you'd see something like this, which recalls the word you're looking for.

```go
$ godoc regexp | grep parse
    Compile parses a regular expression and returns, if successful, a Regexp
    parsed. It simplifies safe initialization of global variables holding
    cannot be parsed. It simplifies safe initialization of global variables
$
```

Go's declaration syntax allows grouping of declarations. A single doc comment can introduce a group of related constants or variables. Since the whole declaration is presented, such a comment can often be perfunctory.

```go
// Error codes returned by failures to parse an expression.
var (
    ErrInternal      = errors.New("regexp: internal error")
    ErrUnmatchedLpar = errors.New("regexp: unmatched '('")
    ErrUnmatchedRpar = errors.New("regexp: unmatched ')'")
    ...
)
```

Grouping can also indicate relationships between items, such as the fact that a set of variables is protected by a mutex.

```go
var (
    countLock   sync.Mutex
    inputCount  uint32
    outputCount uint32
    errorCount  uint32
)
```
