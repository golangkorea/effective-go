# 함수(Functions)

* 원문: [Functions](https://golang.org/doc/effective_go.html#functions)
* 번역자: kyungkoo

## 다중 반환 값(Multiple return values)

Go 언어가 가지고 있는 특징 중 하나는 함수와 메소드가 여러 값을 반환 할 수 있다는 것이다. 이러한 형태는 `C` 프로그램에서 대역 내 (`in-band`) 에러에서 EOF를 나타내기 위해 -1 과 같은 값을 반환하고 주소로 전달한 매개변수를 변환시키는 것과 같은 여러 골치아팠던 문법을 개선하는데 사용할 수 있다.


C 언어 에서는 임의의 장소에 비밀스러운 방법으로 에러 코드와 음수로 쓰기 에러를 나타낸다. Go 언어의 [Write](https://golang.org/pkg/os/#File.Write) 에서는 카운트와 에러를 반환 할 수 있다. "그래, 몇 바이트 정도는 쓰긴 했지만 장치에 가득 차기 때문에 모든 바이트를 쓰지는 못했어". `os` 패키지 파일에 있는 [Write](https://golang.org/pkg/os/#File.Write) 메소드의 시그니처는 다음과 같다.

```go
func (file *File) Write(b []byte) (n int, err error)
```

그리고 문서에서도 언급하고 있듯이, Write 메소드는 `n != len(b)`인 경우에는 쓰인 바이트 갯수와 nil 이 아닌 `error` 를 반환한다. 이와 같은 형태는 지극히 일반적이며 더 많은 예제를 보고자 할 경우에는 에러 핸들링 세션을 살펴보도록 하자.

유사하게 반환 값으로 참조 매개변수 흉내를 냄으로써 포인터를 전달 할 필요가 없게 만들 수 있다. 아래는 숫자와 다음 위치를 반환 함으로써 바이트 슬라이스에 위치한 숫자를 가져오는 간단한 함수이다.

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

또는 다음과 같이 입력 슬라이스 b 에서 숫자를 스캔하는데도 사용할 수 있다.

```go
    for i := 0; i < len(b); {
        x, i = nextInt(b, i)
        fmt.Println(x)
    }
```

## 이름 있는 결과 인자값 (Named result parameters)


Go 함수에서는 반환 "인자"나 결과 "인자"에 이름을 부여하고 인자로 들어온 매개변수처럼 일반 변수로 사용할 수 있다. 이름을 부여하면, 해당 변수는 함수가 시작될 때 해당 타입의 제로 값으로 초기화 된다. 함수가 인자 없이 반환문을 수행할 경우에는 결과 매개변수의 현재 값이 반환 값으로 사용된다.

이름을 부여하는것이 필수는 아니지만 이름을 부여하면 코드를 더 짧고 명확하게 만들어 주며, 문서화가 된다. `nextInt`의 결과에 이름을 부여할 경우, 반환되는 `int` 가 어떠한 것인지 명확해 진다.

```go
func nextInt(b []byte, pos int) (value, nextPos int) {
```

이름있는 결과는 초기화 되고 아무 내용 없이 반환되기 때문에, 명확할 뿐만 아니라 단순해 질 수 있다. 아래는 이를 이용한 `io.ReadFull` 버전이다.

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

Go 의 `defer` 문은 `defer` 를 실행하는 함수가 반환되기 전에 즉각 함수 호출(연기된 함수)을 실행하도록 예약한다. 이는 일반적인 방법은 아니긴 하지만 함수가 어떤 실행경로를 통해 반환을 하던간에 자원을 해지 해야만 하는 것과 같은 상황을 처리해야 하는 경우에는 효과적인 방법이다. 가장 대표적인 예제로는 뮤텍스(mutex)의 잠금을 풀거나 파일을 닫는 것이 있다.

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


`Close`와 같은 함수의 호출을 지연시키면 두 가지 장점을 얻게 된다. 첫번째로 파일을 닫는 것을 잊어버리는 실수를 하지 않도록 보장해 준다. 함수에 새로운 반환 경로를 추가해야 하는 경우에 흔히 발생하는 실수이다. 두 번째로 `open` 근처에 `close` 가 위치하면 함수 맨 끝에 위치하는 것 보다 훨씬 명확한 코드가 되는것을 의미한다.


defer 함수의 매개 변수들(함수가 메서드일 경우는 리시버도 포함되는)은 함수의 호출이 실행될 때가 아닌 defer가 실행될 때 평가된다. 또한 함수가 실행될 때 변수 값이 변하는 것에 대해 걱정할 필요가 없는데, 이는 하나의 defer 호출 위치에서 여러개의 함수 호출을 지연할 수 있음을 의미한다. 여기 다소 유치한 예가 있다.

```go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

지연된 함수는 LIFO 순서에 의해 실행되므로, 위 코드에서는 함수가 반환되면 4 3 2 1 0 을 출력할 것이다. 좀 더 그럴듯 한 예제로 프로그램을 통해 함수 실행을 추적하기 위한 간단한 방법이 있다. 여기서는 아래와 같이 간단한 추적 루틴을 몇가지 작생 했다:

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


`defer` 가 실행 될 때 지연된 함수의 매개변수가 평가된다는 사실을 이용하면 더 잘 할 수 있다. 추적 루틴은 아래와 같이 추적을 끝내는 루틴의 매개변수로 설정할 수 있다:

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


다른 언어로부터 블록 레벨 자원 관리에 익숙한 프로그래머에게는 `defer` 이 생소해 보일지도 모르지만, 가장 흥미로우면서도 강력한 애플리케이션은 분명 블록 기반이 아니라 함수 기반이라는 사실로 부터 온다는 것이다. `panic` 과 `recover` 세션에서는 이러한 가능성에 대한 또 다른 예제를 살펴 볼 것이다.
