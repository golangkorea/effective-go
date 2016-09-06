# 공백 식별자 (The blank identifier)
* 원문 : [The blank identifier]()
* 번역자 : MinJae Kwon (@mingrammer)

`We've mentioned the blank identifier a couple of times now, in the context of for range loops and maps. The blank identifier can be assigned or declared with any value of any type, with the value discarded harmlessly. It's a bit like writing to the Unix /dev/null file: it represents a write-only value to be used as a place-holder where a variable is needed but the actual value is irrelevant. It has uses beyond those we've seen already.`

우리는 이미 `for range` 루프와 `maps`를 설명하면서, 공백 식별자에 대해 두 차례 언급했었다. 공백 식별자는 아무런 피해없이 그 어떤 타입의 그 어떤 값에 대해서도 할당 또는 선언될 수 있다. 이건 마치 Unix에서 `/dev/null` 파일(필요는 하지만 실제 값은 아무런 관련이 없는 변수를 저장하는 용도로서 사용되는 쓰기 전용 값)에 값을 넘기는 것과 유사하다.

## 다중 할당에서의 공백 식별자

`The use of a blank identifier in a for range loop is a special case of a general situation: multiple assignment.`

`for range`루프에서 공백 식별자의 사용은 일반적인 상황에서의 특별한 경우(다중 할당)이다.

`If an assignment requires multiple values on the left side, but one of the values will not be used by the program, a blank identifier on the left-hand-side of the assignment avoids the need to create a dummy variable and makes it clear that the value is to be discarded. For instance, when calling a function that returns a value and an error, but only the error is important, use the blank identifier to discard the irrelevant value.`

만약 좌변에  여려개의 값을 할당해야 하는데, 그 중 하나가 사용되지 않을 경우, 좌변에 공백 식별자를 두면 더미 변수를 생성 해야하는 필요가 없어지고 값 버리기를 깔끔하게 처리할 수 있다. 예를 들면, 하나의 값과 에러를 리턴하는 함수를 호출하는데 오직 에러만이 중요하다면, 무관한 값을  버리기 위해 공백 식별자를 사용한다.

```go
if _, err := os.Stat(path); os.IsNotExist(err) {
	fmt.Printf("%s does not exist\n", path)
}
```

`Occasionally you'll see code that discards the error value in order to ignore the error; this is terrible practice. Always check error returns; they're provided for a reason.`

가끔 에러를 무시하기 위해 에러값을 버리는 코드를 볼 수도 있다. 이건 끔찍한 관례이다. 에러 반환을 항상 확인하라. 에러는 원인을 제공해준다.

```go
// Bad! This code will crash if path does not exist.
fi, _ := os.Stat(path)
if fi.IsDir() {
    fmt.Printf("%s is a directory\n", path)
}
```

## 미사용 임포트와 변수

`It is an error to import a package or to declare a variable without using it. Unused imports bloat the program and slow compilation, while a variable that is initialized but not used is at least a wasted computation and perhaps indicative of a larger bug. When a program is under active development, however, unused imports and variables often arise and it can be annoying to delete them just to have the compilation proceed, only to have them be needed again later. The blank identifier provides a workaround.`

패키지를 임포트하거나 변수를 선언해놓고 쓰지않게되면 에러가 발생한다. 미사용 임포트는 프로그램의 크기를 부풀리며 컴파일 속도도 저하시킨다. 또 사용되진 않지만 초기화된 변수는 적어도 연산을 낭비하며 어쩌면 큰 버그를 암시할 수도 있다. 그러나 프로그램이 개발중에 있을때 미사용 임포트와 변수들이 종종 생겨나게 될테고, 단지 컴파일이 진행되게 하기 위해서 나중에 다시 필요해질 이들을 지우는건 귀찮을 수 있다. 공백 식별자는 이를 피하는 방법을 제공한다.

`This half-written program has two unused imports (fmt and io) and an unused variable (fd), so it will not compile, but it would be nice to see if the code so far is correct.`

