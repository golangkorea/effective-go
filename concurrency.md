# Concurrency

* 원문: [Concurrency](https://golang.org/doc/effective_go.html#concurrency)
* 번역자: Philbert Yoon (@ziwon)

## 통신에 의한 공유

Concurrent programming is a large topic and there is space only for some Go-specific highlights here.

동시성 프로그래밍은 광범위한 주제이므로 여기에서는 Go에 한정된 중요한 것들에 대해서만 지면을 할애한다.

Concurrent programming in many environments is made difficult by the subtleties required to implement correct access to shared variables. Go encourages a different approach in which shared values are passed around on channels and, in fact, never actively shared by separate threads of execution. Only one goroutine has access to the value at any given time. Data races cannot occur, by design. To encourage this way of thinking we have reduced it to a slogan:

공유 변수에 대한 정확한 접근을 구현하기 위해 엄수해야할 세세한 내용들은 다양한 환경에서 동시성 프로그래밍을 어렵게 했다. Go는 공유변수가 채널을 돌려가며 전달된다는 점에서 다른 접근을 권장한다. 그리고 사실 공유변수는 개별 쓰레드 실행에 의해서 결코 공유되지 않는다. 언제든지 하나의 고루틴이 값에 접근한다. 데이터 경쟁은 구현 설계상 발생할 수 없다. 이러한 사고방식을 권장하기 위해 이를 한 슬로건으로 줄였다.

Do not communicate by sharing memory; instead, share memory by communicating.

공유 메모리로 통신하지 말라. 대신, 통신으로 메모리를 공유하라.

This approach can be taken too far. Reference counts may be best done by putting a mutex around an integer variable, for instance. But as a high-level approach, using channels to control access makes it easier to write clear, correct programs.

이런 접근은 너무 지나친 것일 수 있다. 예를 들어, 정수형 변수 주위에 뮤텍스를 두는 방식의 레퍼런스 카운트가 최고일지도 모른다. 그러나 상위 레벨에서 접근하는 방법으로써, 접근을 제어하는 채널을 사용하면 분명하고 정확한 프로그램을 보다 쉽게 작성할 수 있다.

One way to think about this model is to consider a typical single-threaded program running on one CPU. It has no need for synchronization primitives. Now run another such instance; it too needs no synchronization. Now let those two communicate; if the communication is the synchronizer, there's still no need for other synchronization. Unix pipelines, for example, fit this model perfectly. Although Go's approach to concurrency originates in Hoare's Communicating Sequential Processes (CSP), it can also be seen as a type-safe generalization of Unix pipes.

이 모델에 대해 생각해보는 한가지 방법은 단일 CPU에서 실행되는 전형적인 단일 쓰레드 프로그램을 떠올려 보는 것이다. 여기에는 동기화를 위한 기본 자료형이 필요 없다. 지금 또다른 그 인스턴스를 실행시켜 보라. 그러나 역시 동기화가 필요하지 않다. 이제 그 두 개를 통신하게 하는데, 그 통신 자체가 동기화 장치(synchronizer)인 경우, 여전히 다른 동기화가 필요 없다. 예를 들어, 유닉스 파이프 라인은 이 모델에 완벽하게 들어 맞는다. 동시성에 대한 Go의 접근 방식이 호어(Hoare)의 통신 순차적 프로세스 (CSP, Communicating Sequential Processes)에서 비롯되었지만, 타입 안전이 보장되는 식의 일반화된 유닉스 파이프라고 볼 수 있다.

## 고루틴

They're called goroutines because the existing terms—threads, coroutines, processes, and so on—convey inaccurate connotations. A goroutine has a simple model: it is a function executing concurrently with other goroutines in the same address space. It is lightweight, costing little more than the allocation of stack space. And the stacks start small, so they are cheap, and grow by allocating (and freeing) heap storage as required.

쓰레드, 코루틴, 프로세스 등 기존의 용어는 부정확한 함의를 전달하기 때문에 고루틴이라고 부른다. 고루틴은 단순한 모델이다. 즉, 같은 주소 공간에서 다른 고루틴과 동시에 실행되는 함수이다. 고루틴은 가볍다. 스택 영역을 할당하는 것에 비해 비용이 적게 든다. 그리고 그 스택은 작은 크기로 시작된다. 그래서 저렴하다. 그리고 필요한만큼 힙 스토리지를 할당(또는 해제)하여 커진다.

Goroutines are multiplexed onto multiple OS threads so if one should block, such as while waiting for I/O, others continue to run. Their design hides many of the complexities of thread creation and management.

고루틴은 OS의 다중 쓰레드에 멀티플렉싱되는데, I/O 작업을 위해 대기중일 때와 같이 하나의 고루틴이 블락이 되면 다른 고루틴이 계속 실행된다. 이런 설계는 쓰레드의 복잡한 생성과 관리에 대해 굳이 알 필요가 없게 해준다.

Prefix a function or method call with the go keyword to run the call in a new goroutine. When the call completes, the goroutine exits, silently. (The effect is similar to the Unix shell's & notation for running a command in the background.)

새 고루틴을 호출하여 실행하려면 `go` 키워드를 함수 또는 메쏘드 호출 앞에 둔다. 호출이 완료되면, 고루틴은 자동으로 종료된다. (백그라운드에서 명령을 실행하는 유닉스쉘 및 표기법과 유사한 효과이다.)

```go
go list.Sort()  // list.Sort를 (정렬이 완료될 때까지) 기다리지 말고 동시에 실행
```

A function literal can be handy in a goroutine invocation.

함수 리터럴은 고루틴 호출에 유용할 수 있다.

```go
func Announce(message string, delay time.Duration) {
    go func() {
        time.Sleep(delay)
        fmt.Println(message)
    }() //괄호에 주목 - 반드시 함수를 호출해야 함
}
```

In Go, function literals are closures: the implementation makes sure the variables referred to by the function survive as long as they are active.

Go에서 함수 리터럴은 클로저이다. 즉, 함수가 참조하는 변수를 사용하는 동안에는 그 생존을 보장하는 방식으로 구현되어 있다는 것이다.

These examples aren't too practical because the functions have no way of signaling completion. For that, we need channels.

위의 예제는 함수가 종료를 알릴 방법이 없기 때문에 아주 실용적이진 않다. 이를 위해 채널이 필요하다.

## 채널

Like maps, channels are allocated with make, and the resulting value acts as a reference to an underlying data structure. If an optional integer parameter is provided, it sets the buffer size for the channel. The default is zero, for an unbuffered or synchronous channel

맵과 마찬가지로, 채널은 `make`로 할당되고, 그 결과 값은 실제 데이터 구조에 대한 참조로서 동작한다. 선택적인 정수형 매개변수가 주어지면, 채널에 대한 버퍼 크기가 설정된다. 이 값은 언버퍼드(Unbuffered) 또는 동기 채널에 대해서 기본값이 0이다.

```go
ci := make(chan int)            // 정수형의 언버퍼드 채널
cj := make(chan int, 0)         // 정수형의 언버퍼드 채널
cs := make(chan *os.File, 100)  // File 포인터형의 버퍼드 채널
```

Unbuffered channels combine communication—the exchange of a value—with synchronization—guaranteeing that two calculations (goroutines) are in a known state.

언버퍼드 채널은 동기화로 값을 교환하며 두 계산(고루틴들)이 어떤 상태에 있는지 알 수 있다는 것을 보장하는 통신을 결합한다.

There are lots of nice idioms using channels. Here's one to get us started. In the previous section we launched a sort in the background. A channel can allow the launching goroutine to wait for the sort to complete.

채널을 사용하는 멋있는 Go스러운 코드가 많다. 다음 한 예제로 시작해보자. 이전 섹션에서 백그라운드에서 정렬을 했다. 채널은 정렬이 완료될 때까지 고루틴 실행을 대기시킬 수 있다.

```go
c := make(chan int)  // 채널을 할당
// 고루틴에서 정렬 시작하고 완료되면 채널에 신호를 보냄
go func() {
    list.Sort()
    c <- 1  //  신호를 보내지만 값은 문제가 안됨
}()
doSomethingForAWhile()
<-c   // 정렬이 끝날 때까지 기다리고 전달된 값은 버림
```

Receivers always block until there is data to receive. If the channel is unbuffered, the sender blocks until the receiver has received the value. If the channel has a buffer, the sender blocks only until the value has been copied to the buffer; if the buffer is full, this means waiting until some receiver has retrieved a value.

수신부는 수신할 데이터가 있을 때까지 항상 블락된다. 언버퍼드 채널이면, 송신부는 수신부가 값을 받을 때까지 블락된다. 버퍼드 채널이면, 값이 버퍼에 복사될 때까지만 송신부가 블락된다. 그러므로 버퍼가 꽉 차면, 이는 특정 수신부가 값을 획득할 때까지 대기 중이라는 것을 의미한다.

A buffered channel can be used like a semaphore, for instance to limit throughput. In this example, incoming requests are passed to handle, which sends a value into the channel,processes the request, and then receives a value from the channel to ready the “semaphore” for the next consumer. The capacity of the channel buffer limits the number of simultaneous calls to process.

버퍼드 채널은 세마포처럼 사용될 수 있다. 예를 들어 처리량을 제한하는 것이다. 다음 예제에서, 들어오는 요청들은 `handle`에 넘겨진다. `handle`에서는 하나의 값(어떤 값이라도 상관없음)을 채널에 송신하고, 요청을 처리한 다음, 채널로부터 한 값을 받아 다음 소비자를 위해 "세마포"를 준비 시킨다. 채널 버퍼의 크기가 동시에 처리할 수 있는 숫자를 제한하는 것이다.

```go
var sem = make(chan int, MaxOutstanding)

func handle(r *Request) {
    sem <- 1    // 액티브큐가 비워질 때까지 대기
    process(r)  // 오래 걸릴 수 있는 작업
    <-sem       // 완료, 실행될 다음 요청을 활성화
}

func Serve(queue chan *Request) {
    for {
        req := <-queue
        go handle(req)  // 끝날 때까지 대기하지 않음
    }
}
```

Once MaxOutstanding handlers are executing process, any more will block trying to send into the filled channel buffer, until one of the existing handlers finishes and receives from the buffer.

일단 MaxOutstanding 수 만큼의 핸들러가 `process`를 실행하는 동안에는, 기존 핸들러 중 하나가 완료되고 버퍼로부터 값을 받을 때까지 더 이상의 꽉 찬 채널 버퍼에 송신하는 것은 블락될 것이다.

This design has a problem, though: Serve creates a new goroutine for every incoming request, even though only MaxOutstanding of them can run at any moment. As a result, the program can consume unlimited resources if the requests come in too fast. We can address that deficiency by changing Serve to gate the creation of the goroutines. Here's an obvious solution, but beware it has a bug we'll fix subsequently:

그렇지만 이 설계는 문제가 있다. 즉, 전체 요청에서 겨우 `MaxOutstanding` 수 만큼 `process`를 실행할 수 있음에도 서버는 들어오는 모든 요청에 대해 새로운 고루틴을 생성한다는 것이다. 그 결과, 요청이 너무 빨리 들어올 경우, 무제한으로 리소스를 낭비할 수 있다. 이 결함은 고루틴의 생성을 제한하는 게이트로 `Serve`를 수정함으로써 해결될 수 있다. 확실한 솔루션은 다음과 같지만, 나중에 수정하게 될 버그가 있다는 것에 주의하라.
Welcome to StackEdit!
===================


Hey! I'm your first Markdown document in **StackEdit**[^stackedit]. Don't delete me, I'm very helpful! I can be recovered anyway in the **Utils** tab of the <i class="icon-cog"></i> **Settings** dialog.

----------


Documents
-------------

StackEdit stores your documents in your browser, which means all your documents are automatically saved locally and are accessible **offline!**

> **Note:**

> - StackEdit is accessible offline after the application has been loaded for the first time.
> - Your local documents are not shared between different browsers or computers.
> - Clearing your browser's data may **delete all your local documents!** Make sure your documents are synchronized with **Google Drive** or **Dropbox** (check out the [<i class="icon-refresh"></i> Synchronization](#synchronization) section).

#### <i class="icon-file"></i> Create a document

The document panel is accessible using the <i class="icon-folder-open"></i> button in the navigation bar. You can create a new document by clicking <i class="icon-file"></i> **New document** in the document panel.

#### <i class="icon-folder-open"></i> Switch to another document

All your local documents are listed in the document panel. You can switch from one to another by clicking a document in the list or you can toggle documents using <kbd>Ctrl+[</kbd> and <kbd>Ctrl+]</kbd>.

#### <i class="icon-pencil"></i> Rename a document

You can rename the current document by clicking the document title in the navigation bar.

#### <i class="icon-trash"></i> Delete a document

You can delete the current document by clicking <i class="icon-trash"></i> **Delete document** in the document panel.

#### <i class="icon-hdd"></i> Export a document

You can save the current document to a file by clicking <i class="icon-hdd"></i> **Export to disk** from the <i class="icon-provider-stackedit"></i> menu panel.

> **Tip:** Check out the [<i class="icon-upload"></i> Publish a document](#publish-a-document) section for a description of the different output formats.


----------


Synchronization
-------------------

StackEdit can be combined with <i class="icon-provider-gdrive"></i> **Google Drive** and <i class="icon-provider-dropbox"></i> **Dropbox** to have your documents saved in the *Cloud*. The synchronization mechanism takes care of uploading your modifications or downloading the latest version of your documents.

> **Note:**

> - Full access to **Google Drive** or **Dropbox** is required to be able to import any document in StackEdit. Permission restrictions can be configured in the settings.
> - Imported documents are downloaded in your browser and are not transmitted to a server.
> - If you experience problems saving your documents on Google Drive, check and optionally disable browser extensions, such as Disconnect.

#### <i class="icon-refresh"></i> Open a document

You can open a document from <i class="icon-provider-gdrive"></i> **Google Drive** or the <i class="icon-provider-dropbox"></i> **Dropbox** by opening the <i class="icon-refresh"></i> **Synchronize** sub-menu and by clicking **Open from...**. Once opened, any modification in your document will be automatically synchronized with the file in your **Google Drive** / **Dropbox** account.

#### <i class="icon-refresh"></i> Save a document

You can save any document by opening the <i class="icon-refresh"></i> **Synchronize** sub-menu and by clicking **Save on...**. Even if your document is already synchronized with **Google Drive** or **Dropbox**, you can export it to a another location. StackEdit can synchronize one document with multiple locations and accounts.

#### <i class="icon-refresh"></i> Synchronize a document

Once your document is linked to a <i class="icon-provider-gdrive"></i> **Google Drive** or a <i class="icon-provider-dropbox"></i> **Dropbox** file, StackEdit will periodically (every 3 minutes) synchronize it by downloading/uploading any modification. A merge will be performed if necessary and conflicts will be detected.

If you just have modified your document and you want to force the synchronization, click the <i class="icon-refresh"></i> button in the navigation bar.

> **Note:** The <i class="icon-refresh"></i> button is disabled when you have no document to synchronize.

#### <i class="icon-refresh"></i> Manage document synchronization

Since one document can be synchronized with multiple locations, you can list and manage synchronized locations by clicking <i class="icon-refresh"></i> **Manage synchronization** in the <i class="icon-refresh"></i> **Synchronize** sub-menu. This will let you remove synchronization locations that are associated to your document.

> **Note:** If you delete the file from **Google Drive** or from **Dropbox**, the document will no longer be synchronized with that location.

----------


Publication
-------------

Once you are happy with your document, you can publish it on different websites directly from StackEdit. As for now, StackEdit can publish on **Blogger**, **Dropbox**, **Gist**, **GitHub**, **Google Drive**, **Tumblr**, **WordPress** and on any SSH server.

#### <i class="icon-upload"></i> Publish a document

You can publish your document by opening the <i class="icon-upload"></i> **Publish** sub-menu and by choosing a website. In the dialog box, you can choose the publication format:

- Markdown, to publish the Markdown text on a website that can interpret it (**GitHub** for instance),
- HTML, to publish the document converted into HTML (on a blog for example),
- Template, to have a full control of the output.

> **Note:** The default template is a simple webpage wrapping your document in HTML format. You can customize it in the **Advanced** tab of the <i class="icon-cog"></i> **Settings** dialog.

#### <i class="icon-upload"></i> Update a publication

After publishing, StackEdit will keep your document linked to that publication which makes it easy for you to update it. Once you have modified your document and you want to update your publication, click on the <i class="icon-upload"></i> button in the navigation bar.

> **Note:** The <i class="icon-upload"></i> button is disabled when your document has not been published yet.

#### <i class="icon-upload"></i> Manage document publication

Since one document can be published on multiple locations, you can list and manage publish locations by clicking <i class="icon-upload"></i> **Manage publication** in the <i class="icon-provider-stackedit"></i> menu panel. This will let you remove publication locations that are associated to your document.

> **Note:** If the file has been removed from the website or the blog, the document will no longer be published on that location.

----------


Markdown Extra
--------------------

StackEdit supports **Markdown Extra**, which extends **Markdown** syntax with some nice features.

> **Tip:** You can disable any **Markdown Extra** feature in the **Extensions** tab of the <i class="icon-cog"></i> **Settings** dialog.

> **Note:** You can find more information about **Markdown** syntax [here][2] and **Markdown Extra** extension [here][3].


### Tables

**Markdown Extra** has a special syntax for tables:

Item     | Value
-------- | ---
Computer | $1600
Phone    | $12
Pipe     | $1

You can specify column alignment with one or two colons:

| Item     | Value | Qty   |
| :------- | ----: | :---: |
| Computer | $1600 |  5    |
| Phone    | $12   |  12   |
| Pipe     | $1    |  234  |


### Definition Lists

**Markdown Extra** has a special syntax for definition lists too:

Term 1
Term 2
:   Definition A
:   Definition B

Term 3

:   Definition C

:   Definition D

	> part of definition D


### Fenced code blocks

GitHub's fenced code blocks are also supported with **Highlight.js** syntax highlighting:

```
// Foo
var bar = 0;
```

> **Tip:** To use **Prettify** instead of **Highlight.js**, just configure the **Markdown Extra** extension in the <i class="icon-cog"></i> **Settings** dialog.

> **Note:** You can find more information:

> - about **Prettify** syntax highlighting [here][5],
> - about **Highlight.js** syntax highlighting [here][6].


### Footnotes

You can create footnotes like this[^footnote].

  [^footnote]: Here is the *text* of the **footnote**.


### SmartyPants

SmartyPants converts ASCII punctuation characters into "smart" typographic punctuation HTML entities. For example:

|                  | ASCII                        | HTML              |
 ----------------- | ---------------------------- | ------------------
| Single backticks | `'Isn't this fun?'`            | 'Isn't this fun?' |
| Quotes           | `"Isn't this fun?"`            | "Isn't this fun?" |
| Dashes           | `-- is en-dash, --- is em-dash` | -- is en-dash, --- is em-dash |


### Table of contents

You can insert a table of contents using the marker `[TOC]`:

[TOC]


### MathJax

You can render *LaTeX* mathematical expressions using **MathJax**, as on [math.stackexchange.com][1]:

The *Gamma function* satisfying $\Gamma(n) = (n-1)!\quad\forall n\in\mathbb N$ is via the Euler integral

$$
\Gamma(z) = \int_0^\infty t^{z-1}e^{-t}dt\,.
$$

> **Tip:** To make sure mathematical expressions are rendered properly on your website, include **MathJax** into your template:

```
<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS_HTML"></script>
```

> **Note:** You can find more information about **LaTeX** mathematical expressions [here][4].


### UML diagrams

You can also render sequence diagrams like this:

```sequence
Alice->Bob: Hello Bob, how are you?
Note right of Bob: Bob thinks
Bob-->Alice: I am good thanks!
```

And flow charts like this:

```flow
st=>start: Start
e=>end
op=>operation: My Operation
cond=>condition: Yes or No?

st->op->cond
cond(yes)->e
cond(no)->op
```

> **Note:** You can find more information:

> - about **Sequence diagrams** syntax [here][7],
> - about **Flow charts** syntax [here][8].

### Support StackEdit

[![](https://cdn.monetizejs.com/resources/button-32.png)](https://monetizejs.com/authorize?client_id=ESTHdCYOi18iLhhO&summary=true)

  [^stackedit]: [StackEdit](https://stackedit.io/) is a full-featured, open-source Markdown editor based on PageDown, the Markdown library used by Stack Overflow and the other Stack Exchange sites.


  [1]: http://math.stackexchange.com/
  [2]: http://daringfireball.net/projects/markdown/syntax "Markdown"
  [3]: https://github.com/jmcmanus/pagedown-extra "Pagedown Extra"
  [4]: http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference
  [5]: https://code.google.com/p/google-code-prettify/
  [6]: http://highlightjs.org/
  [7]: http://bramp.github.io/js-sequence-diagrams/
  [8]: http://adrai.github.io/flowchart.js/

```go
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func() {
            process(req) // 버그, 아래 설명을 참고
            <-sem
        }()
    }
}
```

The bug is that in a Go for loop, the loop variable is reused for each iteration, so the req variable is shared across all goroutines. That's not what we want. We need to make sure that req is unique for each goroutine. Here's one way to do that, passing the value of req as an argument to the closure in the goroutine:

버그는 Go `for` 루프에 있다. 루프 변수는 각 반복마다 재사용되어 동일한 `req` 변수가 모든 고루틴에 걸쳐 공유된다. 이는 원하는 바가 아니다. 각 고루틴마다 구별된 `req` 변수를 가지도록 해야한다. 다음은 이를 위한 한 가지 방법으로, 고루틴의 클로져에 대한 인자로 `req`의 값을 전달하는 것이다.

{
```go다
func Serve(queue chan *Request) {
    for req := range queue {
        sem <- 1
        go func(req *Request) {
            process(req)
            <-sem
        }(req)
    }
}
```

Compare this version with the previous to see the difference in how the closure is declared and run. Another solution is just to create a new variable with the same name, as in this example:

이전 버전과 클로져가 어떻게 선언되고 실행되는지 차이점을 보기 위해 다음 버전을 비교해보라. 또다른 솔루션은 다음 예제처럼 그냥 같은 이름의 새로운 변수를 생성하는 것이다.

```go
func Serve(queue chan *Request) {
    for req := range queue {
        req := req // 고루틴을 위한 새로운 req 인스턴스를 생성
        sem <- 1
        go func() {
            process(req)
            <-sem
        }()
    }
}
```

It may seem odd to write

작성하는데 이상해 보일지도 모르겠다.

```go
req := req
```

but it's legal and idiomatic in Go to do this. You get a fresh version of the variable with the same name, deliberately shadowing the loop variable locally but unique to each goroutine.

하지만 Go에서 이렇게 하는 것은 합법적이고 Go 언어다운 코드이다. 이름이 같은 새로운 버전의 변수가 의도적으로 루프 변수를 지역적으로 가리지만, 각 고루틴에 대해서는 유니크한 값이다.

Going back to the general problem of writing the server, another approach that manages resources well is to start a fixed number of handle goroutines all reading from the request channel. The number of goroutines limits the number of simultaneous calls to process. This Serve function also accepts a channel on which it will be told to exit; after launching the goroutines it blocks receiving from that channel.

서버를 작성하는 일반적인 문제로 돌아가면, 리소스를 잘 관리하는 다른 방법은 요청 채널을 읽는 모든 `handle` 고루틴을 고정된 수에서 시작하는 것이다. 고루틴의 수는 `process`가 동시에 호출되는 수를 제한한다. `Serve` 함수는 종료 신호를 수신하게 되는 채널도 (인자로) 받고 있으므로 고루틴이 시작되면, 이 채널로부터의 수신은 블락된다.

```go
func handle(queue chan *Request) {
    for r := range queue {
        process(r)
    }
}

func Serve(clientRequests chan *Request, quit chan bool) {
    // 핸들러 시작
    for i := 0; i < MaxOutstanding; i++ {
        go handle(clientRequests)
    }
    <-quit  // 종료 신호를 받을 때까지 대기
}
```

## 채널의 채널

One of the most important properties of Go is that a channel is a first-class value that can be allocated and passed around like any other. A common use of this property is to implement safe, parallel demultiplexing.

Go의 가장 중요한 속성 중 하나는 채널이 다른 것과 마찬가지로 할당되고 전달될 수 있는 일급변수(first-class value)라는 것이다. 일반적으로 이 속성은 안전한 병렬 역다중화(parallel demultiplexing)를 구현하는데 사용된다.

In the example in the previous section, handle was an idealized handler for a request but we didn't define the type it was handling. If that type includes a channel on which to reply, each client can provide its own path for the answer. Here's a schematic definition of type Request.

이전 섹션의 예제에서, `handle` 은 요청에 대해서는 이상적인 핸들러였으나 핸들러가 다루는 타입은 정의하지 않았다. 해당 타입이 회신을 하는 채널을 포함하는 경우, 각 클라이언트는 자신에게 응답 경로를 제공할 수 있다. 다음은 `Request` 타입의 개략적인 정의이다.

```go
type Request struct {
    args        []int
    f           func([]int) int
    resultChan  chan int
}
```

The client provides a function and its arguments, as well as a channel inside the request object on which to receive the answer.

클라이언트는 함수와 그 인자뿐만 아니라 요청 객체 안에서 응답을 수신하는 채널을 제공하고 있다.

```go
func sum(a []int) (s int) {
    for _, v := range a {
        s += v
    }
    return
}

request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
// 요청 전송
clientRequests <- request
// 응답 대기
fmt.Printf("answer: %d\n", <-request.resultChan)
```

On the server side, the handler function is the only thing that changes.

서버 측에서는 핸들러 함수만 수정하면 된다.

```go
func handle(queue chan *Request) {
    for req := range queue {
        req.resultChan <- req.f(req.args)
    }
}
```

There's clearly a lot more to do to make it realistic, but this code is a framework for a rate-limited, parallel, non-blocking RPC system, and there's not a mutex in sight.

실제로 사용하기 위해서는 아직 할 일이 많은 것이 명백하다. 그러나 이 코드는 속도 제한, 병렬, 넌블락 RPC 시스템을 위한 프레임워크이다. 그리고 뮤텍스는 눈에 보이지 않는다.

## 병렬화

Another application of these ideas is to parallelize a calculation across multiple CPU cores. If the calculation can be broken into separate pieces that can execute independently, it can be parallelized, with a channel to signal when each piece completes.

이러한 아이디어의 또 다른 응용 프로그램은 멀티코어 CPU에 대해 계산을 병렬처리하는 것이다. 계산을 독립적으로 실행할 수 있는 부분들로 분리할 수 있다면, 각 부분들이 완료될 때 신호를 보내는 채널들로 병렬처리할 수 있다.

Let's say we have an expensive operation to perform on a vector of items, and that the value of the operation on each item is independent, as in this idealized example.

벡터 아이템에 대한 실행 비용이 비싼 연산이 있다고 가정해보자. 그리고 다음 이상적인 예제에서와 같이 각 아이템을 연산한 값은 독립적이다.

```go
type Vector []float64

//  v[i], v[i+1]에서 v[n-1]까지 연산을 적용
func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
        v[i] += u.Op(v[i])
    }
    c <- 1    // 이 부분이 완료되면 신호함
}
```

We launch the pieces independently in a loop, one per CPU. They can complete in any order but it doesn't matter; we just count the completion signals by draining the channel after launching all the goroutines.

루프에서 독립적으로 각 부분들을 CPU당 하나씩 실행시킨다. 이들은 어떤 순서로도 완료될 수 있지만 순서는 문제되지 않는다. 그러므로 모든 고루틴를 실행시킨 후 채널을 비워서 그냥 완료 신호를 카운트한다.


```go
const numCPU = 4 // CPU  코어수

func (v Vector) DoAll(u Vector) {
    c := make(chan int, numCPU)  // 버퍼는 선택적이나 상식적
    for i := 0; i < numCPU; i++ {
        go v.DoSome(i*len(v)/numCPU, (i+1)*len(v)/numCPU, u, c)
    }
    // 채널을 비움
    for i := 0; i < numCPU; i++ {
        <-c    // 한 태스크가 끝날 때까지 대기
    }
    // 모두 완료
}
```

Rather than create a constant value for numCPU, we can ask the runtime what value is appropriate. The function runtime.NumCPU returns the number of hardware CPU cores in the machine, so we could write

`numCPU`을 상수값으로 생성하기보다는 적절한 값을 런타임시에 요구할 수 있다. `runtime.NumCPU` 함수는 장비의 CPU 하드웨어 코어수를 반환한다. 그래서 아래와 같이 작성할 수 있다.

```go
var numCPU = runtime.NumCPU()
```

There is also a function runtime.GOMAXPROCS, which reports (or sets) the user-specified number of cores that a Go program can have running simultaneously. It defaults to the value of runtime.NumCPU but can be overridden by setting the similarly named shell environment variable or by calling the function with a positive number. Calling it with zero just queries the value. Therefore if we want to honor the user's resource request, we should write

또한 Go 프로그램은 동시에 실행할 수 있는 사용자 지정 코어수를 보고하는 (또는 설정하는) `runtime.GOMAXPROCS` 함수가 있다. `runtime.NumCPU`의 값이 기본값이지만, 비슷하게 명명된 환경 변수 설정에 의해 혹은 양수의 인자로 함수를 호출하여 재정의할 수 있다. 0으로 호출하면 바로 값을 조회한다. 따라서 사용자의 리소스 요청을 따르고 싶은 경우, 다음과 같이 작성해야 한다.

```go
var numCPU = runtime.GOMAXPROCS(0)
```

Be sure not to confuse the ideas of concurrency—structuring a program as independently executing components—and parallelism—executing calculations in parallel for efficiency on multiple CPUs. Although the concurrency features of Go can make some problems easy to structure as parallel computations, Go is a concurrent language, not a parallel one, and not all parallelization problems fit Go's model. For a discussion of the distinction, see the talk cited in this blog post.

컴포넌트를 독립적으로 처리하여 프로그램을 구조화하는 동시성과 다중 CPU에서 효율성을 위해 계산을 병렬로 처리하는 병렬성의 개념을 혼동하지 않길 바란다. Go의 동시성 특징이 병렬 계산으로 문제를 쉽게 구조화할 수 있지만, Go는 병렬이 아닌 동시성 언어이고, 모든 병렬화 문제가 Go에 들어맞지는 않는다. 이 구분에 대한 논의는 [이 블로그 포스트](https://blog.golang.org/concurrency-is-not-parallelism)에 인용된 토크를 참조하라.


## 누설 버퍼

The tools of concurrent programming can even make non-concurrent ideas easier to express. Here's an example abstracted from an RPC package. The client goroutine loops receiving data from some source, perhaps a network. To avoid allocating and freeing buffers, it keeps a free list, and uses a buffered channel to represent it. If the channel is empty, a new buffer gets allocated. Once the message buffer is ready, it's sent to the server on serverChan.

비동시성(non-concurrent) 개념도 동시성 프로그래밍 도구로 쉽게 표현할 수 있다. 다음은 RPC 패키지에서 추출한 예제이다. 클라이언트 고루틴은 아마도 네트워크인 특정 소스의 데이터를 반복해서 수신한다. 버퍼의 할당과 해제를 피하기 위해서 `free list`를 유지하며 이를 대신할 버퍼 채널을 사용한다. 채널이 비어 있으면 새로운 버퍼가 할당된다. 일단 메시지 버퍼가 준비되면 `serverChan`의 서버로 전송한다.

```go
var freeList = make(chan *Buffer, 100)
var serverChan = make(chan *Buffer)

func client() {
    for {
        var b *Buffer
        // 사용할 수 있는 버퍼를 획득하거나 그렇지 않다면 할당
        select {
            case b = <-freeList:
                // 하나를 획득하고 아무 작업도 하지 않음
            default:
                // 획득할 버퍼가 없으니 새 버퍼를 할당함
            b = new(Buffer)
        }
        load(b)              // 네트워크에서 다음 메세지를 읽음
        serverChan <- b      // 서버에 전송
    }
}
```

The server loop receives each message from the client, processes it, and returns the buffer to the free list.

서버 루프는 각 클라이언트로부터 메시지를 수신해서 처리하고, free list에 버퍼를 반환한다.

```go
func server() {
    for {
        b := <-serverChan    // 동작을 위해 대기
        process(b)
        // 가능할 경우 버퍼를 재사용
        select {
            case freeList <- b:
                // free list의 버퍼. 아무 것도 하지 않음
            default:
                // free list가 꽉 참, 그냥 계속
        }
    }
}
```

The client attempts to retrieve a buffer from freeList; if none is available, it allocates a fresh one. The server's send to freeList puts b back on the free list unless the list is full, in which case the buffer is dropped on the floor to be reclaimed by the garbage collector. (The default clauses in the select statements execute when no other case is ready, meaning that the selects never block.) This implementation builds a leaky bucket free list in just a few lines, relying on the buffered channel and the garbage collector for bookkeeping.

클라이언트는 `freeList`에서 버퍼를 획득하려고 시도한다. 그래서 어떤 버퍼도 사용할 수 없는 경우, 새로운 버퍼를 할당한다. `freeList`의 서버 송신은 리스트가 꽉 차지 않는 한 프리 리스트에 `b`를 다시 둔다. 사실 이럴 경우, 버퍼는 바닥에 떨어져 가비지 콜렉터에 의해 회수된다. (`select` 구문에서 `default`  절은 다른 case가 준비되지 않은 경우에 실행된다. 이는 `select`는 결코 블락되지 않는다는 것을 뜻한다.) 버퍼 채널과 장부 기록을 위한 가비지 컬렉터에 의존하는 단지 몇 줄의 구현으로  누설 버킷 프리 리스트(leaky bucket free list)을 만들고 있다.
