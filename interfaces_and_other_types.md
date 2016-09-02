# 인터페이스와 다른 타입들

## 인터페이스

Go언어의 인터페이스는 객체의 행위(behavior)를 지정해 주는 하나의 방법입니다: 만약 어떤 객체가 정해진 행위를 할 수 있다면 호환되는 타입으로 쓸 수 있다는 뜻이죠. 이미 간단한 몇몇 예제들을 본 적인 있습니다; String 메쏘드를 구현하면 개체의 사용자 정의 출력이 가능하고, Fprintf의 출력으로 Write 메쏘드를 가지고 있는 어떤 객체라도 쓸 수 있습니다. Go 코드에서는 한 두개의 메쏘드를 지정해 주는 인터페이스가 보편적이며, 인터페이스의 이름(명사)은 보통 메쏘드(동사)에서 파생됩니다: Write 메쏘드를 구현하면 io.Writer가 인터페이스의 이름이 되는 경우.

타입은 복수의 인터페이스를 구현할 수 있습니다. [sort.Interface](https://godoc.org/sort#Interface)를 구현하고 있는 collection의 예를 들어 봅시다. [sort.Interface](https://godoc.org/sort#Interface)는 Len(), Less(i, j int) bool, 그리고 Swap(i, j int)를 지정하고 있고 이런 인테페이스를 구현한다면 sort 패키지내 [IsSorted](https://godoc.org/sort#IsSorted), [Sort](https://godoc.org/sort#Sort), [Stable](https://godoc.org/sort#Stable) 같은 루틴을 사용할 수 있습니다. 또한 사용자 지정의 포멧터를 구현할 수도 있습니다. 다음의 예제에서 Sequence는 이러한 두개의 인터페이스를 충족시키고 있습니다.

```go
type Sequence []int

// sort.Interface를 위한 필수적인 메쏘드들.
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// 프린팅에 필요한 메쏘드 - 프린트하기 전에 요소들을 정렬함.
func (s Sequence) String() string {
    sort.Sort(s)
    str := "["
    for i, elem := range s {
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

## 타입 변환(Conversions)

Sequence의 String 메쏘드는 Sprint가 벌써 슬라이스(slices)를 가지고 하는 일을 반복하고 있습니다. 하지만 만약에 Sprint를 실행하기 전에 Sequence를 []int로 변환하면 일을 줄일 수 있습니다.

```go
func (s Sequence) String() string {
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

이 같은 방법은 String 메쏘드에서 Sprint를 안전하게 실행할 수 있는 타입 변환 기법의 또 다른 예 입니다. 이것이 가능한 이유는 Sequence와 []int 두 타입이 이름만 무시하면 동일하기 때문에 합법적으로 서로 변환할 수 있는 겁니다. 이러한 타입 변환은 새로운 값을 만들어 내지 않고 현재 값에 새로운 타입이 있는 것 처럼 임시로 행동하게 합니다. (새로운 값을 만드는 다른 합법적 변환도 있습니다. 예를 들면 integer에서 floating point로의 변환)

Go 프로그램에서 일군의 다른 메쏘드를 사용하기 위해 타입을 변환하는 것은 관용적인 표현입니다. 예를 들면, [sort.IntSlice](https://godoc.org/sort#IntSlice)를 사용해 위의 프로그램 전체를 다음과 같이 간소화 시킬 수 있습니다.

```go
type Sequence []int

// 프린팅에 필요한 메쏘드 - 프린트하기 전에 요소들을 정렬함.
func (s Sequence) String() string {
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

이제, Sequence가 복수의(정렬과 출력) 인터페이스를 구현하는 대신, 하나의 데이터 아이템이 복수의 타입(Sequence, [sort.IntSlice](https://godoc.org/sort#IntSlice), 그리고 []int)으로 변환될 수 있는 점을 이용하고 있습니다. 각 타입은 주어진 작업의 일정 부분을 감당하게 됩니다. 실전에서 자주 쓰이진 않지만 매우 효과적일 수 있습니다.

## 인터페이스 변환과 타입 단언(type assertions)

타입 스위치는 변환의 한 형태입니다: 인터페이스를 받았을 때, switch문의 각 case에 맞게 타입 변환을 해 줍니다. 아래 예제는 [fmt.Printf](https://godoc.org/fmt#Printf)가 타입 스위치를 써서 어떻게 주어진 값을 string으로 변환시키는 지를 단순화된 버전으로 보여 주고 있습니다. 만약에 값이 이미 string인 경우는 인터페이스가 잡고 있는 실제 string 값을 원하고, 그렇지 않고 값이 String 메쏘드를 가지고 있을 경우는 메쏘드를 실행한 결과를 원합니다.

```go
type Stringer interface {
    String() string
}

var value interface{} // caller가 제공한 값.
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

첫번째 case는 구체적인 값을 찾은 경우이고; 두번째 case는 인터페이스를 또 다른 인터페이스로 변환한 경우입니다. 이런 식으로 여러 타입을 섞어서 써도 아무 문제가 없습니다.

오로지 한 타입만에만 관심이 있는 경우는 어떨까요? 만약 주어진 값이 string을 저장하는 걸 알고 있고 그냥 그 string 값을 추출하고자 한다면? 단 하나의 case만을 갖는 타입 스위치면 해결 할 수 있지만 타입 단언 표현을 쓸 수도 있습니다. 타입 단언은 인테페이스 값을 가지고 지정된 명확한 타입의 값을 추출합니다. 문법은 타입 스위치를 열 때와 비슷하지만 type 키워드 대신 명확한 타입을 사용합니다.

```go
value.(typeName)
```

그리고 그 결과로 얻는 새 값은 typeName이라는 정적 타입입니다. 그 타입은 인터페이스가 잡고 있는 구체적인 타입이던지 아니면 그 값이 변환 될 수 있는 2번째 인터페이스 타입이어야 합니다. 주어진 값안에 있는 string을 추출하기 위해서, 다음과 같이 쓸 수 있습니다.

```go
str := value.(string)
```

하지만 그 값이 string을 가지고 있지 않을 경우, 프로그램은 런타임 에러를 내고 죽습니다. 이런 참사에 대비하기 위해서, "comma, ok" 관용구를 사용하여 안전하게 값이 string인지 검사하십시요:

```go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

만약 타입 단언이 검사에서 실패할 경우, str는 여전히 string 타입으로 존재하고, string의 영점 값인 빈 문자열을 가지게 됩니다.

가능한 예를 또 들자면, 위에서 보여준 타입 스위치와 동일한 기능을 하는 if-else문이 여기 있습니다.

```go
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

## 일반성

만약 어떤 타입이 인터페이스를 구현하기 위해서만 존재한다면, 즉 인터페이스외 어떤 메쏘드도 외부에 노츨시키지 않은 경우, 타입 자체를 노출 시킬 필요가 없습니다. 인터페이스의 노출만으로 주어진 값이 인테페이스에 묘사된 행위들 외 어떤 흥미로운 기능도 있지 않다는 것을 확실하게 전달 합니다. 이는 또한 공통된 메쏘드에 대한 문서화의 반복을 피할 수 있습니다.

그런 경우에, constructor는 구현 타입보다는 인터페이스 값을 리턴해야 합니다. 예를 들어, 해쉬 라이브러리인 [crc32.NewIEEE](https://godoc.org/hash/crc32#NewIEEE) 와 [adler32.New](https://godoc.org/hash/adler32#New)는 둘 다 인터페이스 타입 [hash.Hash32](https://godoc.org/hash/Hash32)를 리턴합니다. Go 프로그램에서 CRC-32 알로리즘을 Adler-32로 교체하는데 요구되는 사항은 단순히 constructor 콜을 바꿔주는 것입니다; 그 외 코드들은 알고리즘의 변화에 아무런 영향을 받지 않습니다.

이와 유사한 접근을 통하면, 각종 crypto 패키지내의 스트리밍 cipher 알고리즘들을, 이들이 연결해 쓰는 block cipher들로 부터 분리시킬 수 있도록 해 줍니다. crypto/cipher 패키지내 Block 인터페이스는 한 block의 데이터를 암호화하는 block cipher의 행위를 정의 합니다. 그런 후, bufio 패키지에서 유추해 볼 수 있듯이, Block 인터페이스를 구현하는 cipher 패키지들은, Stream 인터페이스로 대표되는 스트리밍 cipher들을 건설하는데 사용될 수 있습니다.   


A similar approach allows the streaming cipher algorithms in the various crypto packages to be separated from the block ciphers they chain together. The Block interface in the crypto/cipher package specifies the behavior of a block cipher, which provides encryption of a single block of data. Then, by analogy with the bufio package, cipher packages that implement this interface can be used to construct streaming ciphers, represented by the Stream interface, without knowing the details of the block encryption.

The crypto/cipher interfaces look like this:

```go
type Block interface {
    BlockSize() int
    Encrypt(src, dst []byte)
    Decrypt(src, dst []byte)
}

type Stream interface {
    XORKeyStream(dst, src []byte)
}
```

Here's the definition of the counter mode (CTR) stream, which turns a block cipher into a streaming cipher; notice that the block cipher's details are abstracted away:

```go
// NewCTR returns a Stream that encrypts/decrypts using the given Block in
// counter mode. The length of iv must be the same as the Block's block size.
func NewCTR(block Block, iv []byte) Stream
```

NewCTR applies not just to one specific encryption algorithm and data source but to any implementation of the Block interface and any Stream. Because they return interface values, replacing CTR encryption with other encryption modes is a localized change. The constructor calls must be edited, but because the surrounding code must treat the result only as a Stream, it won't notice the difference.

## Interfaces and methods

Since almost anything can have methods attached, almost anything can satisfy an interface. One illustrative example is in the http package, which defines the Handler interface. Any object that implements Handler can serve HTTP requests.

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

ResponseWriter is itself an interface that provides access to the methods needed to return the response to the client. Those methods include the standard Write method, so an http.ResponseWriter can be used wherever an io.Writer can be used. Request is a struct containing a parsed representation of the request from the client.

For brevity, let's ignore POSTs and assume HTTP requests are always GETs; that simplification does not affect the way the handlers are set up. Here's a trivial but complete implementation of a handler to count the number of times the page is visited.

```go
// Simple counter server.
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```

(Keeping with our theme, note how Fprintf can print to an http.ResponseWriter.) For reference, here's how to attach such a server to a node on the URL tree.

```go
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

But why make Counter a struct? An integer is all that's needed. (The receiver needs to be a pointer so the increment is visible to the caller.)

```go
// Simpler counter server.
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

What if your program has some internal state that needs to be notified that a page has been visited? Tie a channel to the web page.

```go
// A channel that sends a notification on each visit.
// (Probably want the channel to be buffered.)
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```

Finally, let's say we wanted to present on /args the arguments used when invoking the server binary. It's easy to write a function to print the arguments.

```go
func ArgServer() {
    fmt.Println(os.Args)
}
```

How do we turn that into an HTTP server? We could make ArgServer a method of some type whose value we ignore, but there's a cleaner way. Since we can define a method for any type except pointers and interfaces, we can write a method for a function. The http package contains this code:

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers.  If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler object that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(c, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```

HandlerFunc is a type with a method, ServeHTTP, so values of that type can serve HTTP requests. Look at the implementation of the method: the receiver is a function, f, and the method calls f. That may seem odd but it's not that different from, say, the receiver being a channel and the method sending on the channel.

To make ArgServer into an HTTP server, we first modify it to have the right signature.

```go
// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```

ArgServer now has same signature as HandlerFunc, so it can be converted to that type to access its methods, just as we converted Sequence to IntSlice to access IntSlice.Sort. The code to set it up is concise:

```go
http.Handle("/args", http.HandlerFunc(ArgServer))
```

When someone visits the page /args, the handler installed at that page has value ArgServer and type HandlerFunc. The HTTP server will invoke the method ServeHTTP of that type, with ArgServer as the receiver, which will in turn call ArgServer (via the invocation f(c, req) inside HandlerFunc.ServeHTTP). The arguments will then be displayed.

In this section we have made an HTTP server from a struct, an integer, a channel, and a function, all because interfaces are just sets of methods, which can be defined for (almost) any type.
