## Interface

Go의 인터페이스는 **묵시적으로** 적용됨 ( java에서 implement 키워드로 인터페이스를 구현하는 것과 아주 다르다 )

* 주어진 Concrete type에 요구되는 모든 인터페이스를 선언할 필요 없이, 단순히 필요한 메소드만 있으면 충분함
* 이런 설계로 인해 기존 Concrete type을 충족하는 새 인터페이스를, 기존 타입의 변경 없이 생성할 수 있으며 특히 제어할 수 없는 패키지 타입을 사용할 때 유용 
* 인터페이스와 구현체가 상하 관계가 아니라 **수평적인** 관계가 됨

** Concrete type : 값의 정확한 표현을 지정하고, 메소드를 통해 부가적인 동작이 정확히 정의되어 있음*



#### Ex. Io.Writer

io.Writer 인터페이스는 Fprintf와 호출자간의 규약을 정의함. 호출자는 규약에 따라 io.Writer의 Write() 메소드가 구현된 Concrete type의 값을 넘겨줘야 한다. Fprintf는 규약에 의해 io.Writer 인터페이스를 충족하는 어떤 값이 오더라도 주어진 일을 수행할 것이 보장됨 

```go
package fmt

// io.Writer 인터페이스를 충족하는 어떤 값이 오더라도 주어진 일을 수행할 것이 보장됨 
func Fprintf(w io.Writer, format string, args ...interface{})(int, error)

func Printf(format string, args ...interface{})(int, error){
  // os.stdout(io.Writeer 구현체)에 쓴다
  return Fprintf(os.stdout, format, args...)
}

func Sprintf(format string, args ...interface{}) string{
  var buf bytes.Buffer
  // buf(io.Writer 구현체)에 쓴다 
  Fprint(&buf, format, args...)
  return buf.String()
}
```

* **Substitutability (대체 가능성)** : 한 타입을 동일한 인터페이스를 충족하는 다른 타입으로 자유롭게 변경하는 것



### Interface Embedding

기존의 인터페이스의 조합으로 이뤄진 새 인터페이스를 정의할 수 있음

```go
type Reader interface{
	Read(p []byte) (n int, err error)
}

type Writer interface{
	Write(p []byte) (n int, err error)
}

type ReadWriter interface{
	Reader
	Writer
}

type ReadWriter2 interface{
	Reader
	Writer(p []byte) (n int, err error)	// Embedding과 혼합도 가능 
}
```



### Interface 충족

타입 안에 인터페이스에서 요구하는 모든 메소드가 있으면 이 타입이 인터페이스를 **충족**한다고 함. 보통 Concrete type이 특정 인터페이스 타입의 `한 종류(is a)` 라고 부른다

인터페이스를 충족하지 않는 타입을 해당 인터페이스에 할당하려는 경우, 컴파일 에러가 발생

```go
var w io.Writer
w = os.Stdout					// ok
w = new(bytes.Buffer)	// ok
w = time.Second 			// compile error

var rc io.ReadCloser
var rw io.ReadWriter
w = rc		// compile error
w = rw		// ok (Write메소드 가지고 있음)
```



#### Type이 method를 갖는다?

Go에서는 Value 인스턴스가 포인터 리시버를 갖는 메소드를 바로 호출하는 것이 가능했으나, 이는 Go에서 묵시적으로 해당 인스턴스 앞에 &를 붙여주었기 때문이지 실제로 두 타입(Value vs Pointer)을 혼용하여 쓰는 것이 가능한 건 아님! 다른 애들임  

따라서 인터페이스가 요구하는 메소드를 정.확.히 해당 타입이 가지고 있을 때에만, 해당 인터페이스를 충족한다고 할 수 있다

