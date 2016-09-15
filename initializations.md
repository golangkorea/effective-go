# 초기화

* 원문 : [initialization](https://golang.org/doc/effective_go.html#initialization)
* 번역자 : Joseph Lee (@PrayForWisdom)

`Although it doesn't look superficially very different from initialization in C or C++, initialization in Go is more powerful. Complex structures can be built during initialization and the ordering issues among initialized objects, even among different packages, are handled correctly.`

표면적으로 C나 C++의 초기화와 많이 다르지 않지만, Go의 초기화는 좀 더 강력하다. 초기화되는 동안 복잡한 구조체들을 생성할 수 있고, 초기화된 객체???의 -심지어 서로 다른 패키지들의 객체까지도- 정렬???배치??? 이슈가 올바르게 동작한다.

## 상수(Constants)

`Constants in Go are just that—constant. They are created at compile time, even when defined as locals in functions, and can only be numbers, characters (runes), strings or booleans. Because of the compile-time restriction, the expressions that define them must be constant expressions, evaluatable by the compiler. For instance, 1<<3 is a constant expression, while math.Sin(math.Pi/4) is not because the function call to math.Sin needs to happen at run time.`

Go에서 상수는 우리가 일반적으로 생각하는 상수와 동일하다. 상수는 -함수 내의 지역변수로 정의된 상수조차도- 컴파일할 때 생성되며, 숫자, 문자(rune형), 문자열, 불리언(?) 중의 하나가 되어야 한다. 컴파일 시점이라는 제약 때문에, 상수를 정의하는 표현식은 컴파일러에서 평가(???) 가능한 상수 표현식(???)이어야 한다. 예를 들어,  `1<<3`은 상수 표현식이다. 반면에 `math.Sin(math.Pi/4)`는 math 패키지의 Sin 함수에 대한 호출이 런타임 시에만 가능하므로 상수 표현식이 아니다.

`In Go, enumerated constants are created using the iota enumerator. Since iota can be part of an expression and expressions can be implicitly repeated, it is easy to build intricate sets of values.`

Go에서는 열거형(자동 증가 상수형)이 iota라는 열거자를 이용해서 생성된다. iota는 표현식의 일부로 묵시적으로 반복될 수 있으므로 복잡한 값 집합들을 만들기 쉽게 해준다.

```go
type ByteSize float64

const (
    _           = iota // 공백 식별자를 이용해서 값인 0을 무시
    KB ByteSize = 1 << (10 * iota)
    MB
    GB
    TB
    PB
    EB
    ZB
    YB
)
```

`The ability to attach a method such as String to any user-defined type makes it possible for arbitrary values to format themselves automatically for printing. Although you'll see it most often applied to structs, this technique is also useful for scalar types such as floating-point types like ByteSize.`

Go는 String과 같은 메서드를 유저가 정의한 타입에 붙일 수 있다. 이것은 출력을 위해 마음대로 형식을 바꿀 수 있는도록 한다. 메서드는 일반적으로 구조체에 결함함에도 불구하고, 이 테크닉은 ByteSize와 같은 실수 형태의 스칼라 타입에서 또한 유용하다.

```go
func (b ByteSize) String() string {
    switch {
    case b >= YB:
        return fmt.Sprintf("%.2fYB", b/YB)
    case b >= ZB:
        return fmt.Sprintf("%.2fZB", b/ZB)
    case b >= EB:
        return fmt.Sprintf("%.2fEB", b/EB)
    case b >= PB:
        return fmt.Sprintf("%.2fPB", b/PB)
    case b >= TB:
        return fmt.Sprintf("%.2fTB", b/TB)
    case b >= GB:
        return fmt.Sprintf("%.2fGB", b/GB)
    case b >= MB:
        return fmt.Sprintf("%.2fMB", b/MB)
    case b >= KB:
        return fmt.Sprintf("%.2fKB", b/KB)
    }
    return fmt.Sprintf("%.2fB", b)
}
```

`The expression YB prints as 1.00YB, while ByteSize(1e13) prints as 9.09TB.`
ByteSize(1e13)은 9.09TB로 출력되는 반면에 YB는 1.00YB로 출력된다.

`The use here of Sprintf to implement ByteSize's String method is safe (avoids recurring indefinitely) not because of a conversion but because it calls Sprintf with %f, which is not a string format: Sprintf will only call the String method when it wants a string, and %f wants a floating-point value.`

여기에서 ByteSize의 String 메서드에 적용한 Sprintf의 사용은 반복적으로 재귀호출되는 것을 피했기 때문에 안전하다. 형변환 때문이 아니라, string 형태가 아닌 Sprintf를 %f 옵션으로 호출했기 때문이다. Sprintf는 단지 문자열이 필요할 때만 String 메서드를 호출할 것이고, %f는 실수형을 가져온???

## 변수(Variables)

`Variables can be initialized just like constants but the initializer can be a general expression computed at run time.`

변수의 초기화는 상수와 같은 방식이지만, 초기화는 런타임에 계산되는 일반적인 표현식이 될 수도 있다.

```go
var (
    home   = os.Getenv("HOME")
    user   = os.Getenv("USER")
    gopath = os.Getenv("GOPATH")
)
```

## init 함수(The init function)

`Finally, each source file can define its own niladic init function to set up whatever state is required. (Actually each file can have multiple init functions.) And finally means finally: init is called after all the variable declarations in the package have evaluated their initializers, and those are evaluated only after all the imported packages have been initialized.`

최종적으로, 각 소스파일은 필요한 어떤 상태든지 셋업하기 위해서 각자 init 함수를 정의할 수 있다. (init 함수는 매개변수를 가지지 않으며, 각 파일은 여러 개의 init 함수를 가질 수 있다.)여기서 "최종적으로" 라는 말은 정말로 마지막을 가리킨다: init 함수는 모든 임포트된 패키지들이 초기화되고 패키지 내의 모든 변수 선언이 평가된 이후에 호출된다.

`Besides initializations that cannot be expressed as declarations, a common use of init functions is to verify or repair correctness of the program state before real execution begins.`

초기화는 선언 형태로 표현될 수 없지만, init 함수는 실제 프로그램의 실행이 일어나기 전에 프로그램의 상태를 검증하고 올바르게 복구하는데 자주 사용된다.

```go
func init() {
    if user == "" {
        log.Fatal("$USER not set")
    }
    if home == "" {
        home = "/home/" + user
    }
    if gopath == "" {
        gopath = home + "/go"
    }
    // gopath may be overridden by --gopath flag on command line.
    flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
}
```
