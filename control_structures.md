# 제어구조

* 원문 : [Control Structures](https://golang.org/doc/effective_go.html#control-structures)
* 번역자 : Jungsoo Ahn (@findstar)

The control structures of Go are related to those of C but differ in important ways. There is no `do` or `while` loop, only a slightly generalized `for`; `switch` is more flexible; `if` and `switch` accept an optional initialization statement like that of `for`; `break` and `continue` statements take an optional label to identify what to break or continue; and there are new control structures including a type switch and a multiway communications multiplexer, `select`. The syntax is also slightly different: there are no parentheses and the bodies must always be brace-delimited.

Go언어의 제어구조는 C 와 연관성이 있지만 중요한 점에서 차이가 있다. Go언어에서는 `do` 나 `while` 반복문이 존재하지 않으며, 단지 좀 더 일반화된 `for`, 좀 더 유연한 `switch`가  존재한다. `if` 와 `switch` 는 선택적으로 `for` 와 같은 같은 초기화 구문을 받을 수 있다. `break` 와  `continue` 구문들은 선택적으로 어느것을 멈추거나 계속할지 식별하기 위해서 라벨을 받을 수 있다. 또한 `type switch`와 `multiway communications multiplexer`, `select`의 새로운 제어 구조가 포함되어 있다. 문법은 조금 다르다. 괄호는 필요하지 않으며 body는 항상 중괄호로 구분해야 한다. 

## If

In Go a simple if looks like this:

Go언어에서 if문의 간단한 예제는 다음과 같다:

```go
if x > 0 {
    return y
}
```

Mandatory braces encourage writing simple if statements on multiple lines. It's good style to do so anyway, especially when the body contains a control statement such as a `return` or `break`.

중괄호를 의무적으로 사용해야 하기 때문에, 다중 라인에서 if 구문들이 간단하게 작성된다. 특히 body 안에 `return` 이나 `break` 와 같은 제어 구문을 포함하고 있는 경우 좋은 스타일이 되게 해준다.

Since `if` and `switch` accept an initialization statement, it's common to see one used to set up a local variable.

`if` 와 `switch` 가 초기화 구문을 받을 수 있기 때문에, 지역변수를 설정하기 위해서 이 방법을 사용하는 것을 일반적으로 볼 수 있다. 

```go
if err := file.Chmod(0664); err != nil {
    log.Print(err)
    return err
}
```

In the Go libraries, you'll find that when an if statement doesn't flow into the next statement—that is, the body ends in `break`, `continue`, `goto`, or `return`—the unnecessary `else` is omitted.

GO 라이브러리에서, if 구문이 다음 구문으로 진행되지 않을 때 -`break`, `continue`, `goto` 또는 `return` 으로 인해서 body가 으로 종료되는- 불 필요한 `else` 는 생략되는 것을 찾을 수 있다.

```go
f, err := os.Open(name)
if err != nil {
    return err
}
codeUsing(f)
```

This is an example of a common situation where code must guard against a sequence of error conditions. The code reads well if the successful flow of control runs down the page, eliminating error cases as they arise. Since error cases tend to end in `return` statements, the resulting code needs no `else` statements.

다음은 코드가 에러 상태의 시퀀스를 방지해야하는 일반적인 상황에 대한 예제이다. 만약 제어의 흐름이 성공적이라면 코드는 잘 동작할 것이고, 에러가 발생할 때마다 에러를 제거 할 것이다. 에러 케이스들은 `return` 구문에서 종료하는 경향이 있기 때문에, 결과적으로 코드에는 `else` 구문이 필요하지 않다. 

```go
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

## Redeclaration and reassignment
## 재선언과 재할당

An aside: The last example in the previous section demonstrates a detail of how the := short declaration form works. The declaration that calls `os.Open` reads,

이어서: 이전 섹션의 마지막 예제에서는 어떻게 := 짧은 선언문이 동작하는지 확인할 수 있었다. `os.Open`을 호출하는 선언코드를 보자.

```go
f, err := os.Open(name)
```

This statement declares two variables, f and err. A few lines later, the call to f.Stat reads,

이 구문은 `f` 와 `err`의 두개의 변수를 선언한다. 몇줄 아래 `f.Stat` 를 호출하는 부분을 보자.

```go
d, err := f.Stat()
```

which looks as if it declares d and err. Notice, though, that `err` appears in both statements. This duplication is legal: err is declared by the first statement, but only re-assigned in the second. This means that the call to `f.Stat` uses the existing `err` variable declared above, and just gives it a new value.

여기서 `d` 와 `err` 를 선언하는 것처럼 보인다. 주목할 부분은 저 `err`가 위에서와 아래 두 곳 모두에서 나타난다는 것이다. 이 선언의 중복은 합법적이다. `err` 은 첫번째 구문을 통해서 선언되어지만 두번째에서는 재할당된다. 이는 `f.Stat` 를 호출하는 것에서는 이미 선언되어 존재하는 `err` 변수를 사용하고, 다시 새로운 값을 부여한다는 것을 의미한다.

In `a :=` declaration a variable v may appear even if it has already been declared, provided:

어떤 `:=` 선언에서 변수 v 는 그 변수가 이미 선언되었더라도 다음의 경우, 나타날 수 있다. 

* this declaration is in the same scope as the existing declaration of v (if v is already declared in an outer scope, the declaration on will create a new variable §),
* the corresponding value in the initialization is assignable to v, and
* there is at least one other variable in the declaration that is being declared anew.

* 이 선언은 v 의 기존 선언과 같은 범위에 있는 경우( 만약 v가 바깥의 스코프에서 선언되었다면 새로운 변수를 생성한다 §참고)
* 초기화된 값과 일치하는 경우 v에 할당 가능하다
* 그리고 이것이 마지막 하나의 다른 변수 선언 안에서 

This unusual property is pure pragmatism, making it easy to use a single err value, for example, in a long if-else chain. You'll see it used often.

이 독특한 속성은 완전히 실용적이며, 예를 들어  긴 if-else 구문에서 하나의 에러 값을 손쉽게 사용하게 만들어 준다. 

§ It's worth noting here that in Go the scope of function parameters and return values is the same as the function body, even though they appear lexically outside the braces that enclose the body.

§ GO언어에서 함수 파라미터의 스코프와 리턴 값들은, 비록 그들이 사전적으로 body 에 둘어싸인 중괄호 밖에 나타나더라도 함수의 body 로써 동일하며, 이부분을 주목할 가치가 있다

## For

The Go for loop is similar to—but not the same as—C's. It unifies for and while and there is no do-while. There are three forms, only one of which has semicolons.

GO언어에서 for 반복문은 C언어와 비슷하지만 일치하지는 않는다. for 는 while 처럼 동작할 수 있고, 따라서 do-while 이 없다. 다음의 세가지 형태를 확인할 수 있으며, 하나의 경우에서만 세미콜론을 사용되는 것을 확인할 수 있다. 

```go
// C언어와 같은 경우
for init; condition; post { }

// C언어의 while 처럼 사용
for condition { }

// C언어의 for(;;) 처럼 사용
for { }
```

Short declarations make it easy to declare the index variable right in the loop.

짧은 선언문은 반복문에서 index 변수 선언을 쉽게 만든다.

```go
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}
```

If you're looping over an array, slice, string, or map, or reading from a channel, a range clause can manage the loop.

만약 배열, slice, string, map, 채널로 부터 읽어 들이는 반복문을 작성한다면, range 구문이 이 반복문을 관리해줄 수 있다.

```go
for key, value := range oldMap {
    newMap[key] = value
}
```

If you only need the first item in the range (the key or index), drop the second:

만약 range 안에서 첫번째 아이템만이 필요하다면 (키 또는 인덱스), 두번째 뒤는 날려버리자:

```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