```go
type IntSet struct { ... }
func (*IntSet) String() string{ ... }
func test(s *IntSet)

var s IntSet
var _ = s.String() 	// (&s).String() 으로 내부적으로 변환
test(s)			// Compile Error! 
test(&s)		// ok

var _, fmt.Stringer = s		// Compile Error! s에는 String 메소드가 없음
var _, fmt.Stringer = &s	// ok
```



#### Empty Interface

빈 인터페이스 타입은 어떤 값도 할당할 수 있음.  따라서 함수에서 타입의 구분없이 parameter를 받고 싶은 경우 `interface{}` 타입으로 받아오면 된다. 

```go
var any interface{}
any = true
any = 2.32132
any = "hello"
any = map[string]int{"one":1}
any = new(byte.Buffer)

// 따라서 타입의 구분없이 parameter를 받고 싶은 경우 
// 아래와 같이 args ...interface{} 이런 식으로 받으면 됨 
func Fprintf(w io.Writer, format string, args ...interface{})(int, error)
```



### Interface Value

인터페이스 타입의 값이나 인터페이스 값에는 두 개의 구성 요소인 `Concrete type`과 `값`이 있음.  이를 인터페이스의  `동적 타입` 과 `동적 값` 이라고 함 *( Go와 같은 정적 타입 언어에서는 타입이 컴파일 시의 개념이므로, 타입은 값이 아님 )* 

* **Type Descriptor**에 각 타입에 대한 이름과 메소드 같은 정보가 있음



#### Example

```go
var w io.Writer					// nil
w = os.Stdout	
w.Write([]byte("hello"))
```

<img src="../images/go-interface.png" alt="go-interface" style="zoom: 50%;" />

* 일반적으로 컴파일 시에는 인터페이스 값의 동적 타입을 예상할 수 없으므로, 인터페이스를 통한 호출은 동적으로 이뤄져야 함
* 컴파일러는 직접 호출하는 대신 Type Descriptor에서 Write 메소드의 주소를 얻는 코드를 작성하고, 이 코드로 얻은 주소를 간접 호출
* 호출 시 Receiver는 인터페이스와 동적 값인 os.Stdout의 복사본임



#### nil pointer가 있는 interface

`아무 값도 담고 있지 않은 nil 인터페이스의 값`은 `nil일 수도 있는 포인터를 갖는 인터페이스 값`과 다르다

```go
const debug = true
func main() {
  var buf *bytes.Buffer
  if debug {
    buf = new(bytes.Buffer) 
  }
  f(buf) 
  if debug {
    // ...use buf...
  } 
}

// out이 nil이 아니면 write
func f(out io.Writer) {
  // ...do something...
  if out != nil {
    out.Write([]byte("done!\n"))
	} 
}
```

* 위에서 debug = false로 바꾸면, buf가 nil인 채로 f에 넘어온다

* 이때, out에는 *bytes.Buffer 타입의 nil 포인터가 할당된다. 

* 즉 **out 자체는 nil이 아니기 때문에** `out != nil `이 True가 되어 out.Write(~) 이 호출되어 패닉이 발생하게 됨

  ( 집합에서 `공집합`과 `공집합을 원소로 갖는 집합`이 다르듯이! )

  <img src="../images/go-interface2.png" alt="go-interface2" style="zoom:50%;" />



### Type Assertion

인터페이스 값에 적용되는 연산으로 피연산자의 동적 타입이 단언 타입과 일치하는지를 확인한다

```go
var w io.Writer
w = os.Stdout
f := w.(*os.File)      // success: w의 동적 타입이 os.File이므로 ok
c := w.(*bytes.Buffer) // panic: w의 동적 타입이 *bytes.Buffer가 아니므로 
```

-  `[interface-type].(assertin-type)` 형태로 이루어짐

- 해당 인터페이스의 동적 타입을 확인하고 해당 타입이 아닌 경우 패닉이 일어남

