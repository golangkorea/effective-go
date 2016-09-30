# 함수(Functions)

* 원문: [Functions](https://golang.org/doc/effective_go.html#functions)
* 번역자: kyungkoo

## 다중 반환 값(Multiple return values)

Go 언어가 가지고 있는 특징 중 하나는 함수와 메소드가 여러 값을 반환 할 수 있다는 것이다. 이러한 형태는 `C` 프로그램에서 대역 내 (`in-band`) 에러에서 EOF를 나타내기 위해 -1 과 같은 값을 반환하고 주소로 전달한 매개변수를 변환시키는 것과 같은 여러 골치아팠던 문법을 개선하는데 사용할 수 있다.

In C, a write error is signaled by a negative count with the error code secreted away in a volatile location. In Go, `Write` can return a count and an error: “Yes, you wrote some bytes but not all of them because you filled the device”. The signature of the Write method on files from package `os` is:

C 언어 에서는 임의의 장소에 비밀스러운 방법으로 에러 코드와 음수로 쓰기 에러를 나타낸다. Go 언어의 [Write](https://golang.org/pkg/os/#File.Write) 에서는 카운트와 에러를 반환 할 수 있다. "그래, 몇 바이트 정도는 쓰긴 했지만 장치에 가득 차기 때문에 모든 바이트를 쓰지는 못했어". `os` 패키지 파일에 있는 [Write](https://golang.org/pkg/os/#File.Write) 메소드의 시그니처는 다음과 같다.

```go
func (file *File) Write(b []byte) (n int, err error)
```

and as the documentation says, it returns the number of bytes written and a non-nil `error` when `n != len(b)`. This is a common style; see the section on error handling for more examples.

그리고 문서에서도 언급하고 있듯이, Write 메소드는 `n != len(b)`인 경우에는 쓴 바이트 숫자와 nil 이 아닌 `error` 를 반환한다. 이와 같은 형태는 지극히 일반적이며 더 많은 예제를 보고자 할 경우에는 에러 핸들링 세션을 살펴보도록 하자.

A similar approach obviates the need to pass a pointer to a return value to simulate a reference parameter. Here's a simple-minded function to grab a number from a position in a byte slice, returning the number and the next position.

유사한 방식으로 반환 값을 참조 매개변수를 흉내냄으로써 포인터를 전달 할 필요가 없게 만들 수 있다. 아래에는 숫자와 다음 위치를 반환 함으로써 바이트 슬라이스에 위치한 숫자를 가져오는 간단한 함수가 있다.

```go
func nextInt(b []byte, i int) (int, int) {
    for ; i < len(b) && !isDigit(b[i]); i++ {
    }
    x := 0
    for ; i < len(b) && isDigit(b[i]); i++ {
        x = x*10 + int(b[i]) - '0'
    }
    return x, i
}
```

You could use it to scan the numbers in an input slice b like this:

또는 다음과 같이 입력 슬라이스 b 에서 숫자를 스캔하는데도 사용할 수 있다.

```go
    for i := 0; i < len(b); {
        x, i = nextInt(b, i)
        fmt.Println(x)
    }
```

## 이름 있는 결과 인자값 (Named result parameters)

The return or result "parameters" of a Go function can be given names and used as regular variables, just like the incoming parameters. When named, they are initialized to the zero values for their types when the function begins; if the function executes a return statement with no arguments, the current values of the result parameters are used as the returned values.

Go 함수에서는 반환 "인자"나 결과 "인자"에 이름을 부여하고 인자로 들어온 매개변수 처럼 일반 변수로 사용할 수 있다. 이름을 부여하면, 해당 변수는 함수가 시작될 때 해당 타입의 제로 값으로 초기화 된다. 함수가 인자 없이 반환 문을 수행할 경우에는 결과 매개변수의 현재 값이 반환 값으로 사용된다.

The names are not mandatory but they can make code shorter and clearer: they're documentation. If we name the results of `nextInt` it becomes obvious which returned `int` is which.

이름을 부여하는것이 필수는 아니지만 이름을 부여하면 코드를 더 짧고 명확하게 만들어 주며, 문서화가 된다. `nextInt`의 결과에 이름을 부여할 경우, 반환되는 `int` 가 어떠한 것인지 명확해 진다.

```go
func nextInt(b []byte, pos int) (value, nextPos int) {
```

Because named results are initialized and tied to an unadorned return, they can simplify as well as clarify. Here's a version of `io.ReadFull` that uses them well:

이름있는 결과는 초기화 되고 아무 내용 없이 반환되기 떄문에, 명확할 뿐만 아니라 단순해 질 수 있다. 아래는 이를 이용한 `io.ReadFull` 버전이다.

```go
func ReadFull(r Reader, buf []byte) (n int, err error) {
    for len(buf) > 0 && err == nil {
        var nr int
        nr, err = r.Read(buf)
        n += nr
        buf = buf[nr:]
    }
    return
}
```

## Defer

Go's `defer` statement schedules a function call (the deferred function) to be run immediately before the function executing the `defer` returns. It's an unusual but effective way to deal with situations such as resources that must be released regardless of which path a function takes to return. The canonical examples are unlocking a mutex or closing a file.

Go 의 `defer` 문은 `defer` 를 실행하는 함수가 반환되기 전에 즉각 함수 호출(연기된 함수)을 실행하도록 예약한다. 이는 일반적인 방법은 아니긴 하지만 함수가 반환하기 위해 갖는 경로에 관계 없이 자원을 해지 해야만 하는 것과 같은 상황을 처리해야 하는 경우에는 효과적인 방법이다. 가장 대표적인 예제로는 뮤텍스(mutex)의 잠금을 풀거나 파일을 닫는 것이 있다.

```go
// Contents returns the file's contents as a string.
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```

Deferring a call to a function such as Close has two advantages. First, it guarantees that you will never forget to close the file, a mistake that's easy to make if you later edit the function to add a new return path. Second, it means that the close sits near the open, which is much clearer than placing it at the end of the function.

`Close`와 같은 함수의 호출을 지연시키면 두 가지 장점을 얻게 된다. 첫 번째로 새로운 반환 경로를 추가하기 위해 나중에 함수를 수정할 경우 하기 쉬운 실수인 파일을 닫는 것을 결코 잊지 않도록 보장 해준다. 두 번째로 `open` 근처에 `close` 가 위치하면 함수 맨 끝에 위치하는 것 보다 훨씬 명확한 코드가 되는것을 의미한다.

The arguments to the deferred function (which include the receiver if the function is a method) are evaluated when the defer executes, not when the call executes. Besides avoiding worries about variables changing values as the function executes, this means that a single deferred call site can defer multiple function executions. Here's a silly example.

지연된 함수의 매개변수 () 는 호출을 실행할 때가 아닌 `defer` 가 실행될 때 평가된다. 또한 함수가 실행될 때 변수 값이 변하는 것에 대해 걱정할 필요가 없는데, 이는 단일 지연 호출 위치는

```go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

Deferred functions are executed in LIFO order, so this code will cause 4 3 2 1 0 to be printed when the function returns. A more plausible example is a simple way to trace function execution through the program. We could write a couple of simple tracing routines like this:

지연된 함수는 LIFO 순서에 의해 실행되므로, 위 코드에서는 함수가 반환되면 4 3 2 1 0 을 출력할 것이다. 좀 더 그럴듯 한 예제로 프로그램을 통해 함수 실행을 추적하기 위한 간단한 방법이 있다. 여기서는 아래와 같이 간단한 추적 루틴을 몇가지 작생 했다.

```go
func trace(s string)   { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }

// Use them like this:
func a() {
    trace("a")
    defer untrace("a")
    // do something....
}
```

  We can do better by exploiting the fact that arguments to deferred functions are evaluated when the `defer` executes. The tracing routine can set up the argument to the untracing routine. This example:

`defer` 가 실행 될 때 지연된 함수의 매개변수가 평가된다는 사실을 이용하면 더 잘 할 수 있다. 추적 루틴은 아래와 같이 추적을 끝내는 루틴의 매개변수로 설정할 수 있다.

```go
func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}
```

prints

위 함수는 아래와 같은 결과물을 출력한다.

```sh
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```

For programmers accustomed to block-level resource management from other languages, defer may seem peculiar, but its most interesting and powerful applications come precisely from the fact that it's not block-based but function-based. In the section on `panic` and `recover` we'll see another example of its possibilities.

다른 언어로 부터 블록 레벨 자원 관리에 익숙한 프로그래머에게는 `defer` 이 생소해 보일지도 모르지만, 가장 흥미로우면서도 강력한 애플리케이션은 분명 블록 기반이 아니라 함수 기반이라는 사실로 부터 온다는 것이다. `panic` 과 `recover` 세션에서는 이러한 가능성에 대한 또 다른 예제를 살펴 볼 것이다.