아래의 반만 완성된 프로그램은 두 개의 미사용 임포트(`fmt`, `io`)와 미사용 변수(`fd`)를 가지고 있다. 따라서 이는 컴파일되지 않을 것이다. 하지만 지금까지는 코드가 정확하기에 보기 좋을 것이다.

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
}
```

`To silence complaints about the unused imports, use a blank identifier to refer to a symbol from the imported package. Similarly, assigning the unused variable fd to the blank identifier will silence the unused variable error. This version of the program does compile.`

미사용 임포트에 대한 불평을 잠재우려면, 임포트된 패키지의 상징을 참조하는 공백 식별자를 써라. 이와 유사하게, 미사용 변수 `fd`를 공백 식별자에 할당하면 미사용 변수에 대한 에러를 잠재울 수 있을 것이다. 이 버전의 프로그램은 컴파일이 된다.

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // For debugging; delete when done.
var _ io.Reader    // For debugging; delete when done.

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

`By convention, the global declarations to silence import errors should come right after the imports and be commented, both to make them easy to find and as a reminder to clean things up later.`

규약에 의하면, 임포트 에러를 잠재우기 위한 전역 선언은 임포트 구문 바로 다음에 위치되게 하며, 주석을 달아줘야 한다. 이들은 나중에 코드를 정리해야함을 쉽게 상기시키고 쉽게 찾아주게 만든다.

## 부작용(side effect)을 위한 임포트 

`An unused import like fmt or io in the previous example should eventually be used or removed: blank assignments identify code as a work in progress. But sometimes it is useful to import a package only for its side effects, without any explicit use. For example, during its init function, the net/http/pprof package registers HTTP handlers that provide debugging information. It has an exported API, but most clients need only the handler registration and access the data through a web page. To import the package only for its side effects, rename the package to the blank identifier:`

이전 예시에서 `fmt`나 `io`와 같은 미사용 임포트는 결국 사용되야 하거나 그렇지 않을 경우엔 없애야한다. (공백 할당은 진행중인 작업으로서 코드를 식별한다) 그러나 가끔은 명시적으로 사용하지는 않으면서, 단지 부작용만을 위해 임포트를 하는건 유용하다. 예를 들면,`net/http/pprof`패키지는 패키지의 초기화 함수를 실행하는 동안 디버깅 정보를 제공하는 HTTP 핸들러를 등록한다. 이는 노출된 API를 가지고 있지만 대다수의 클라어언트는 오직 핸들러 등록만이 필요하고 정보에는 웹페이지를 통해 접근한다. 부작용만을 위해 이 패키지를 임포트하기 위해선 이 패키지 이름을 공백 식별자로 바꾸면 된다.

```go
import _ "net/http/pprof"
```

`This form of import makes clear that the package is being imported for its side effects, because there is no other possible use of the package: in this file, it doesn't have a name. (If it did, and we didn't use that name, the compiler would reject the program.)`

이러한 형태의 임포트는 패키지가 부작용을 위해 임포트되고 있음을 명확하게 할 수 있다. 왜냐하면 이 파일에서는 패키지가 이름을 갖고 있지 않기 때문에 사용될 가능성이 없기 때문이다. (만약 이름을 갖고 있고 이를 사용하지 않는다면, 컴파일러는 프로그램을 거부할 것이다.)

## 인터페이스 검사

`As we saw in the discussion of interfaces above, a type need not declare explicitly that it implements an interface. Instead, a type implements the interface just by implementing the interface's methods. In practice, most interface conversions are static and therefore checked at compile time. For example, passing an *os.File to a function expecting an io.Reader will not compile unless `*os.File` implements the io.Reader interface.`

위의 인터페이스에 대한 논의에서 봤듯이, 타입은 인터페이스를 구현했음을 명시적으로 선언할 필요가 없다. 대신에, 타입은 인터페이스의 메서드를 구현함으로써 인터페이스를 구현한다. 실제로 ,대다수의 인터페이스 변환은 정적이며 따라서 컴파일 도중에 검사가 이루어진다. 예를 들면, 만약 `*os.File`이 `io.Reader`인터페이스를 구현하고 있지 않는데 이를 `io.Reader`를 기대하는 함수에 인자로 전달하게 되면 컴파일이 되지 않을 것이다. 

