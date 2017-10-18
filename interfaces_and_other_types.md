# 인터페이스와 다른 타입들

* 원문: [Interfaces and other types](https://golang.org/doc/effective_go.html#interfaces_and_types)
* 번역자: Jhonghee Park

## 인터페이스


Go언어의 인터페이스는 객체의 행위(behavior)를 지정해 주는 하나의 방법이다: 만약 어떤 객체가 정해진 행동를 할 수 있다면 호환되는 타입으로 쓸 수 있다는 뜻이다. 이미 간단한 몇몇 예제들을 본 적이 있다; String 메서드를 구현하면 개체의 사용자 정의 출력이 가능하고, Fprintf의 출력으로 Write 메서드를 가지고 있는 어떤 객체라도 쓸 수 있다. Go 코드에서는 한 두개의 메서드를 지정해 주는 인터페이스가 보편적이며, 인터페이스의 이름(명사)은 보통 메서드(동사)에서 파생된다: Write 메서드를 구현하면 io.Writer가 인터페이스의 이름이 되는 경우.

타입은 복수의 인터페이스를 구현할 수 있다. [sort.Interface](https://godoc.org/sort#Interface)를 구현하고 있는 collection의 예를 들어 보자. [sort.Interface](https://godoc.org/sort#Interface)는 Len(), Less(i, j int) bool, 그리고 Swap(i, j int)를 지정하고 있고 이런 인테페이스를 구현한다면 sort 패키지내 [IsSorted](https://godoc.org/sort#IsSorted), [Sort](https://godoc.org/sort#Sort), [Stable](https://godoc.org/sort#Stable) 같은 루틴을 사용할 수 있다. 또한 사용자 지정의 포멧터를 구현할 수도 있다. 다음의 예제에서 Sequence는 이러한 두개의 인터페이스를 충족시키고 있다.

```go
type Sequence []int

// sort.Interface를 위한 필수적인 메서드들.
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// 프린팅에 필요한 메서드 - 프린트하기 전에 요소들을 정렬함.
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

## 타입 변환


Sequence의 String 메서드는 Sprint가 이미 슬라이스(slices)를 가지고 하는 일을 반복하고 있다. 하지만 Sprint를 실행하기 전에 Sequence를 []int로 변환하면 일을 줄일 수 있다.

```go
func (s Sequence) String() string {
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}
```

이 같은 방법은 String 메서드에서 Sprint를 안전하게 실행할 수 있는 타입 변환 기법의 또 다른 예이다. 이것이 가능한 이유는 Sequence와 []int 두 타입이 이름만 무시하면 동일하기 때문에 합법적으로 서로 변환할 수 있는 것이다. 이러한 타입 변환은 새로운 값을 만들어 내지 않고 현재 값에 새로운 타입이 있는 것 처럼 임시로 행동하게 한다. (새로운 값을 만드는 다른 합법적 변환도 있다. 예를 들면 integer에서 floating point로의 변환)


Go 프로그램에서 일군의 다른 메서드를 사용하기 위해 타입을 변환하는 것은 Go 언어다운 표현이다. 예를 들면, [sort.IntSlice](https://godoc.org/sort#IntSlice)를 사용해 위의 프로그램 전체를 다음과 같이 간소화 시킬 수 있다.

```go
type Sequence []int

// 프린팅에 필요한 메서드 - 프린트하기 전에 요소들을 정렬함.
func (s Sequence) String() string {
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

보시라! Sequence가 복수의(정렬과 출력) 인터페이스를 구현하는 대신, 하나의 데이터 아이템이 복수의 타입(Sequence, [sort.IntSlice](https://godoc.org/sort#IntSlice), 그리고 []int)으로 변환될 수 있는 점을 이용하고 있다. 각 타입은 주어진 작업의 일정 부분을 감당하게 된다. 실전에서 자주 쓰이진 않지만 매우 효과적일 수 있다.

## 인터페이스 변환과 타입 단언


타입 스위치는 변환의 한 형태이다: 인터페이스를 받았을 때, switch문의 각 case에 맞게 타입 변환을 한다. 아래 예제는 [fmt.Printf](https://godoc.org/fmt#Printf)가 타입 스위치를 써서 어떻게 주어진 값을 string으로 변환시키는 지를 단순화된 버전으로 보여 주고 있다. 만약에 값이 이미 string인 경우는 인터페이스가 잡고 있는 실제 string 값을 원하고, 그렇지 않고 값이 String 메서드를 가지고 있을 경우는 메서드를 실행한 결과를 원한다.

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


첫번째 case는 구체적인 값을 찾은 경우이고; 두번째 case는 인터페이스를 또 다른 인터페이스로 변환한 경우이다. 이런 식으로 여러 타입을 섞어서 써도 아무 문제가 없다.

오로지 한 타입만에만 관심이 있는 경우는 어떨까? 만약 주어진 값이 string을 저장하는 걸 알고 있고 그냥 그 string 값을 추출하고자 한다면? 단 하나의 case만을 갖는 타입 스위치면 해결 할 수 있지만 타입 단언 표현을 쓸 수도 있다. 타입 단언은 인테페이스 값을 가지고 지정된 명확한 타입의 값을 추출한다. 문법은 타입 스위치를 열 때와 비슷하지만 type 키워드 대신 명확한 타입을 사용한다:

```go
value.(typeName)
```


그리고 그 결과로 얻는 새 값은 typeName이라는 정적 타입이다. 그 타입은 인터페이스가 잡고 있는 구체적인 타입이던지 아니면 그 값이 변환 될 수 있는 2번째 인터페이스 타입이어야 한다. 주어진 값 안에 있는 string을 추출하기 위해서, 다음과 같이 쓸 수 있다:

```go
str := value.(string)
```


하지만 그 값이 string을 가지고 있지 않을 경우, 프로그램은 런타임 에러를 내고 죽는다. 이런 참사에 대비하기 위해서, "comma, ok" 관용구를 사용하여 안전하게 값이 string인지 검사 해야 한다:

```go
str, ok := value.(string)
if ok {
    fmt.Printf("string value is: %q\n", str)
} else {
    fmt.Printf("value is not a string\n")
}
```


만약 타입 단언이 검사에서 실패할 경우, str는 여전히 string 타입으로 존재하고, string의 제로값인 빈 문자열을 가지게 된다.


가능한 예를 또 들자면, 위에서 보여준 타입 스위치와 동일한 기능을 하는 if-else문이 여기 있다.

```go
if str, ok := value.(string); ok {
    return str
} else if str, ok := value.(Stringer); ok {
    return str.String()
}
```

## 일반성


만약 어떤 타입이 오로지 인터페이스를 구현하기 위해서만 존재한다면, 즉 인터페이스외 어떤 메서드도 외부에 노츨시키지 않은 경우, 타입 자체를 노출 시킬 필요가 없다. 단지 인터페이스만을 노출하는 것은 주어진 값이 인테페이스에 묘사된 행위들 외 어떤 흥미로운 기능도 있지 않다는 것을 확실하게 전달한다. 이는 또한 공통된 메서드에 대한 문서화의 반복을 피할 수 있다.


그런 경우에, constructor는 구현 타입보다는 인터페이스 값을 반환해야 한다. 예를 들어, 해쉬 라이브러리인 [crc32.NewIEEE](https://godoc.org/hash/crc32#NewIEEE) 와 [adler32.New](https://godoc.org/hash/adler32#New)는 둘 다 인터페이스 타입 [hash.Hash32](https://godoc.org/hash/Hash32)를 반환한다. Go 프로그램에서 CRC-32 알로리즘을 Adler-32로 교체하는데 요구되는 사항은 단순히 constructor 콜을 바꿔주는 것이다; 그 외 코드들은 알고리즘의 변화에 아무런 영향을 받지 않는다.


이와 유사한 방식을 통해서, 각종 crypto 패키지내의 스트리밍 cipher 알고리즘들을, 이들이 연결해 쓰는 block cipher들로 부터 분리시킬 수 있다. crypto/cipher 패키지내 [Block](https://godoc.org/crypto/cipher#Block) 인터페이스는 한 block의 데이터를 암호화하는 block cipher의 행위를 정의한다. 그런 다음, bufio 패키지에서 유추해 볼 수 있듯이, [Block](https://godoc.org/crypto/cipher#Block) 인터페이스를 구현하는 cipher 패키지들은, [Stream](https://godoc.org/crypto/cipher#Stream) 인터페이스로 대표되는 스트리밍 cipher들을 건설할 때, block 암호화의 자세한 내용을 알지 못하더라도, 사용될 수 있다.   


crypto/cipher 인터페이스들은 다음과 같다:

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


여기 block cipher를 스트리밍 cipher로 바꾸어 주는 카운터 모드 (CTR) 스트림의 정의가 있다; block cipher의 자세한 내용이 추상화되어 있는 점을 유의하라:

```go
// NewCTR은 카운더 모드로 주어진 Block을 이용하여 암호화하고/해독하는 스트림을 반환한다.
// iv의 길이는 Block의 block 크기와 같아야 한다.
func NewCTR(block Block, iv []byte) Stream
```


NewCTR은 특정한 암호화 알고리즘과 데이터 소스에만 적용되는 것이 아니라 [Block](https://godoc.org/crypto/cipher#Block)와 [Stream](https://godoc.org/crypto/cipher#Stream) 인터페이스를 구현하는 어떤 알고리즘이나 데이터 소스에도 적용이 가능하다. 왜냐하면 인터페시스 값들을 반환하고, CTR 암호화를 다른 암호화 모드로 교체하는 것이 국부적인 변화이기 때문이다. constructor 콜은 반드시 편집되어야 합니다. 하지만 둘러싸고 있는 코드는 반환 결과를 [Stream](https://godoc.org/crypto/cipher#Stream)으로 처리해야 하기 때문에, 차이를 알지 못 한다.

## 인터페이스와 메서드


거의 모든 것에 메서드를 첨부할 수 있다는 말은 거의 모든 것이 인터페이스를 만족 시킬 수 있다는 말이기도 하다. 한 회화적인 예가 [http](https://godoc.org/net/http) 패키지내 정의되어 있는 [Handler](https://godoc.org/net/http#Handler) 인터페이스 이다. [Handler](https://godoc.org/net/http#Handler)를 구현하는 어떤 객체도 HTTP request에 서비스를 제공할 수 있다.

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```


[ResponseWriter](https://godoc.org/net/http#ResponseWriter) 역시 클라이어트에 응답을 반환하는데 필요한 메서드들의 접근을 제공하는 인터페이스이다. 이 메서드들은 표준 Write 메서드를 포함하여서, [http.ResponseWriter](https://godoc.org/net/http#ResponseWriter)는 [io.Writer](https://godoc.org/io#Writer)가 사용될 수 있는 곳이면 어디든 사용할 수 있다. [Request](https://godoc.org/net/http#Request)는 클라이언트로 부터 오는 request의 분석된 내용을 담은 struct이다.


간결함을 위해, POST는 무시하고 항상 HTTP request가 GET라고 가정하자; 이런 단순화는 handler들이 셑업되는 방식에 영향을 미치지 않는다. 여기 사소한 예이긴 하지만 페이지 방문 수를 세는 hander의 완전한 구현이 있다.

```go
// 단순한 카운터 서버.
type Counter struct {
    n int
}

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ctr.n++
    fmt.Fprintf(w, "counter = %d\n", ctr.n)
}
```



(지금까지 얘기와 일맥상통하는 예로 [Fprintf](https://godoc.org/fmt#Fprintf)가 [http.ResponseWriter](https://godoc.org/net/http#ResponseWriter)에 출력할 수 있음을 주목하라.) 참고로, 여기 URL에 그런 서버를 어떻게 부착하는 예가 있다.

```go
import "net/http"
...
ctr := new(Counter)
http.Handle("/counter", ctr)
```

그런데 굳이 Counter를 struct으로 만들 이유가 있을까? integer면 충분하다. (caller에게 값의 증가를 보이기 위해 리시버는 포인터일 필요가 있다.)

```go
// 단순한 카운터 서버.
type Counter int

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
}
```

만약에 여러분의 프로그램 내부 상태가 페이지를 방문을 알아야 할 경우라면 어떨까? 웹 페이지에 채널을 묶으라.

```go
// 채널이 매 방문마다 알린다.
// (아마 이 채널에는 버퍼를 사용해야 할 것이다.)
type Chan chan *http.Request

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
}
```


마지막으로, 서버를 구동할 때 사용한 명령줄 인수들을 /args에 보여주려는 경우를 상상해 보자. 명령줄 인수를 출력하는 함수를 쓰는 것은 간단하다.

```go
func ArgServer() {
    fmt.Println(os.Args)
}
```


이것을 어떻게 HTTP 서버로 바꿀 수 있을까? 어떤 타입에다가 값은 무시하면서 ArgServer를 메서드로 만들 수 있을 것이다. 하지만 더 좋은 방법이 있다. 포인터와 인터페이스만 빼고는 어떤 타입에도 메서드를 정의할 수 있는 사실을 이용해서, 함수에 메서드를 쓸 수 있다. [http](https://godoc.org/net/http) 패키지에 다음과 같은 코드가 있다:

```go
// HandlerFunc는 어뎁터로써 평범한 함수를 HTTP handler로 쓸 수 있게 해 준다.
// 만약에 f가 적절한 함수 signature를 가지면,
// HandlerFunc(f)는 f를 부르는 Handler 객체인 것이다.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, req).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
}
```


[HandlerFunc](https://godoc.org/net/http#HandlerFunc)는 [ServeHTTP](https://godoc.org/net/http#ServeHTTP)라는 매쏘드를 같는 타입으로, 이 타입의 값은 HTTP request에 서비스를 제공한다. 메서드의 구현을 한번 살펴 보라: 리시버는 함수, f이고 메서드가 f를 부른다. 이상해 보일 수도 있지만, 리시버가 채널이고 메서드가 채널에 데이터를 보내는 예와 비교해도 크게 다르지 않다.


ArgServer를 HTTP 서버로 만들기 위해서, 우선 적절한 signature를 갖도록 고쳐야 한다.

```go
// Argument server.
func ArgServer(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(w, os.Args)
}
```


ArgServer는 이제 [HandlerFunc](https://godoc.org/net/http#HandlerFunc)와 signature가 동일 하다. 마치 IntSlice.Sort 메서드를 쓰기 위해 Sequence를 [IntSlice](https://godoc.org/sort#IntSlice)로 변환 했듯이, ServeHTTP를 쓰기 위해 ArgServer를 HandlerFunc로 변환 시킬 수 있다. 셑업을 하는 코드는 매우 간결하다:

```go
http.Handle("/args", http.HandlerFunc(ArgServer))
```


누가 /args를 방문했을 때, 그 페이지에 설치된 handler는 ArgServer 값을 갖는 [HandlerFunc](https://godoc.org/net/http#HandlerFunc)타입 이다. HTTP 서버는 그 타입의 ServeHTTP 메서드를 부르면서 ArgServer를 리시버로 사용하고, 결국 ArgServer를 부르게 된다: HandlerFunc.ServeHTTP안에서 f(w, req)를 부르게 된다. 그리고 나면 명령줄 인수가 나타나 보인다.


지금까지 struct, integer, channel, 그리고 함수(function)을 가지고 HTTP 서버를 만들어 보았다. 이것이 가능한 이유는 인터페이스가 거의 모든 타입에 정의 할 수 있는 단순한 메서드의 집합이기 때문이다.
