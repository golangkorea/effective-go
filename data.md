# 데이터

* 원문: [Data](https://golang.org/doc/effective_go.html#data)
* 번역자: Jhonghee Park

## `new`를 사용하는 메모리 할당

Go에는 메모리를 할당하는 두가지 기본 방식이 있는데, 내장(built-in) 함수인 new와 make이다. 서로 다른 일을 하고 다른 타입들에 적용되기 때문에 혼란스러울 수 있지만, 규칙은 간단하다. new부터 살펴보자. 내장 함수로 메모리를 할당하지만 다른 언어에 존재하는 같은 이름의 기능과는 다르게 메모리를 초기화하지 않고, 단지 값을 제로화(zero) 한다. 다시 말하면, new(T)는 타입 T의 새로운 객체에 제로값이 저장된 공간(zeroed storage)을 할당하고 그 객체의 주소인, `*T`값을 반환한다. Go의 용어로 얘기하자면, 새로 제로값으로 할당된 타입 T를 가리키는 포인터를 반환하는 것이다.

`new`를 통해 반환된 메모리는 제로값을 갖기 때문에, 굳이 초기화 과정이 없이도 사용된 타입의 제로값을 그대로 쓸 수 있도록 데이터 구조를 설계하면 도움이 된다. 무슨 말이냐 하면 데이터 구조의 사용자가 new로 새 구조체를 만들고 바로 일에 사용할 수 있다는 것이다. 예를 들어, [bytes.Buffer](https://godoc.org/bytes#Buffer)의 문서는 "Buffer의 제로값은 바로 사용할 수 있는 텅빈 버퍼이다"라고 말하고 있다. 유사하게, [sync.Mutex](https://godoc.org/sync#Mutex)도 명시된 constructor도 Init 메서드도 가지고 있지 않다. 대신, [sync.Mutex](https://godoc.org/sync#Mutex)의 제로값인 잠기지 않은 mutex로 정의되어 있다.

제로값의 유용함은 전이적인(transitive) 특성이 있다. 다음의 타입 선언을 검토해 보자.

```go
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}
```

`SyncedBuffer` 타입의 값들은 메모리 할당이나 단순히 선언만 으로도 당장 사용할 준비가 된다. 아래 코드 단편에서는, p와 v가 뒤이은 조정이 없어도 모두 정확히 작동한다.

```go
p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```

## 생성자와 합성 리터럴

때로 제로값만으로는 충분치 않고 생성자(constructor)로 초기화해야 할 필요가 생기는데, 아래 예제는 [os](https://godoc.org/os)패키지에서 가지고 온 것이다.

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

이 예제에는 불필요하게 반복된(boiler plate) 코드들이 많다. 합성 리터럴(composite literal)로 간소화할 수 있는데, 그 표현이 실행될 때마다 새로운 인스턴스를 만들어 낸다.

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

C와는 달리, 로컬 변수의 주소를 반환해도 아무 문제가 없음을 주목하라; 변수에 연결된 저장공간은 함수가 반환해도 살아 남는다. 실제로, 합성 리터럴의 주소를 취하는 표현은 매번 실행될 때마다 새로운 인스턴스에 연결된다. 그러므로 마지막 두 줄을 묶어 버릴 수 있다.

```go
    return &File{fd, name, nil, 0}
```

합성 리터럴의 필드들은 순서대로 배열되고 반드시 입력해야 한다. 하지만, 요소들에 레이블을 붙여 필드:값 식으로 명시적으로 짝을 만들면, 초기화는 순서에 관계 없이 나타날 수 있다. 입력되지 않은 요소들은 각자에 맞는 제로값을 갖는다. 그러므로 아래와 같이 쓸 수 있다.  

```go
    return &File{fd: fd, name: name}
```


제한적인 경우로, 만약 합성 리터럴이 전혀 필드를 갖지 않을 때는, 그 타입의 제로값을 생성한다. `new(File)`은 `&File{}`과 동일한 표현이다.


또 합성 리터럴은 arrays, slices, 와 maps를 생성하는데 사용될 수도 있는데, 필드 레이블로 인덱스와 맵의 키를 적절히 사용해야 한다. 아래 예제의 경우, `Enone`, `Eio` 그리고 `Einval`의 값에 상관없이, 서로 다르기만 하면 초기화가 작동한다.

```go
a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
```

## make를 사용하는 메모리 할당

Back to allocation. The built-in function make(T, args) serves a purpose different from new(T). It creates slices, maps, and channels only, and it returns an initialized (not zeroed) value of type T (not `*T`). The reason for the distinction is that these three types represent, under the covers, references to data structures that must be initialized before use. A slice, for example, is a three-item descriptor containing a pointer to the data (inside an array), the length, and the capacity, and until those items are initialized, the slice is `nil`. For slices, maps, and channels, make initializes the internal data structure and prepares the value for use. For instance,

다시 메로리 할당으로 돌아가자. 내장 함수인 make(T, args)는 new(T)와 다른 목적으로 제공된다. slices, maps, 그리고 channels에만 사용하고 (`*T`가 아닌) 타입 T의 (제로값이 아닌) 초기화된 값을 반환한다. 이러한 차이가 있는 이유는 이 세 타입이 내부적으로 반드시 사용 전 초기화 되어야 하는 데이터 구조를 가리키고 있기 때문이다. 예를 들어, slice는 세가지 항목의 기술항으로 (array내) 데이터를 가리키는 포인터, 크기, 그리고 용량를 가지며, 이 항목들이 초기화되기 전 까지, slice는 `nil`이다. slices, maps, 그리고 channels은 내부 데이터 구조를 초기화하고 사용할 값들을 준비한다. 예를 들면,

```go
make([]int, 10, 100)
```

allocates an array of 100 ints and then creates a slice structure with length 10 and a capacity of 100 pointing at the first 10 elements of the array. (When making a slice, the capacity can be omitted; see the section on slices for more information.) In contrast, `new([]int)` returns a pointer to a newly allocated, zeroed slice structure, that is, a pointer to a `nil` slice value.

메모리에 크기가 100인 int 배열을 할당하고 그 배열의 처음 10개를 가리키는, 크기 10와 용량이 100인 slice 데이터 구조를 생성한다. (slice를 만들때 용량은 생략해도 된다. 자세한 내용은 slice 섹션을 보기 바란다.) 그에 반해, `new([]int)`는 새로 할당되고, 제로값으로 채워진 slice 구조를 가리키는 포인터를 반환하는데, 이 때 slice 값은 `nil`이다.

These examples illustrate the difference between new and make.

다음 예제들이 new와 make의 차이점을 잘 그리고 있다.

```go
var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely useful
var v  []int = make([]int, 100) // the slice v now refers to a new array of 100 ints

// Unnecessarily complex:
var p *[]int = new([]int)
*p = make([]int, 100, 100)

// Idiomatic:
v := make([]int, 100)
```

`Remember that make applies only to maps, slices and channels and does not return a pointer. To obtain an explicit pointer allocate with new or take the address of a variable explicitly.`

make는 maps, slices 그리고 channels에만 적용되며 포인터를 반환하지 않음을 기억하라. 포인터를 얻고 싶으면 new를 사용해서 메모리를 할당하거나 변수의 주소를 명시적으로 취하라.

## 배열

`Arrays are useful when planning the detailed layout of memory and sometimes can help avoid allocation, but primarily they are a building block for slices, the subject of the next section. To lay the foundation for that topic, here are a few words about arrays.`

배열은 메모리의 상세한 레이아웃을 계획하는데 유용하며 때로는 메모리 할당을 피하는데 도움이 된다. 하자만 주로 다음 섹션의 주제인, slice의 재료로 쓰인다. slice를 얘기하기 전에 배열에 대해 몇자 적음으로 해서 기초를 닦아 보자.

There are major differences between the ways arrays work in Go and C. In Go,

Go와 C에서는 배열의 작동원리에 큰 차이가 있다. Go에서는,

* Arrays are values. Assigning one array to another copies all the elements.
* In particular, if you pass an array to a function, it will receive a copy of the array, not a pointer to it.
* The size of an array is part of its type. The types [10]int and [20]int are distinct.

* 배열은 값이다. 한 배열을 다른 배열에 할당(assign)할 때 모든 요소가 복사된다.
* 특히, 함수에 배열을 패스할 때, 함수는 포인터가 아닌 복사된 배열을 받는다.
* 배열의 크기는 타입의 한 부분이다. 타입 [10]int과 [20]int는 서로 다르다.

The value property can be useful but also expensive; if you want C-like behavior and efficiency, you can pass a pointer to the array.

배열에 값(value)을 사용하는 것이 유용할 수도 있지만 또한 비용이 큰 연산이 될 수도 있다; 만약 C와 같은 실행이나 효율성을 원한다면, 아래와 같이 배열 포인터를 보낼 수도 있다.

```go
func Sum(a *[3]float64) (sum float64) {
    for _, v := range *a {
        sum += v
    }
    return
}

array := [...]float64{7.0, 8.5, 9.1}
x := Sum(&array)  // Note the explicit address-of operator
```

`But even this style isn't idiomatic Go. Use slices instead.`

하지만 이런 스타일조차 Go언어 답지는 않다. 대신 slice를 사용하라.

## Slices

Slices wrap arrays to give a more general, powerful, and convenient interface to sequences of data. Except for items with explicit dimension such as transformation matrices, most array programming in Go is done with slices rather than simple arrays.

Slice는 배열을 포장하므로써 데이터 시퀀스에 더 일반적이고, 강력하며, 편리한 인터페이스를 제공한다. 변환 메스릭스와 같이 뚜렷한 차원(dimension)을 갖고 있는 항목들을 제외하고는, Go에서 거의 모든 배열 프로그래밍은 단순한 배열보다는 slice를 사용한다.

Slices hold references to an underlying array, and if you assign one slice to another, both refer to the same array. If a function takes a slice argument, changes it makes to the elements of the slice will be visible to the caller, analogous to passing a pointer to the underlying array. A Read function can therefore accept a slice argument rather than a pointer and a count; the length within the slice sets an upper limit of how much data to read. Here is the signature of the Read method of the `File` type in package os:

Slice는 내부의 배열을 가리키는 레퍼런스를 쥐고 있어, 만약에 다른 slice에 할당(assign)되어도, 둘 다 같은 배열을 가리킨다. 함수가 slice를 받아 그 요소에 변화를 주면 호출자도 볼 수 있는데, 이것은 내부의 배열를 가리키는 포인터를 함수에 보내는 것과 유사하다. 그러므로 Read 함수는 포인터와 카운터 대신, slice를 받아 들일 수 있다; slice내 length는 데이터를 읽을 수 있는 최대 한계치에 정해져 있다. 아래에 [File](https://godoc.org/os#File)타입의 Read 메서드의 시그너쳐가 있다.

```go
func (f *File) Read(buf []byte) (n int, err error)
```

The method returns the number of bytes read and an error value, if any. To read into the first 32 bytes of a larger buffer buf, slice (here used as a verb) the buffer.

메서드는 읽은 바이트의 수와 생길 수도 있는 에러 값을 반환한다. 아래 예제에서는, 더 큰 버퍼 buf의 첫 32 바이트를 읽어 들이기 위해 그 버퍼를 슬라이스했다(여기에서는 slice를 동사로 썼다).

```go
    n, err := f.Read(buf[0:32])
```

Such slicing is common and efficient. In fact, leaving efficiency aside for the moment, the following snippet would also read the first 32 bytes of the buffer.

그런 슬라이싱은 흔히 사용되고 효율적이다. 효율성은 잠시 뒤로한 체 얘기를 하자면, 아래 예제도 역시 버퍼의 첫 32 바이트를 읽는다.

```go
    var n int
    var err error
    for i := 0; i < 32; i++ {
        nbytes, e := f.Read(buf[i:i+1])  // Read one byte.
        if nbytes == 0 || e != nil {
            err = e
            break
        }
        n += nbytes
    }
```

The length of a slice may be changed as long as it still fits within the limits of the underlying array; just assign it to a slice of itself. The capacity of a slice, accessible by the built-in function `cap`, reports the maximum length the slice may assume. Here is a function to append data to a slice. If the data exceeds the capacity, the slice is reallocated. The resulting slice is returned. The function uses the fact that `len` and `cap` are legal when applied to the `nil` slice, and return 0.

slice의 길이는 내부배열의 한계내에서는 얼마든지 바뀔 수 있다; 단순히 slice 차체에 할당하면 된다. slice의 용량은, 내장함수 `cap`을 통해 얻을 수 있는데, slice가 가질 수 있는 최대 크기를 보고한다. 여기 slice에 데이터를 부착할 수 있는 함수가 있다. 만약 데이터가 용량을 초과하면, slice의 메모리는 재할당된다. 결과물인 slice는 반환된다. 이 함수는 `len`과 `cap`이 `nil` slice에 합법적으로 적용할 수 있고, 0을 반환하는 사실을 이용하고 있다.

```go
func Append(slice, data []byte) []byte {
    l := len(slice)
    if l + len(data) > cap(slice) {  // reallocate
        // Allocate double what's needed, for future growth.
        newSlice := make([]byte, (l+len(data))*2)
        // The copy function is predeclared and works for any slice type.
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:l+len(data)]
    for i, c := range data {
        slice[l+i] = c
    }
    return slice
}
```

We must return the slice afterwards because, although Append can modify the elements of `slice`, the slice itself (the run-time data structure holding the pointer, length, and capacity) is passed by value.

slice는 꼭 처리후 반환되어야 한다. Append가 `slice`의 요소들을 변경할 수 있지만, slice 자체(포인터, 크기, 용량을 갖고 있는 런타임 데이터 구조)는 값으로 패스되었기 때문이다.

The idea of appending to a slice is so useful it's captured by the append built-in function. To understand that function's design, though, we need a little more information, so we'll return to it later.

Slice에 부착한다는 생각은 매우 유용하기 때문에 내장함수 append로 만들어져 있다. 이 함수의 설계를 이해하려면, 우선 좀 더 정보가 필요하다. 그래서 나중에 다시 얘기하기로 하겠다.

## 이차원 slices

Go's arrays and slices are one-dimensional. To create the equivalent of a 2D array or slice, it is necessary to define an array-of-arrays or slice-of-slices, like this:

Go의 배열과 slice는 일차원적이다. 이차원의 배열이나 slice와 동일한 것을 만들려면, 배열의 배열 혹은 slice의 slice을 다음과 같이 정의해야 한다.

```go
type Transform [3][3]float64  // A 3x3 array, really an array of arrays.
type LinesOfText [][]byte     // A slice of byte slices.
```

Because slices are variable-length, it is possible to have each inner slice be a different length. That can be a common situation, as in our LinesOfText example: each line has an independent length.

Slice는 크기가 변할 수 있기 때문에, 내부의 slice들은 크기가 각자 다를 수 있다. 다음의 LinesOfText 예와 같이 흔히 일어 날 수 있는 일이다: 각 줄은 독립적으로 다른 크기를 가지고 있다.

```go
text := LinesOfText{
	[]byte("Now is the time"),
	[]byte("for all good gophers"),
	[]byte("to bring some fun to the party."),
}
```

Sometimes it's necessary to allocate a 2D slice, a situation that can arise when processing scan lines of pixels, for instance. There are two ways to achieve this. One is to allocate each slice independently; the other is to allocate a single array and point the individual slices into it. Which to use depends on your application. If the slices might grow or shrink, they should be allocated independently to avoid overwriting the next line; if not, it can be more efficient to construct the object with a single allocation. For reference, here are sketches of the two methods. First, a line at a time:

때로 이차원의 slice를 메모리에 할당할 필요가 생기는데, 예를 들면 픽셀로 된 줄들은 스캔하는 상황이다. 두 가지 방식으로 해결할 수 있는데, 첫번째는 각 slice를 독립적으로 할당하는 것이고; 두번째는 slice 하나를 할당해서 각 slice를 자른 다음 가리키는 것이다. 어느 것을 쓸지는 애플리케이션에 달렸다. 만약 slice가 자라거나 줄어들 수 있다면, 독립적으로 할당해서 다음 줄을 덮어쓰는 일을 방지해야 하고; 그렇지 않다면, 객체를 생성하기 위한 메모리 할당을 한번에 하는 것이 더 효율적일 수 있다. 참고로, 여기 두 방식의 스케치가 잇다. 우선, 한번에 한줄씩:

```go
// Allocate the top-level slice.
// 최상위 레벨의 slice를 할당하라.
picture := make([][]uint8, YSize) // One row per unit of y. 유닛 y마다 한 줄씩
// Loop over the rows, allocating the slice for each row.
//
for i := range picture {
	picture[i] = make([]uint8, XSize)
}
```

And now as one allocation, sliced into lines:

그리고 이제 한번의 메모리 할당으로, 잘라서 줄을 만드는 경우:

```go
// Allocate the top-level slice, the same as before.
picture := make([][]uint8, YSize) // One row per unit of y.
// Allocate one large slice to hold all the pixels.
pixels := make([]uint8, XSize*YSize) // Has type []uint8 even though picture is [][]uint8.
// Loop over the rows, slicing each row from the front of the remaining pixels slice.
for i := range picture {
	picture[i], pixels = pixels[:XSize], pixels[XSize:]
}
```

## Maps

Maps are a convenient and powerful built-in data structure that associate values of one type (the key) with values of another type (the element or value) The key can be of any type for which the equality operator is defined, such as integers, floating point and complex numbers, strings, pointers, interfaces (as long as the dynamic type supports equality), structs and arrays. Slices cannot be used as map keys, because equality is not defined on them. Like slices, maps hold references to an underlying data structure. If you pass a map to a function that changes the contents of the map, the changes will be visible in the caller.

Map은 편리하고 강력한 내장 데이타 구조로 한 타입의 값들(the key)을 다른 타입의 값들에 연결해준다. Key는 equality연산이 정의되어 있는 어떤 타입이라도 사용 가능하며, integers, floating point, 복소수(complex numbers), strings, 포인터(pointers), 인터페이스(equality를 지원하는 동적 타입에 한해서), structs 그리고 배열(arrays)이 그러한 예이다. Slice는 map의 key로 사용될 수 없는데, 그 이유는 equality가 정의되어 있지 않기 때문이다. Slice와 마찬가지로 map 역시 내부 데이터 구조를 가진다. 함수에 map을 입력하고 map의 내용물을 변경하면, 그 변화는 호출자에게도 보인다.

Maps can be constructed using the usual composite literal syntax with colon-separated key-value pairs, so it's easy to build them during initialization.

Map 또한 콜론으로 분리된 key-value 짝을 이용한 합성 리터럴로 생성될 수 있으며, 초기화중에 쉽게 만들 수 있다.

```go
var timeZone = map[string]int{
    "UTC":  0*60*60,
    "EST": -5*60*60,
    "CST": -6*60*60,
    "MST": -7*60*60,
    "PST": -8*60*60,
}
```

Assigning and fetching map values looks syntactically just like doing the same for arrays and slices except that the index doesn't need to be an integer.

Map에 값을 할당하거나 추출하는 문법은, 인덱스가 integer일 필요가 없다는 것외에는 배열과 slice과 거의 동일하다.

```go
offset := timeZone["EST"]
```

An attempt to fetch a map value with a key that is not present in the map will return the zero value for the type of the entries in the map. For instance, if the map contains integers, looking up a non-existent key will return 0. A set can be implemented as a map with value type bool. Set the map entry to `true` to put the value in the set, and then test it by simple indexing.

Map에 없는 key를 가지고 값을 추출하려는 시도는 값의 타입에 해당하는 제로값을 반환할 것이다. 예를 들어, 만약 map이 integer를 가지고 있으면, 존재하지 않는 key에 대한 조회는 0을 반환한다. bool 타입의 값을 가진 map으로 set을 구현할 수 있다. map의 엔트리를 `true`로 저장함으로써 set안에 값을 집어 넣을 수 있고, 간단히 인덱싱을 통해 검사해 볼 수 있다.

```go
attended := map[string]bool{
    "Ann": true,
    "Joe": true,
    ...
}

if attended[person] { // will be false if person is not in the map
    fmt.Println(person, "was at the meeting")
}
```

Sometimes you need to distinguish a missing entry from a zero value. Is there an entry for "UTC" or is that the empty string because it's not in the map at all? You can discriminate with a form of multiple assignment.

때로는 부재값과 제로값을 구분할 필요도 있다. "UTC"에 대한 엔트리가 있는지 혹은 map내 정말 없어 빈 문자열인 건지? 복수 할당의 형태로 차이를 나타낼 수 있다.

```go
var seconds int
var ok bool
seconds, ok = timeZone[tz]
```

For obvious reasons this is called the “comma ok” idiom. In this example, if tz is present, seconds will be set appropriately and ok will be true; if not, seconds will be set to zero and ok will be false. Here's a function that puts it together with a nice error report:

뚜렷한 이유로 이것을 "comma ok" 관용구라고 부른다. 이 예제에서, 만약 tz가 있다면, seconds는 적절히 세팅될 것이고 ok는 true가 된다; 반면 없다면, seconds는 제로값이 되고 ok는 false가 된다. 여기 보기 좋은 에러 보고와 동반하는 함수의 예가 있다.

```go
func offset(tz string) int {
    if seconds, ok := timeZone[tz]; ok {
        return seconds
    }
    log.Println("unknown time zone:", tz)
    return 0
}
```

To test for presence in the map without worrying about the actual value, you can use the [blank identifier](https://golang.org/doc/effective_go.html#blank) (`_`) in place of the usual variable for the value.

실제 값에 상관없이 map내 존재 여부를 검사하려면, [공백 식별자](https://golang.org/doc/effective_go.html#blank) (`_`)를 값에 대한 변수가 있어야 할 자리에 놓으면 된다.

```go
_, present := timeZone[tz]
```

To delete a map entry, use the delete built-in function, whose arguments are the map and the key to be deleted. It's safe to do this even if the key is already absent from the map.

Map의 엔트리를 제거하기 위해서는, 내장 함수 delete을 쓰는데, map과 제거할 key를 인수로 쓴다. map에 key가 이미 부재하는 경우에도 안전하게 사용할 수 있다.

```go
delete(timeZone, "PDT")  // Now on Standard Time
```

## 출력

Formatted printing in Go uses a style similar to C's `printf` family but is richer and more general. The functions live in the fmt package and have capitalized names: `fmt.Printf`, `fmt.Fprintf`, `fmt.Sprintf` and so on. The string functions (`Sprintf` etc.) return a string rather than filling in a provided buffer.

Go에서 포맷된 출력은 C의 `printf`와 유사하지만 더 기능이 풍부하고 일반적인다. 함수들은 [fmt](https://godoc.org/fmt) 패키지에 있고 대문자화된 이름을 가진다: `[fmt.Printf](https://godoc.org/fmt#Printf)`, `[fmt.Fprintf](https://godoc.org/fmt#Fprintf)`, `[fmt.Sprintf](https://godoc.org/fmt#Sprintf)`, 기타 등등. 문자열 함수(`Sprintf`, 기타등등)은 제공된 버퍼를 채우기 보다는 문자열을 반환한다.

You don't need to provide a format string. For each of Printf, `Fprintf` and `Sprintf` there is another pair of functions, for instance `Print` and `Println`. These functions do not take a format string but instead generate a default format for each argument. The `Println` versions also insert a blank between arguments and append a newline to the output while the `Print` versions add blanks only if the operand on neither side is a string. In this example each line produces the same output.

반드시 포맷 문자열을 제공할 필요는 없다. `Printf`, `Fprintf` 그리고 `Sprintf`에 대해 짝을 이루는 함수들이 있는데, 예를 들면 `Print`와 `Println`이다. 이 함수들은 포맷 문자열을 취하지 않는 대신 입력된 인수에 대해 이미 정해진 포맷을 발생시킨다. `Println` 버전들은 인수들 사이에 공백을 삽입하고 출력에 줄바꿈을 추가한다. 그런 반면 `Print` 버전들은 인수 양쪽이 다 문자역이 난 경우에만 공백을 삽입한다. 아래 예제에서 각 줄이 같은 출력을 만든다.

```go
fmt.Printf("Hello %d\n", 23)
fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
fmt.Println("Hello", 23)
fmt.Println(fmt.Sprint("Hello ", 23))
```

The formatted print functions fmt.Fprint and friends take as a first argument any object that implements the `io.Writer` interface; the variables `os.Stdout` and `os.Stderr` are familiar instances.

포맷된 출력 함수인 [fmt.Fprint](https://godoc.org/fmt#Fprint)와 유사 함수들은 첫번째 인수로 `io.Writer` 인터페이스를 구현한 객체를 취한다; 변수 `os.Stdout`과 `os.Stderr`가 친숙한 인스턴스들이다.

Here things start to diverge from C. First, the numeric formats such as %d do not take flags for signedness or size; instead, the printing routines use the type of the argument to decide these properties.

여기서 부터 C로 부터 갈라져 나오기 시작한다. 첫째로, %d같은 숫자 형식들은 부호기호의 여부나 크기를 나타내는 플래그를 취하지 않는다; 그 대신, 출력 루틴이 인수의 타입을 이용해서 이러한 특징들을 결정한다.

```go
var x uint64 = 1<<64 - 1
fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
```

prints

위의 예제는 다음과 같은 출력을 한다.

```go
18446744073709551615 ffffffffffffffff; -1 -1
```

If you just want the default conversion, such as decimal for integers, you can use the catchall format %v (for “value”); the result is exactly what `Print` and `Println` would produce. Moreover, that format can print any value, even arrays, slices, structs, and maps. Here is a print statement for the time zone map defined in the previous section.

정수(integer)에 대한 소수같은 기본적인 변환을 원할 경우는, 다목적 용도 포맷인 %v(value라는 의미로)를 사용할 수 있다; 결과는 `Print`와 `Println`의 출력과 동일하다. 더우기, 그 포맷은 어떤 값이라도 출력할 수 있으며, 심지어 배열, slice, 그리고 map도 출력한다. 아래에는 이전 섹션에서 정의된 시간대 map을 위한 print문이 있다.

```go
fmt.Printf("%v\n", timeZone)  // or just fmt.Println(timeZone)
```

which gives output

아래와 같이 출력을 제공한다.

```go
map[CST:-21600 PST:-28800 EST:-18000 UTC:0 MST:-25200]
```

For maps the keys may be output in any order, of course. When printing a struct, the modified format %+v annotates the fields of the structure with their names, and for any value the alternate format %#v prints the value in full Go syntax.

물론, map의 경우 key들은 무작위로 출력될 수 있다. struct를 출력할 때는, 수정 포맷인 %+v를 통해 구조체의 필드에 주석으로 이름을 달며, 대안 포맷인 %#v를 사용하면 어떤 값이든 와전한 Go 문법을 출력한다.

```go
type T struct {
    a int
    b float64
    c string
}
t := &T{ 7, -2.35, "abc\tdef" }
fmt.Printf("%v\n", t)
fmt.Printf("%+v\n", t)
fmt.Printf("%#v\n", t)
fmt.Printf("%#v\n", timeZone)
```

prints

위의 예제는 다음과 같이 출력된다.

```go
&{7 -2.35 abc   def}
&{a:7 b:-2.35 c:abc     def}
&main.T{a:7, b:-2.35, c:"abc\tdef"}
map[string] int{"CST":-21600, "PST":-28800, "EST":-18000, "UTC":0, "MST":-25200}
```

(Note the ampersands.) That quoted string format is also available through %q when applied to a value of type string or []byte. The alternate format %#q will use backquotes instead if possible. (The %q format also applies to integers and runes, producing a single-quoted rune constant.) Also, %x works on strings, byte arrays and byte slices as well as on integers, generating a long hexadecimal string, and with a space in the format (% x) it puts spaces between the bytes.

(엠퍼센트에 주목하라.) 인용 문자열 포맷은 %q를 이용해 string 타입이나 []byte 타입 값에 적용했을때 얻어진다. 대안 포맷인 %#q는 가능한 경우 backquote을 사용한다. (%q 포맷은 또 integer와 rune에 적용되어, single-quoted rune 상수를 만든다.) %x는 문자열, byte 배열, 그리고 byte slice와 integer에 작용하여 긴 16진수 문자열을 생성하는데, (% x) 포맷처럼 스페이스를 중간에 넣으면 (출력하는) byte사이에 공백을 넣어 준다.

Another handy format is %T, which prints the type of a value.

또 다른 유용한 포맷은 %T로 값의 타입을 출력한다.

```go
fmt.Printf("%T\n", timeZone)
```

prints

위의 예제는 다음과 같이 출력된다.

```go
map[string] int
```

If you want to control the default format for a custom type, all that's required is to define a method with the signature String() string on the type. For our simple type T, that might look like this.

커스텀 타입의 기본 포맷을 조종하기 위해 요구되는 모든 조치는 단지 `String() string`의 시그너처를 갖는 메서드를 정의해 주면 된다. (위에 정의된) 단순한 타입 T는 아래와 같은 포맷을 가질 수 있다.

```go
func (t *T) String() string {
    return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
}
fmt.Printf("%v\n", t)
```

to print in the format

다음과 같은 포맷으로 출력된다.

```go
7/-2.35/"abc\tdef"
```

(If you need to print values of type T as well as pointers to T, the receiver for `String` must be of value type; this example used a pointer because that's more efficient and idiomatic for struct types. See the section below on [pointers vs. value receivers](https://golang.org/doc/effective_go.html#pointers_vs_values) for more information.)

(만약 타입 T와 동시에 포인터 타입 T도 함께 출력할 필요가 있으면, `String`의 리시버는 값 타입이러야 한다; 위에 예제에서 struct 타입에 포인터를 사용한 이유는 더 효율적이고 Go 언어다운 선택이기 때문이다. 더 상세한 정보는 다음의 링크를 참고하라: [pointers vs. value receivers](https://golang.org/doc/effective_go.html#pointers_vs_values) )

Our String method is able to call Sprintf because the print routines are fully reentrant and can be wrapped this way. There is one important detail to understand about this approach, however: don't construct a String method by calling Sprintf in a way that will recur into your String method indefinitely. This can happen if the Sprintf call attempts to print the receiver directly as a string, which in turn will invoke the method again. It's a common and easy mistake to make, as this example shows.

String 메서드가 Sprintf를 호출할 수 있는 이유는 print 루틴들의 재진입(reentrant)이 충분히 가능하고 예제와 같이 감싸도 되기 때문이다. 하지만 이 방식에 대해 한가지 이해하고 넘어가야 하는 매우 중요한 디테일이 있는데: String 매서드를 만들면서 Sprintf를 호출할 때 다시 String 매서드로 영구히 재귀하는 방식은 안 된다는 것이다. Sprintf가 리시터를 string처럼 직접 출력하는 경우에 이런 일이 발생할 수 있는데, 그렇게 되면 다시 같은 메서드를 호출하게 되고 말 것이다. 흔하고 쉽게 하는 실수로, 다음의 예제에서 살펴보자.

```go
type MyString string

func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", m) // Error: will recur forever.
}
```

It's also easy to fix: convert the argument to the basic string type, which does not have the method.

이러한 실수는 또 쉽게 고칠 수도 있다: 인수를 기본적인 문자열 타입으로 변환하면, 같은 메서드가 없기 때문이다.

```go
type MyString string
func (m MyString) String() string {
    return fmt.Sprintf("MyString=%s", string(m)) // OK: note conversion.
}
```

In the [initialization section](https://golang.org/doc/effective_go.html#initialization) we'll see another technique that avoids this recursion.

[initialization section](https://golang.org/doc/effective_go.html#initialization) 섹션에 가면 이 같은 재귀호출을 피할 수 있는 다른 테크닉을 보게 될 것이다.

Another printing technique is to pass a print routine's arguments directly to another such routine. The signature of `Printf` uses the type `...interface{}` for its final argument to specify that an arbitrary number of parameters (of arbitrary type) can appear after the format.

또 다른 출력 기법으로는 출력 루틴의 인수들을 직접 또 다른 유사한 루틴으로 대입하는 것이다. `Printf`의 시그너처는 마지막 인수로 임의적인 숫자의 파라미터가 포맷 다음에 나타날 수 있음을 명시하기 위해 타입 `...interface{}`를 사용한다.

```go
func Printf(format string, v ...interface{}) (n int, err error) {
```

Within the function `Printf`, v acts like a variable of type `[]interface{}` but if it is passed to another variadic function, it acts like a regular list of arguments. Here is the implementation of the function `log.Println` we used above. It passes its arguments directly to `fmt.Sprintln` for the actual formatting.

`Printf` 함수내에, v는 `[]interface{}` 타입의 변수 처럼 행동하지만 만약에 다른 가변 함수(variadic function)에 대입되면, 보통의 인수리스트로 동작한다. 위에서 사용한 `log.Println`의 구현이 아래에 있다. 실질적 출력을 위해 `fmt.Sprintln` 함수에 직접 인수들을 대입하고 있다.

```go
// Println prints to the standard logger in the manner of fmt.Println.
func Println(v ...interface{}) {
    std.Output(2, fmt.Sprintln(v...))  // Output takes parameters (int, string)
}
```

We write ... after v in the nested call to `Sprintln` to tell the compiler to treat v as a list of arguments; otherwise it would just pass v as a single slice argument.

`Sprintln`을 부르는 중첩된 호출안에 v 다음에 오는 ...는 컴파일러에게 v를 인수 리스트로 취급하라고 말하는 것이고; 그렇지 않은 경우는 v를 하나의 slice 인수로 대입한다.

There's even more to printing than we've covered here. See the `godoc` documentation for package fmt for the details.

여기에서 살펴 본 출력에 관한 내용보다 훨씬 많은 정보들이 있다. `godoc`에 있는 [fmt](https://godoc.org/fmt) 패키지를 통해 상세하게 알아보라.

By the way, a ... parameter can be of a specific type, for instance `...int` for a min function that chooses the least of a list of integers:

그나저나, ... 파라미터는 특정한 타입을 가질 수도 있는데, 예로 integer 리스트에서 최소값을 선택하는 함수인 min에 대한 `...int`를 살려보라.

```go
func Min(a ...int) int {
    min := int(^uint(0) >> 1)  // largest int
    for _, i := range a {
        if i < min {
            min = i
        }
    }
    return min
}
```

## Append

Now we have the missing piece we needed to explain the design of the append built-in function. The signature of append is different from our custom Append function above. Schematically, it's like this:

이제 내장함수 append의 설계를 설명하는데 필요했지만 없었던 부분을 갖게 되었다. append의 시그너처는 위에 주문제작된 Append 함수와 다르다. 의미론적으로, 아래와 같이 생각해 볼 수 있다.

```go
func append(slice []T, elements ...T) []T
```

where T is a placeholder for any given type. You can't actually write a function in Go where the type T is determined by the caller. That's why append is built in: it needs support from the compiler.

여기서 T는 어떤 타입의 플래이스홀더이다. 실제로 Go 언어에서는 호출자에 의해 결정되는 타입 T를 쓰는 함수를 만들 수 없다. 그래서 append는 내장함수 인 것이다: 컴파일러의 지원이 필요한 것이다.

What append does is append the elements to the end of the slice and return the result. The result needs to be returned because, as with our hand-written Append, the underlying array may change. This simple example

append이 하는 일은 slice의 끝에 요소들을 붙이고 결과를 반환하는 것이다. 결과는 반환되어야 한다. 왜냐면, 손으로 쓴 Append와 같이, 내부의 배열은 변할 수 있다. 다음의 간단한 예를 보라.

```go
x := []int{1,2,3}
x = append(x, 4, 5, 6)
fmt.Println(x)
```

prints [1 2 3 4 5 6]. So append works a little like `Printf`, collecting an arbitrary number of arguments.

이 예제는 [1 2 3 4 5 6]을 출력한다. append가 `Printf`처럼 임의적인 숫자의 인수들을 모으며 동작하는 것이다.

But what if we wanted to do what our Append does and append a slice to a slice? Easy: use ... at the call site, just as we did in the call to Output above. This snippet produces identical output to the one above.

그런데 만약 Append와 같이 slice에 slice를 붙이고 싶다면 어떻게 해야 하는가? 쉬운 방법으로, 위에서 Output을 호출하면서 그랬듯이, 호출 지점에서 ...를 이용하는 것이다. 아래 예재 단편은 위와 동일한 결과를 산출한다.

```go
x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x)
```

Without that ..., it wouldn't compile because the types would be wrong; y is not of type int.

...이 없다면 컴파일되지 않는다. 왜냐하면 타입이 틀리기 때문이다: y는 int 타입이 아니다.
