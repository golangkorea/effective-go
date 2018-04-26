# 임베딩(Embedding)

* 원문: [Embedding](https://golang.org/doc/effective_go.html#embedding)
* 번역자: Jhonghee Park


Go는 전형적인, 타입주도형의 subclassing을 제공하지 않는다. 하지만 struct나 인터페이스안에 타입을 임베딩함으로써 구현체의 일부를 "빌리는"것은 가능하다.


인터페이스 임베딩은 매우 간단하다. 이미 언급된 바 있는 [io.Reader](https://godoc.org/io#Reader)와 [io.Writer](https://godoc.org/io#Writer) 인터페이스의 정의를 살펴보자.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```


[io](https://godoc.org/io)패키지는 다수의 또 다른 인터페이스들을 노출시키고 있는데, 다수의 메서드를 구현할 수 있는 객체를 명시하는데 쓴다. 예를 들어, [io.ReadWriter](https://godoc.org/io#ReadWriter)는 Read와 Write를 모두 가지고 있다. 두 메서드를 명시적으로 나열해서 [io.ReadWriter](https://godoc.org/io#ReadWriter)를 정의 할 수도 있겠지만, 더 쉽고 기억하기 좋은 방법은 두개의 인터페이스를 끼워 넣어서(embedded) 다음과 같이 하나의 인터페이스를 형성시키는 것이다:

```go
// ReadWriter는 Reader와 Writer 인터페이스를 조합하는 인터페이스이다.
type ReadWriter interface {
    Reader
    Writer
}
```


보이는 데로 [Reader](https://godoc.org/io#Reader)가 하는 일과 [Writer](https://godoc.org/io#Writer)가 하는 일을 [ReadWriter](https://godoc.org/io#ReadWriter)는 모두 할 수 있다는 말이다; 끼워진(embedded) 인터페이스들의 조합이다 (서로 공통된 메서드가 없는 인터페이스들 이어야 한다). 오직 인터페이스만이 인터페이스에 끼워질 수 있다.


기본적으로 똑같은 아이디어가 struct에도 적용할 수 있는데 그 영향은 훨씬 광범위하다. [bufio](https://godoc.org/bufio)패키지에는 [bufio.Reader](https://godoc.org/bufio#Reader)와 [bufio.Writer](https://godoc.org/bufio#Writer) 두 struct타입이 있는데, 물론 [io](https://godoc.org/io)패키지에 있는 유사한 인터페이스를 구현하고 있다. 그런데 [bufio](https://godoc.org/bufio)는 또 버퍼를 내재한 reader/writer를 구현하기도 하는데, 임베딩(embedding)을 이용하여 reader와 writer를 하나의 struct에 조합하는 것이다: struct안에 타입들을 나열하되 필드 이름을 주지 않는 방식이다.

```go
// ReadWriter Reader와 Writer에 대한 포인터들을 저장한다.
// io.ReadWriter를 구현한다.
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```


임베드된 요소들은 struct를 가리키는 포인터들이고, 물론 사용하기 전에 유효한 struct로 포인트를 걸어서 초기화 시켜주어야 한다. ReadWriter struct는 아래와 같이 작성될 수 있다.

```go
type ReadWriter struct {
    reader *Reader
    writer *Writer
}
```


하지만 이렇게 하면 [io](https://godoc.org/io)를 충족시키고 reader와 writer가 가지고 있는 메서드를 사용하기 위해서 전송용 메서드(forwarding method)를 따로 제공해야 한다:

```go
func (rw *ReadWriter) Read(p []byte) (n int, err error) {
    return rw.reader.Read(p)
}
```


이런 귀찮은 일들을 피하기 위해서는 struct를 직접 임베딩하면 된다. 임베드된 타입의 메서드는 자동으로 따라 오며, 그 의미는 [bufio.ReadWriter](https://godoc.org/bufio#ReadWriter)는 [bufio.Reader](https://godoc.org/bufio#Reader)와 [bufio.Writer](https://godoc.org/bufio#Writer)의 메서드를 모두 가지게 된다는 말이다. 동시에 다음의 세 인터페이스를 충족시키기도 한다: [io.Reader](https://godoc.org/io#Reader), [io.Writer](https://godoc.org/io#Writer), 그리고 [io.ReadWriter](https://godoc.org/io#ReadWriter).


이제 임베딩이 subclassing과 다른 중요한 방식을 살펴보자. 타입을 임베드하게 되면 그 타입의 메서드들이 외부 타입의 메서드가 된다. 하지만 호출된 메서드의 리시버는 내부 타입이지 외부 타입이 아니다. 예제에서, [bufio.ReadWriter](https://godoc.org/bufio.ReadWriter)의 Read 메서드가 호출되었을 때, 전달용 메서드를 사용한 것과 같은 똑같은 효과가 있다; 리시버는 ReadWriter의 reader 필드이지, ReadWriter 자체가 아닌 것이다.


임베딩은 또한 단순한 편리함일 수 있다. 이 예제는 임베드된 필드를 이름이 있는 보통 필드와 함께 보여준다.

```go
type Job struct {
    Command string
    *log.Logger
}
```


Job 타입은 이제 `*log.Logger`에 속한 Log, Logf 그리고 다른 메서드들을 가지게 된다. [Logger](https://godoc.org/log#Logger)에 이름을 줄 수도 있었겠지만, 그럴 필요가 전혀 없다. 그리고 이제, 일단 초기화가 되면, Job에 직접 log를 사용할 수 있다:

```go
job.Log("starting now...")
```


Logger는 Job struct의 보통 필드이기 때문에 constructor 안에서 항상 하는 것 처럼 다음과 같이 초기화 할 수 있다.

```go
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
```


혹은 composite literal을 써서

```go
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```


만약 임베드된 필드를 직접 언급해야 할 경우가 생기면, ReadWriter struct의 Read 메서드처럼, 패키지를 무시한 필드의 타입명이 필드의 이름으로 사용된다. Job 타입의 변수인 job의 `*log.Logger`에 접근할 필요가 있다면, job.Logger라고 쓰면 되고, Logger의 메서드를 개선하길 원할 때 유용하다.

```go
func (job *Job) Logf(format string, args ...interface{}) {
    job.Logger.Logf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```


임베딩 타입들은 이름 충돌의 문제를 발생시킬 수도 있지만 해결하는 규칙들은 간단하다. 첫번째, 필드나 메서드 X는 타입내 더 깊숙히 중첩된 부분에 있는 또 다른 X를 가려서 안 보이게 한다. 만약 log.Logger가 Command라는 필드나 메서드가 있다면, Job의 Command 필드가 더 우세하다.


둘째로, 똑같은 이름이 같은 레벨로 중첩되어 나타나면, 보통은 에러가 생긴다; 만약에 Job struct이 Logger라는 이름으로 다른 필드나 메서드를 가지고 있다면 log.Logger를 임베드하는 것은 잘못된 것이다. 그렇지만, 복제된 이름이 타입 정의 밖에서 사용된 일이 없다면 괜찮다. 이러한 자격은 밖으로 부터 임베드된 타입에 생기는 변화에 대해서 어느 정도 보호를 제공한다; 필드 하나가 추가되면서 또 다른 subtype의 필드와 충돌이 생기는 경우, 둘 중 어느 필드도 사용되지 않았다면 아무 문제가 없다.
