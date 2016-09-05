# 메서드
* 원문 : [Methods](https://golang.org/doc/effective_go.html#methods)
* 번역자 : MinJae Kwon (@mingrammer)

## 포인터 vs. 값

`As we saw with ByteSize, methods can be defined for any named type (except a pointer or an interface); the receiver does not have to be a struct.`

`ByteSize`에서 보듯이, 메서드는 포인터와 인터페이스를 제외한 모든 타입에 대해 정의가 가능하다. 리시버는 꼭 구조체일 필요는 없다.

`In the discussion of slices above, we wrote an Append function. We can define it as a method on slices instead. To do this, we first declare a named type to which we can bind the method, and then make the receiver for the method a value of that type.`

위의 슬라이스에 대한 논의에서 우리는 Append함수를 작성했었다. 우리는 이를 슬라이스의 메서드로서 재정의할 수 있다. 이를 위해 이 메서드를 바인딩할 타입을 하나 선언하자. 그리고 메서드에서 타입의 값을 받기위한 리시버를 만들자.

```go
type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte {
    // 함수 내용은 위에서 정의된 Append함수와 정확히 동일하다.
}
```

`This still requires the method to return the updated slice. We can eliminate that clumsiness by redefining the method to take a pointer to a ByteSlice as its receiver, so the method can overwrite the caller's slice.`

이 함수는 여전히 갱신된 슬라이스를 반환해야할 필요가 있다. 우리는 이러한 번거로움을 메서드가 `ByteSlice`에 대한 포인터를 리시버로서 받을 수 있게 재정의함으로써 없앨 수 있고, 이 메서드는 메서드를 호출한 슬라이스를 덮어쓸 수 있다.

```go
func (p *ByteSlice) Append(data []byte) {
    slice := *p
    // 함수 내용은 위와 같지만, return이 없다.
    *p = slice
}
```

`In fact, we can do even better. If we modify our function so it looks like a standard Write method, like this,`

사실, 이보다 더 나은 함수를 작성할 수도 있다. 위의 함수를 다음과 같이 표준 `Write`메서드 처럼 작성한다면

```go
func (p *ByteSlice) Write(data []byte) (n int, err error) {
    slice := *p
    // 내용은 위와 같다.
    *p = slice
    return len(data), nil
}
```

`then the type `*ByteSlice` satisfies the standard interface io.Writer, which is handy. For instance, we can print into one.`

타입 `*ByteSlice`는 표준 인터페이스 `io.Writer`를 따르게되며, 다루기가 편해진다. 예를 들면, 다음처럼 `ByteSlice`에 값을 넣을 수 있다.

```go
var b ByteSlice
fmt.Fprintf(&b, "This hour has %d days\n", 7)
```

`We pass the address of a ByteSlice because only *ByteSlice satisfies io.Writer. The rule about pointers vs. values for receivers is that value methods can be invoked on pointers and values, but pointer methods can only be invoked on pointers.`

`*ByteSlice`는 `io.Writer`를 따르기 때문에 단순히 `ByteSlice`의 주소값만 넘겼다. 리시버를 위한 포인터 vs. 값에 대한 규칙은 값을 사용하는 메서드는 포인터와 값 위에서 모두 사용할 수 있으며, 포인터 메서드의 경우 포인터 위에서만 사용이 가능하다는 것이다.

`This rule arises because pointer methods can modify the receiver; invoking them on a value would cause the method to receive a copy of the value, so any modifications would be discarded. The language therefore disallows this mistake. There is a handy exception, though. When the value is addressable, the language takes care of the common case of invoking a pointer method on a value by inserting the address operator automatically. In our example, the variable b is addressable, so we can call its Write method with just b.Write. The compiler will rewrite that to (&b).Write for us.`

이러한 규칙은 포인터 메서드는 리시버를 변형시킬 수 있는데 메서드를 값 위에서 호출하게 되면 값의 복사본을 받기 때문에 원래값을 변형할 수 없기 때문에 생겨났다. Go언어는 이러한 실수(값 위에서 포인터 메서드를 실행하는 일)를 허용하지 않는다. 하지만 다루기 쉬운 예외가 있다. 값의 주소로 접근이 가능할 때, Go언어는 포인터 메서드를 값 위에서 실행할 경우 자동으로 주소 연산을 넣어준다. 위의 예시에서, 변수 `b`는 주소로 접근이 가능하기 때문에 단순히 `b.Write`만으로 `Write`메서드를 호출할 수 있다. 컴파일러는 이를 `(&b).Write`로 재작성할 것이다.  

`By the way, the idea of using Write on a slice of bytes is central to the implementation of bytes.Buffer.`

부연적으로, 바이트 슬라이스에 `Write`를 사용하는 아이디어는 `bytes.Buffer` 구현의 핵심이기도 하다.
