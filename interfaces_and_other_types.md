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

## Interface conversions and type assertions

Type switches are a form of conversion: they take an interface and, for each case in the switch, in a sense convert it to the type of that case. Here's a simplified version of how the code under fmt.Printf turns a value into a string using a type switch. If it's already a string, we want the actual string value held by the interface, while if it has a String method we want the result of calling the method.

```go
type Stringer interface {
    String() string
}

var value interface{} // Value provided by caller.
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

The first case finds a concrete value; the second converts the interface into another interface. It's perfectly fine to mix types this way.

What if there's only one type we care about? If we know the value holds a string and we just want to extract it? A one-case type switch would do, but so would a type assertion. A type assertion takes an interface value and extracts from it a value of the specified explicit type. The syntax borrows from the clause opening a type switch, but with an explicit type rather than the type keyword:

```go
value.(typeName)
```

and the result is a new value with the static type typeName. That type must either be the concrete type held by the interface, or a second interface type that the value can be converted to. To extract the string we know is in the value, we could write:

```go
str := value.(string)
```

But if it turns out that the value does not contain a string, the program will crash with a run-time error. To guard against that, use the "comma, ok" idiom to test, safely, whether the value is a string:

```go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```

If the type assertion fails, str will still exist and be of type string, but it will have the zero value, an empty string.

As an illustration of the capability, here's an if-else statement that's equivalent to the type switch that opened this section.

```go
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

## Generality

If a type exists only to implement an interface and will never have exported methods beyond that interface, there is no need to export the type itself. Exporting just the interface makes it clear the value has no interesting behavior beyond what is described in the interface. It also avoids the need to repeat the documentation on every instance of a common method.

In such cases, the constructor should return an interface value rather than the implementing type. As an example, in the hash libraries both crc32.NewIEEE and adler32.New return the interface type hash.Hash32. Substituting the CRC-32 algorithm for Adler-32 in a Go program requires only changing the constructor call; the rest of the code is unaffected by the change of algorithm.

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