```go
var w io.Writer
w = os.Stdout
// 아래와 같은 방법으로 인터페이스의 동적 타입을 확인할 수 있음
// *os.File는 Read & Write 메소드를 모두 가지고 있음
// ok = true, rw = io.ReadWriter 
w : read / write
rw, ok := w.(io.Reader) 		// io.Reader 처럼 (rw.Write() xxxx // rw := rw.(io.Writer) panic!!!) 
rw, ok := w.(*io.File)			// io.File 처럼 
rw, ok := w.(*os.intf) 			// 구체적인 타입
rw, ok := w.(os.intf)				// 인터페이스 


w = new(byteCounter)
rw, ok = w.(io.ReadWriter) 	// ok = false , rw = nil
```



#### Type Assertion 으로 동작 조회

```go
// writeString는 w에 s를 쓴다
// w가 WriteString method를 가지면, w.Write 대신 WriteString을 호출
func writeString(w io.Writer, s string) (n int, err error) {
	type stringWriter interface {
    WriteString(string) (n int, err error)
	}
	if sw, ok := w.(stringWriter); ok {
    return sw.WriteString(s) 
	}
	return w.Write([]byte(s)) // allocate temporary copy
}
 
func writeHeader(w io.Writer, contentType string) error {
	if _, err := writeString(w, "Content-Type: "); err != nil {
		return err
	}
	if _, err := writeString(w, contentType); err != nil {
		return err
	}
	// ...
}
```

* WriteString만 가진 인터페이스를 정의하고, Type Assertion으로 w가 해당 인터페이스를 충족하는지(WriteString 메소드를 가지는지) 확인한다.



### Type Switches

```go
func toString(x interface{}) string{
	switch x := x.(type){
	case nil:
		return "NULL"
  case int, uint:
  	return fmt.Sprintf("%d", x)
  case bool:
  	if x {
  		return "TRUE"
  	}
  	return "FALSE"
  case string:
  	return x
  default :
  	panic(fmt.Sprintf("unexpected type %T : %v", x,x))
	}
}
```



### Exercise

```go
package main

import (
	"fmt"
	"io"
	"os"
)

type byteCounter struct {
	w       io.Writer
	written int64
}

func (c *byteCounter) Write(p []byte) (int, error) {
	n, err := c.w.Write(p)
	c.written += int64(n)
	return n, err
}

func (c *byteCounter) GetWritten() (int64){
	return c.written
}

func CountingWritter(w io.Writer) (io.Writer, *int64) {
	c := &byteCounter{w, 0}
	return c, &(c.written)
}

func main() {
	c, n := CountingWritter(os.Stdout)
	data := []byte("hello world!")
	c.Write(data)

	if *n != int64(len(data)){
		fmt.Printf("Error! %d != %d\n", *n, len(data))
	}
}
```



### Go Master 분들의 Tips 

* 인터페이스 안의 메소드는 1~2개가 적당하다! ( 너무 많으면 유연성이 떨어지며, 인터페이스를 만든 의미가 줄어든다 )

* 라이브러리성 코드가 아니면 최대한 인터페이스를 사용하지 말자! 

  ( 인터페이스는 두 개 이상의 Concrete type을 같은 방식으로 처리할 때에만 필요)

* 기존 인터페이스에 메소드를 추가하고 싶은 경우, 기존 인터페이스의 메소드 + 추가할 메소드를 가지는 인터페이스를 **새로** 정의하고, Type Assertion으로 동작을 조회하여 구분한다

  ```go
  if ( 기존 인터페이스인데 추가 메소드를 가진 인터페이스이면 ) { 
  	// 새로운 인터페이스 타입으로 사용 ...
  }else{
  	// 기존 인터페이스 타입으로 사용 ...
  }
  ```

* 인터페이스 이름은 Reader, Writer, Closer 처럼 보통 행위 주체자? 형태로 이름을 붙인다

* Go에 OOP를 위한 많은 지원이 있지만, 이것이 반드시 OOP만을 사용해야 한다는 뜻은 아니다!  