If you only need the second item in the range (the value), use the blank identifier, an underscore, to discard the first:

만약 range 안에서 두번째 아이템만이 필요하다면 (값), 빈 식별자, 언더스코어를 사용하여 첫번째를 버리도록 하자:

```go
sum := 0
for _, value := range array {
    sum += value
}
```

The blank identifier has many uses, as described in [a later section](https://golang.org/doc/effective_go.html#blank).

공백 식별자는 많은 사용법이 있는데, [a later section](https://golang.org/doc/effective_go.html#blank)에 잘 설명되어 있다. 

For strings, the range does more work for you, breaking out individual Unicode code points by parsing the UTF-8. Erroneous encodings consume one byte and produce the replacement rune U+FFFD. (The name (with associated builtin type) rune is Go terminology for a single Unicode code point. See [the language specification](https://golang.org/ref/spec#Rune_literals) for details.) 

string 의 경우, range 는 UTF-8 파싱에 의한 개별적인 유니코드 문자를 처리하는데 유용할 것이다. 잘못된 인코딩은 하나의 바이트를 제거하고 U+FFFD 룬 문자로 대체할 것이다. ( 룬(내장된 타입으로 지정된)의 이름은 GO 언어의 단일 유니코드 코드에 대한 용어이다. 보다 자세한 사항은 [언어 스펙](https://golang.org/ref/spec#Rune_literals)을 참고하자) 

The loop

다음 반복문은 

```go
for pos, char := range "日本\x80語" { // \x80 은 합법적인 UTF-8 인코딩이다 
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
}
```

prints

다음과 같이 출력된다

```go
character U+65E5 '日' starts at byte position 0
character U+672C '本' starts at byte position 3
character U+FFFD '�' starts at byte position 6
character U+8A9E '語' starts at byte position 7
```

Finally, Go has no comma operator and ++ and -- are statements not expressions. Thus if you want to run multiple variables in a `for` you should use parallel assignment (although that precludes ++ and --).

마지막으로 Go언어는 콤마(,) 연산자가 없으며 ++, --는 표현식이 아니라 명령문이다. 따라서 만약 for문 안에서 여러개의 변수를 사용하려면 병렬 할당(parallel assignment)을 사용해야만 한다(++과 --을 배제하더라도).

```go
// Reverse a
for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
    a[i], a[j] = a[j], a[i]
}
```

## Switch

Go's switch is more general than C's. The expressions need not be constants or even integers, the cases are evaluated top to bottom until a match is found, and if the `switch` has no expression it switches on `true`. It's therefore possible—and idiomatic—to write an `if-else-if-else` chain as a `switch`.

GO언어에서 스위치는 C언어에서 보다 더 일반적인 표현이 가능하다. 표현식은 상수이거나 꼭 정수일 필요가 없고 case 구문은 위에서부터 바닥까지 해당 구문이 `true` 가 아닌 동안에 일치하는 값을 찾을 때까지 계속 값을 비교한다. 따라서 `if-else-if-else` 형태로 작성하는 것 대신 `switch` 를 사용하는 것이 가능하다-그리고 이것이 좀 더 관용적 표현이 가능하다-.

```go
func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
        return c - '0'
    case 'a' <= c && c <= 'f':
        return c - 'a' + 10
    case 'A' <= c && c <= 'F':
        return c - 'A' + 10
    }
    return 0
}
```

There is no automatic fall through, but cases can be presented in comma-separated lists.

스위치에서는 자동으로 다음으로 통과하는 동작이 없지만(케이스 구문을 지나가는 동작), 콤마로 구분된 목록을 사용해 case들을 표현할 수 있다. 

```go
func shouldEscape(c byte) bool {
    switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
        return true
    }
    return false
}
```

Although they are not nearly as common in Go as some other C-like languages, break statements can be used to terminate a switch early. Sometimes, though, it's necessary to break out of a surrounding loop, not the switch, and in Go that can be accomplished by putting a label on the loop and "breaking" to that label. This example shows both uses.

비록 GO언어의 것들이 다른 C와 같은 언어에서의 일반적인 경우와 거리가 있지만, `break` 구문은 스위치를 일찍 종료하기 위해서 사용될 수 있다. 가끔이지만, 스위치가 아닌 둘러쌓인 반복문을 중단하는 것이 필요하기도 하고, 라벨을 반복문 위에 넣고, 해당 라벨로 "탈출" 하는 것에 의해서 완료되어 질수도 있다. 다음 예제는 이 두가지 사용예이다. 

```go
Loop:
	for n := 0; n < len(src); n += size {
		switch {
		case src[n] < sizeOne:
			if validateOnly {
				break
			}
			size = 1
			update(src[n])

		case src[n] < sizeTwo:
			if n+1 >= len(src) {
				err = errShortInput
				break Loop
			}
			if validateOnly {
				break
			}
			size = 2
			update(src[n] + src[n+1]<<shift)
		}
	}
```

Of course, the `continue` statement also accepts an optional label but it applies only to loops.

물론 `continue` 구문 또한 선택적으로 라벨을 받을 수 있지만, 반복문에만 적용된다.

To close this section, here's a comparison routine for byte slices that uses two `switch` statements:

섹션을 마무리하며, 다음은 두개의 `switch` 구문을 사용하여 바이트 슬라이스를 비교하는 루틴에 대한 예제이다. 

```go
// Compare returns an integer comparing the two byte slices,
// lexicographically.
// The result will be 0 if a == b, -1 if a < b, and +1 if a > b
func Compare(a, b []byte) int {
    for i := 0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    switch {
    case len(a) > len(b):
        return 1
    case len(a) < len(b):
        return -1
    }
    return 0
}
```

## Type switch

A switch can also be used to discover the dynamic type of an interface variable. Such a type switch uses the syntax of a type assertion with the keyword type inside the parentheses. If the switch declares a variable in the expression, the variable will have the corresponding type in each clause. It's also idiomatic to reuse the name in such cases, in effect declaring a new variable with the same name but a different type in each case.

스위치 구문은 인터페이스 변수의 동적 타입을 확인하는데 사용될 수도 있다. 타입 스위치가 괄호안에서 키워드 타입을 통해서 타입 단업표현의 문법을 사용하는 것 처럼, 만약 스위치 표현식 안에서 변수를 선언한다면, 변수는 각각의 절에서 일치하는 타입을 가질 것이다. 사실상 각각의 절 안에서 새로운 변수를 다른 타입이지만 동일한 이름으로 선언하는 것과 각각의 절 안에서 이름을 재사용하는 것이 관용적이다. 

```go
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T\n", t)     // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```
