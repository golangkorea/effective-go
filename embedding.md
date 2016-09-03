# 임베딩(Embedding)

* 원문: [Embedding](https://golang.org/doc/effective_go.html#embedding)
* 번역자: Jhonghee Park

`Go does not provide the typical, type-driven notion of subclassing, but it does have the ability to “borrow” pieces of an implementation by embedding types within a struct or interface.`

Go는 전형적인, 타입주도형의 subclassing을 제공하지 않는다. 하지만 struct나 인터페이스안에 타입을 임베딩함으로써 구현체의 일부를 "빌리는"것은 가능하다.

`Interface embedding is very simple. We've mentioned the io.Reader and io.Writer interfaces before; here are their definitions.`

인터페이스 임베딩은 매우 간단하다. 이미 언급된 바 있는 [io.Reader](https://godoc.org/io#Reader)와 [io.Writer](https://godoc.org/io#Writer) 인터페이스의 정의를 살펴보자.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

`The io package also exports several other interfaces that specify objects that can implement several such methods. For instance, there is io.ReadWriter, an interface containing both Read and Write. We could specify io.ReadWriter by listing the two methods explicitly, but it's easier and more evocative to embed the two interfaces to form the new one, like this:`

[io](https://godoc.org/io)패키지는 또 다른 다수의 인터페이스들을 노출시키고 있는데, 다수의 메서드를 구현할 수 있는 객체를 명시하는데 씁니다. 예를 들어, [io.ReadWriter](https://godoc.org/io#ReadWriter)는 Read와 Write를 모두 가지고 있다. 두 메서드를 명시적으로 나열해서 [io.ReadWriter](https://godoc.org/io#ReadWriter)를 정의 할 수도 있겠지만, 더 쉽고 기억하기 좋은 방법은 두개의 인터페이스를 끼워 넣어서 다음과 같이 하나의 인터페이스를 형성시키는 것이다.

```go
// ReadWriter is the interface that combines the Reader and Writer interfaces.
type ReadWriter interface {
    Reader
    Writer
}
```

`This says just what it looks like: A ReadWriter can do what a Reader does and what a Writer does; it is a union of the embedded interfaces (which must be disjoint sets of methods). Only interfaces can be embedded within interfaces.`

보이는 데로 [Reader](https://godoc.org/io#Reader)가 하는 일과 [Writer](https://godoc.org/io#Writer)가 하는 일을 [ReadWriter](https://godoc.org/io#ReadWriter)는 모두 할 수 있다는 말이다; 끼워진(embedded) 인터페이스들의 조합이다 (서로 공통된 메서드가 없는 인터페이스들 이어야 한다). 오직 인터페이스만이 인터페이스에 끼워질 수 있다.

`The same basic idea applies to structs, but with more far-reaching implications. The bufio package has two struct types, bufio.Reader and bufio.Writer, each of which of course implements the analogous interfaces from package io. And bufio also implements a buffered reader/writer, which it does by combining a reader and a writer into one struct using embedding: it lists the types within the struct but does not give them field names.`

기본적으로 똑같은 아이디어를 struct에도 적용할 수 있는데 그 영향은 훨씬 광범위하다. [bufio](https://godoc.org/bufio)패키지에는 [bufio.Reader](https://godoc.org/bufio#Reader)와 [bufio.Writer](https://godoc.org/bufio#Writer) 두 struct타입이 있는데, 

```go
// ReadWriter stores pointers to a Reader and a Writer.
// It implements io.ReadWriter.
type ReadWriter struct {
    *Reader  // *bufio.Reader
    *Writer  // *bufio.Writer
}
```

The embedded elements are pointers to structs and of course must be initialized to point to valid structs before they can be used. The ReadWriter struct could be written as

```go
type ReadWriter struct {
    reader *Reader
    writer *Writer
}
```

but then to promote the methods of the fields and to satisfy the io interfaces, we would also need to provide forwarding methods, like this:

```go
func (rw *ReadWriter) Read(p []byte) (n int, err error) {
    return rw.reader.Read(p)
}
```

By embedding the structs directly, we avoid this bookkeeping. The methods of embedded types come along for free, which means that bufio.ReadWriter not only has the methods of bufio.Reader and bufio.Writer, it also satisfies all three interfaces: io.Reader, io.Writer, and io.ReadWriter.

There's an important way in which embedding differs from subclassing. When we embed a type, the methods of that type become methods of the outer type, but when they are invoked the receiver of the method is the inner type, not the outer one. In our example, when the Read method of a bufio.ReadWriter is invoked, it has exactly the same effect as the forwarding method written out above; the receiver is the reader field of the ReadWriter, not the ReadWriter itself.

Embedding can also be a simple convenience. This example shows an embedded field alongside a regular, named field.

```go
type Job struct {
    Command string
    *log.Logger
}
```

The Job type now has the Log, Logf and other methods of `*log.Logger`. We could have given the Logger a field name, of course, but it's not necessary to do so. And now, once initialized, we can log to the Job:

```go
job.Log("starting now...")
```

The Logger is a regular field of the Job struct, so we can initialize it in the usual way inside the constructor for Job, like this,

```go
func NewJob(command string, logger *log.Logger) *Job {
    return &Job{command, logger}
}
```

or with a composite literal,

```go
job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
```

If we need to refer to an embedded field directly, the type name of the field, ignoring the package qualifier, serves as a field name, as it did in the Read method of our ReaderWriter struct. Here, if we needed to access the `*log.Logger` of a Job variable job, we would write job.Logger, which would be useful if we wanted to refine the methods of Logger.

```go
func (job *Job) Logf(format string, args ...interface{}) {
    job.Logger.Logf("%q: %s", job.Command, fmt.Sprintf(format, args...))
}
```

Embedding types introduces the problem of name conflicts but the rules to resolve them are simple. First, a field or method X hides any other item X in a more deeply nested part of the type. If log.Logger contained a field or method called Command, the Command field of Job would dominate it.

Second, if the same name appears at the same nesting level, it is usually an error; it would be erroneous to embed log.Logger if the Job struct contained another field or method called Logger. However, if the duplicate name is never mentioned in the program outside the type definition, it is OK. This qualification provides some protection against changes made to types embedded from outside; there is no problem if a field is added that conflicts with another field in another subtype if neither field is ever used.
