# 에러
* 원문 : [Errors](https://golang.org/doc/effective_go.html#errors)
* 번역자 : MinJae Kwon (@mingrammer)

`Library routines must often return some sort of error indication to the caller. As mentioned earlier, Go's multivalue return makes it easy to return a detailed error description alongside the normal return value. It is good style to use this feature to provide detailed error information. For example, as we'll see, os.Open doesn't just return a nil pointer on failure, it also returns an error value that describes what went wrong.`

라이브러리 루틴은 반드시 에러를 발생시킨 부분을 가리키며 에러들을 리턴해야한다. 이전에 언급되었듯이, Go의 다중값 리턴은 일반적인 리턴값에 비해 상세한 에러 내용을 제공하기 쉽게 만들어준다. 상세한 에러 내용을 제공하기 위해 이러한 특징을 활용하는 것은 좋은 방식이다. 예를 들면, 앞으로 보게될 `os.Open`은 실패할 경우 단순히 `nil`만을 리턴하지 않으며, 무엇이 잘못되었는지에 대한 에러 내용까지 리턴을 한다. 

`By convention, errors have type error, a simple built-in interface.`

규약에 의하면 에러는 간단한 내장 인터페이스의 에러 타입을 가진다

```go
type error interface {
    Error() string
}
```

`A library writer is free to implement this interface with a richer model under the covers, making it possible not only to see the error but also to provide some context. As mentioned, alongside the usual *os.File return value, os.Open also returns an error value. If the file is opened successfully, the error will be nil, but when there is a problem, it will hold an os.PathError:`

라이브러리 개발자는 에러를 보여줄뿐만 아니라 컨텍스트까지 제공해야하는 복잡한 모델을 가진 이러한 인터페이스 구현하는데에 있어 자유롭다. 언급된 바와 같이, 보통 `os.Open`는 `*os.File` 값을 리턴하는 동시에 에러값도 리턴한다. 만약 파일이 성공적으로 오픈되면, 에러는 `nil`이 될 것이고, 오픈 도중 문제가 발생할경우 `os.PathError`의 값을 가지게 될 것이다.

```go
// PathError는 에러와 연산, 문제를 발생시킨 파일 경로를 가지고 있다.
type PathError struct {
    Op string    // "open", "unlink", 등.
    Path string  // 관련 파일.
    Err error    // 시스템 콜에 의해 리턴됨.
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

`PathError's Error generates a string like this:`

`PathError`의 `Error`메서드는 다음과 같은 문자열을 생성한다.

```go
open /etc/passwx: no such file or directory
```

`Such an error, which includes the problematic file name, the operation, and the operating system error it triggered, is useful even if printed far from the call that caused it; it is much more informative than the plain "no such file or directory".`

위의 에러는 문제가 발생한 파일명과 연산, 그리고 문제를 발생시킨 운영체제 에러를 가지고 있으며, 이는 문제를 발생시킨 시스템 콜로부터 멀리 떨어진 곳에서 보여준다해도 유용하다. 이는 단순히 `no such file or directory`를 보여주는것보다 훨씬 많은 정보를 제공한다.

`When feasible, error strings should identify their origin, such as by having a prefix naming the operation or package that generated the error. For example, in package image, the string representation for a decoding error due to an unknown format is "image: unknown format".`

가능한 경우, 에러 문자열은 에러가 발생된 명령이나 패키지를 접두어로 쓰는등, 에러의 근원지를 파악할 수 있어야한다. 예를 들면, `image`패키지에서 알 수 없는 포맷으로 인해 발생한 디코딩 에러를 위한 문자열 표현은 `image: unknwon format.`이다. 

`Callers that care about the precise error details can use a type switch or a type assertion to look for specific errors and extract details. For PathErrors this might include examining the internal Err field for recoverable failures.`

정확한 상세 에러내용에 관심이 있는 사람은 자세한 그리고 추가적인 정보를 얻기 위해 타입 스위칭이나 타입 단언을 사용할 수 있다.  `PathErrors`의 경우 오류를 복구하기 위해 내부 `Err`필드를 조사할 수 있다.

```go
for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles()  // Recover some space.
        continue
    }
    return
}
```

`The second if statement here is another type assertion. If it fails, ok will be false, and e will be nil. If it succeeds, ok will be true, which means the error was of type *os.PathError, and then so is e, which we can examine for more information about the error.`

두 번째 if문은 또 다른 타입 단언이다. 만약 `os.Create`가 실패할 경우, `ok`는 `false`가 되고, 에러에 대해서 더 많은 정보를 조사할 수 있는 `e`는 `nil`이 될 것이다.

## 패닉 (Panic)

`The usual way to report an error to a caller is to return an error as an extra return value. The canonical Read method is a well-known instance; it returns a byte count and an error. But what if the error is unrecoverable? Sometimes the program simply cannot continue.`

호출자에게 에러를 알려주는 일반적인 방법은 에러를 부가적인 값으로서 리턴하는 것이다. 규범적인 `Read` 메서드는 잘 알려진 인스턴스이다. 이는 바이트 수와 에러를 리턴한다. 그러나 복구할 수 없는 에러라면 어쩔 것인가? 때때로 프로그램은 더이상 진행이 안될 것이다.

`For this purpose, there is a built-in function panic that in effect creates a run-time error that will stop the program (but see the next section). The function takes a single argument of arbitrary type—often a string—to be printed as the program dies. It's also a way to indicate that something impossible has happened, such as exiting an infinite loop.`

이런 목적을 위해 Go에는 런타임 에러를 일으켜 프로그램을 중단시키는 `panic`이라는 내장함수가 있다 (그러나 다음 장을 보라). 이 함수는 프로그램이 종료되었을 때 출력될 임의의 타입 하나를 인자로 받는다 (대개 `string`이다). 이는 또한 무한 루프가 발생하는것과 같이 불가능한 일이 일어나고 있는 무언가를 알리는 방법이기도 하다.

```go
// A toy implementation of cube root using Newton's method.
func CubeRoot(x float64) float64 {
    z := x/3   // Arbitrary initial value
    for i := 0; i < 1e6; i++ {
        prevz := z
        z -= (z*z*z-x) / (3*z*z)
        if veryClose(z, prevz) {
            return z
        }
    }
    // 백만회의 루프를 돌려도 수렴하지 않는다. 무언가 잘못되었다.
    panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
}
```

`This is only an example but real library functions should avoid panic. If the problem can be masked or worked around, it's always better to let things continue to run rather than taking down the whole program. One possible counterexample is during initialization: if the library truly cannot set itself up, it might be reasonable to panic, so to speak.`

이것은 예시일 뿐이다. 실제 라이브러리 함수는 `panic`을 피해야한다. 만약 문제가 숨어있거나 주변에서 문제를 발생시킨다해도, 실행을 계속 진행하는것이 전체 프로그램을 종료하는 것보다 낫다. 한 가지 가능한 반례는, 초기화 작업중에 있는데, 만약 라이브러리가 초기 셋업을 못하게 된다면, 이는 패닉을 발생시키고 알려야 할 것이다.

```go
var user = os.Getenv("USER")

func init() {
    if user == "" {
        panic("no value for $USER")
    }
}
```

## 복구 (Recover)

`When panic is called, including implicitly for run-time errors such as indexing a slice out of bounds or failing a type assertion, it immediately stops execution of the current function and begins unwinding the stack of the goroutine, running any deferred functions along the way. If that unwinding reaches the top of the goroutine's stack, the program dies. However, it is possible to use the built-in function recover to regain control of the goroutine and resume normal execution.`

슬라이스의 범위를 넘어선 인덱싱이나 타입 단언 실패등의 런타임 에러를 포함한 패닉이 발생하였을 때, 이는 즉시 현재 함수의 실행을 중단시키며 모든 지연된 함수들과 고루틴 스택을 풀기 시작한다. 만약 풀기 작업이 고루틴 스택의 최정상에 도달했을 때, 프로그램은 종료된다. 그러나 내장함수인 `recover`를 사용하면 고루틴의 통제권을 다시 얻을 수 있으며, 명령어 실행을 정상적으로 진행할 수 있게 된다.

`A call to recover stops the unwinding and returns the argument passed to panic. Because the only code that runs while unwinding is inside deferred functions, recover is only useful inside deferred functions.`

`recover`를 호출하면 풀기 작업이 중지되며, `panic`에 전달된 인자값이 리턴된다. 풀기 작업을 하는 동안에 실행되는 코드는 오직 지연된 함수안에서만 실행되기 때문에 , `recover`는 오직 지연된 함수 내에서만 유용하다.

`One application of recover is to shut down a failing goroutine inside a server without killing the other executing goroutines.`

`recover`의 한 응용으로는 서버 내의 실패한 고루틴을 다른 실행중인 고루틴들을 종료시키지 않고 종료시키는 것이다.

```go
func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```

`In this example, if do(work) panics, the result will be logged and the goroutine will exit cleanly without disturbing the others. There's no need to do anything else in the deferred closure; calling recover handles the condition completely.`

이 예시에서, 만약 `do(work)`에서 패닉이 발생하면, 그 결과가 로그로 남겨질 것이고 고루틴은 다른 고루틴들을 방해하지 않으면서 깔끔하게 종료될 것이다. 호출된 `recover`가 이 상황을 완전히 처리할 것이기 때문에 지연된 클로져에서는 그 어떤 것도 할 필요가 없다.

`Because recover always returns nil unless called directly from a deferred function, deferred code can call library routines that themselves use panic and recover without failing. As an example, the deferred function in safelyDo might call a logging function before calling recover, and that logging code would run unaffected by the panicking state.`

`recover`는 지연된 함수로부터 직접 호출되는 경우를 제외하면 항상 `nil`을 리턴하기 때문에, 지연된 코드는 `panic`과 `recover`를 사용하는 라이브러리 루틴을 실패없이 호출할 수 있다. 한 가지 예를 들어보면, `safelyDo`내의 지연된 함수는 `recover`를 호출하기 전에 logging 함수를 호출할 것이다. 그리고 logging 코드는 패닉 상태에 영향을 받지 않으면서 잘 실행될 것이다. 

`With our recovery pattern in place, the do function (and anything it calls) can get out of any bad situation cleanly by calling panic. We can use that idea to simplify error handling in complex software. Let's look at an idealized version of a regexp package, which reports parsing errors by calling panic with a local error type. Here's the definition of Error, an error method, and the Compile function.`

위에서의 복구 패턴을 보면, `do`함수 (그리고 호출을 하는 그 어느것도)는 `panic`을 호출함으로써 안좋은 상황을 피해갈 수 있다. 이 아이디어를 활용하면 복잡한 소프트웨어에서의 에러 핸들링을 단순화 할 수 있다. `regexp`패키지의 가장 이상적인 버전을 보자. 이는 자체 에러 타입과 함께 `panic`을 호출함으로써 파싱 에러를 알린다. 아래에 `Error`와 에러 메서드, 그리고 `Compile`함수의 정의가 있다.

```go
// Error is the type of a parse error; it satisfies the error interface.
type Error string
func (e Error) Error() string {
    return string(e)
}

// error is a method of *Regexp that reports parsing errors by
// panicking with an Error.
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

// Compile returns a parsed representation of the regular expression.
func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse will panic if there is a parse error.
    defer func() {
        if e := recover(); e != nil {
            regexp = nil    // Clear return value.
            err = e.(Error) // Will re-panic if not a parse error.
        }
    }()
    return regexp.doParse(str), nil
}
```

`If doParse panics, the recovery block will set the return value to nil—deferred functions can modify named return values. It will then check, in the assignment to err, that the problem was a parse error by asserting that it has the local type Error. If it does not, the type assertion will fail, causing a run-time error that continues the stack unwinding as though nothing had interrupted it. This check means that if something unexpected happens, such as an index out of bounds, the code will fail even though we are using panic and recover to handle parse errors.`

만약 `doParse`가 패닉을 발생시키면, 복구 블록은 리턴 값을 nil로 설정할 것이다. 지연된 함수는 이름있는 리턴 값을 변형할 수 있다. `err`에 값을 할당하는 과정에서  에러를 자체 에러 타입으로 단언함으로써 그 문제가 파싱 에러인지 아닌지를 검사를 할 것이다. 만약 그렇지 않다면, 타입 단언은 실패를 하게 될 것이고, 아무 인터럽트가 나지 않았음에도 불구하고 스택 풀기작업을 진행하며 런타임 에러를 일으킬 것이다. 이 검사는 인덱스가 범위를 벗어나는 등의 의도치않은 일이 생길 때 파싱 에러를 처리하기위해 `panic`과 `recover`를 사용했음에도 불구하고 코드가 실패함을 의미한다. 

`With error handling in place, the error method (because it's a method bound to a type, it's fine, even natural, for it to have the same name as the builtin error type) makes it easy to report parse errors without worrying about unwinding the parse stack by hand:`

아래에서의 에러 처리를 보면, 에러 메서드는 (타입에 바인딩하는 메서드이기 때문에 내장 에러 타입과 동일한 이름을 사용하는것은 자연스러우며 괜찮다.) 직접 파싱 스택을 푸는것에 대한 걱정없이 파싱 에러를 알리기 쉽게 한다.

```go
if pos == 0 {
    re.error("'*' illegal at start of expression")
}
```

`Useful though this pattern is, it should be used only within a package. Parse turns its internal panic calls into error values; it does not expose panics to its client. That is a good rule to follow.`

이 패턴은 유용함에도 불구하고, 오직 패키지 안에서만 사용되어야 한다. `Parse`는 내부 패닉 호출을 에러 값으로 바꾸며, 이는 클라이언트에게 패닉을 노출시키지 않는다. 이는 본받을만한 좋은 규칙이다.

`By the way, this re-panic idiom changes the panic value if an actual error occurs. However, both the original and new failures will be presented in the crash report, so the root cause of the problem will still be visible. Thus this simple re-panic approach is usually sufficient—it's a crash after all—but if you want to display only the original value, you can write a little more code to filter unexpected problems and re-panic with the original error. That's left as an exercise for the reader.`

부연적으로, `re-panic` 구문은 실제 에러가 발생했을 때 패닉 값을 변경한다. 그러나 원래의 실패와 새로운 실패가 오류 리포트에서 보여질 것이고, 그래서 루트는 여전히 문제 발생을 보여지게 할 것이다. 따라서 이러한 간단한 `re-panic`접근법은 보통 모든게 다 끝난 후 오류가 발생할 경우에 충분하며, 그러나 오직 원래의 값을 보여주고 싶을 땐, 의도치않은 문제를 필터링하는 코드를 작성할 수 있으며 원래의 에러를 가지고 `re-panic`을 할 수 있다. 이는 독자들에게 숙제로 남기겠다. 