`Some interface checks do happen at run-time, though. One instance is in the encoding/json package, which defines a Marshaler interface. When the JSON encoder receives a value that implements that interface, the encoder invokes the value's marshaling method to convert it to JSON instead of doing the standard conversion. The encoder checks this property at run time with a type assertion like:`

하지만 몇 몇 인터페이스 검사는 런타임때 발생한다. 한 가지 예시는 `Marshaler` 인터페이스를 정의하는 `encoding/json`패키지에 있다. JSON 인코더가 저 인터페이스를 구현한 값을 받을 때, 인코더는 JSON으로 변환을 하기 위해 표준 변환을 진행하는 대신 값의 `marshiling` 메서드를 실행한다. 인코더는 런타임때 다음과 같이 타입 단언을 하면서 프로퍼티를 검사한다.

```go
m, ok := val.(json.Marshaler)
```

`If it's necessary only to ask whether a type implements an interface, without actually using the interface itself, perhaps as part of an error check, use the blank identifier to ignore the type-asserted value:`

만약 타입이 인터페이스를 구현했는지 안했는지를 실제 인터페이스 자체를 사용하지 않고, 에러 검사의 일부로서 확인할 필요가 있을 때, 타입 단언된 값을 무시하기 위해 공백 식별자를 사용하라. 

```go
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

`One place this situation arises is when it is necessary to guarantee within the package implementing the type that it actually satisfies the interface. If a type—for example, json.RawMessage—needs a custom JSON representation, it should implement json.Marshaler, but there are no static conversions that would cause the compiler to verify this automatically. If the type inadvertently fails to satisfy the interface, the JSON encoder will still work, but will not use the custom implementation. To guarantee that the implementation is correct, a global declaration using the blank identifier can be used in the package:`

이러한 상황이 나타나는 경우는 패키지가 실제로 인터페이스를 만족하는 타입을 구현하고 있는지를 보장할 필요가 있을 때이다. 만약 어떤 타입이 커스터마이징된 JSON 표기법이 필요하다면(예를 들면, `json.RawMessage`), 이는 `json.Marshaler`를 구현해야 한다. 그러나 컴파일러가 이를 자동으로 확인하도록 하는 정적 변환은 없다. 만약 부주의하게 타입이 그 인터페이스를 만족하는데에 실패를 하게되면 JSON 인코더는 여전히 실행되나 커스터마이징된 구현체를 사용할 수 없게된다. 

```go
var _ json.Marshaler = (*RawMessage)(nil)
```

`In this declaration, the assignment involving a conversion of a *RawMessage to a Marshaler requires that *RawMessage implements Marshaler, and that property will be checked at compile time. Should the json.Marshaler interface change, this package will no longer compile and we will be on notice that it needs to be updated.`

위의 선언에서 `*RawMessage`를 `Marshaler`로의 변환을 포함하고 있는 할당은 `*RawMessage`가 `Marshaler`를 구현해야함을 필요로 하며, 프로퍼티는 컴파일 도중에 검사될 것이다. `json.Marshaler`인터페이스를 변경해야만, 이 패키지가 더 이상 컴파일 되지 않을거고, 우리는 이를 업데이트 해야함을 알 수 있을 것이다.

`The appearance of the blank identifier in this construct indicates that the declaration exists only for the type checking, not to create a variable. Don't do this for every type that satisfies an interface, though. By convention, such declarations are only used when there are no static conversions already present in the code, which is a rare event.`

이 구조에서 공백 식별자가 나타나는것은 위 선언이 변수를 만드는게 아니라 단지 타입 검사를 위해서만 존재함을 알려준다. 하지만 이를 하나의 인터페이스를 만족하는 모든 타입에 사용하지는 마라. 규약에 의하면, 위와 같은 선언은 드문 경우이지만, 이미 코드에 존재하는 정적 변환이 없는 경우에만 사용하라는 것이다. 
