# Golang陷阱 （二）

```go
package interview

import (
	"testing"
	"io/ioutil"
	"fmt"
	"net/http"
	"io"
	"encoding/json"
	"log"
	"bytes"
	"encoding/base64"
	"reflect"
	"github.com/google/go-cmp/cmp"
	"github.com/google/go-cmp/cmp/cmpopts"
	"sync"
	"time"
)

func checkError(err error) {
	if err != nil {
		panic(err)
	}
}

/**
	body close的标准方式
	1. defer 前必须检查 resp是否为nil
	2. 必须check err ,

resp.Body.Close() 早先版本的实现是读取响应体的数据之后丢弃，保证了 keep-alive 的 HTTP 连接能重用处理不止一个请求。
但 Go 的最新版本将读取并丢弃数据的任务交给了用户，如果你不处理，HTTP 连接可能会直接关闭而非重用，参考在 Go 1.5 版本文档。


 */
func TestCloseBody(t *testing.T) {
	resp, err := http.Get("http://www.baidu.com")

	// 关闭 resp.Body 的正确姿势
	if resp != nil {
		defer resp.Body.Close()
	}

	checkError(err)
	defer resp.Body.Close()

	body, err := ioutil.ReadAll(resp.Body)
	checkError(err)
	fmt.Println(string(body))
	//resp.Body.Close() 早先版本的实现是读取响应体的数据之后丢弃，保证了 keep-alive 的 HTTP 连接能重用处理不止一个请求。
	// 手动丢弃读取完毕的数据
	_, err = io.Copy(ioutil.Discard, resp.Body)
}

/**
	对于 keep-alive的HTTP链接
标准库 "net/http" 的连接默认只在服务器主动要求关闭时才断开,这样很有导致服务器的socket 连接符被耗尽
解决办法 ：
	1. 在响应头中添加 "Connection"  "Close" : req.Header.Add("Connection", "close")    // 等效的关闭方式
	2. req.Close = true

如果你的应用只是向一个服务器发送请求，那么复用socket链接是经济的。
但是如果你的应用向大量的服务发送请求，且发送完毕之后立即关掉，需要请求完毕之后尽快像上述一样关闭socket复用
 */
func TestHttpClose(t *testing.T) {
	req, err := http.NewRequest("GET", "http://baidu.com", nil)
	checkError(err)

	//req.Close = true
	//req.Header.Add("Connection", "close")    // 等效的关闭方式

	resp, err := http.DefaultClient.Do(req)
	if resp != nil {
		defer resp.Body.Close()
	}
	checkError(err)

	body, err := ioutil.ReadAll(resp.Body)
	checkError(err)

	fmt.Println(string(body))
}

//在 encode/decode JSON 数据时，Go 默认会将数值当做 float64 处理，比如下边的代码会造成 panic：
func TestJsonFloat(t *testing.T) {
	var data = []byte(`{"status": 200}`)
	var result map[string]interface{}

	if err := json.Unmarshal(data, &result); err != nil {
		log.Fatalln(err)
	}

	fmt.Printf("%T\n", result["status"]) // float64
	var status = result["status"].(int)  // 类型断言错误
	fmt.Println("Status value: ", status)
}

/**
 json encoder 是为流设计的,对于json 流来讲，json对象之间的分隔符就相当于一个换行符
所以这里encode的结果会导致两个字符串不相等
 */
func TestJsonEncoder(t *testing.T) {
	data := map[string]int{"key": 1}

	var b bytes.Buffer
	json.NewEncoder(&b).Encode(data)

	raw, _ := json.Marshal(data)

	if b.String() == string(raw) {
		fmt.Println("same encoded data")
	} else {
		fmt.Printf("'%s' != '%s'\n", raw, b.String())
		//prints:
		//'{"key":1}' != '{"key":1}\n'
	}
}

/**
	go marshal 会将特殊的字符转换为 html character,
	所以如果你的字符串里面有类似 > < 这种时 需要call tbis enc.SetEscapeHTML(false)
 */
func TestSpecialCharacter(t *testing.T) {
	data := "x < y"

	raw, _ := json.Marshal(data)
	fmt.Println(string(raw))
	//prints: "x \u003c y" <- probably not what you expected

	var b1 bytes.Buffer
	json.NewEncoder(&b1).Encode(data)
	fmt.Println(b1.String())
	//prints: "x \u003c y" <- probably not what you expected

	var b2 bytes.Buffer
	enc := json.NewEncoder(&b2)
	enc.SetEscapeHTML(false)
	enc.Encode(data)
	fmt.Println(b2.String())
	//prints: "x < y" <- looks better
}

/**
	JSON String Values Will Not Be Ok with Hex or Other non-UTF8 Escape Sequences

 */
type config struct {
	Data string `json:"data"`
}

func TestJsonExpectUtf8(t *testing.T) {
	raw := []byte(`{"data":"\xc2"}`) //像这种HEX 格式的数字是无法进行 marshal 的。
	var decoded config
	if err := json.Unmarshal(raw, &decoded); err != nil {
		fmt.Println(err)
		//prints: invalid character 'x' in string escape code
	}

	//可以通过这种方式来进行处理，当然后面 反序列化之后需要自己进行处理
	raw = []byte(`{"data":"\\xc2"}`)
	json.Unmarshal(raw, &decoded)
	fmt.Printf("use double \\ array :  %#v \n", decoded) //prints: main.config{Data:"\\xc2"}
	//todo: do your own hex escape decoding for decoded.Data

	//use base 64 encode / decode
	base64EncodeStr := base64.StdEncoding.EncodeToString([]byte("\xc2"))
	var s string
	writer := bytes.NewBufferString(s)
	fmt.Fprintf(writer, "{\"data\":\"%s\"}", base64EncodeStr)
	fmt.Println(string(raw))
	if err := json.Unmarshal(raw, &decoded); err != nil {
		fmt.Println(err)
	}
	// than use base64 to decode
	dst := make([]byte, 0)
	fmt.Println(base64.StdEncoding.Decode(dst, []byte(decoded.Data)))
	fmt.Printf(" use bytes array : %#v", decoded) //prints: main.config{Data:[]uint8{0xc2}}

}

/*
  use go-cmps(https://github.com/google/go-cmp) to compare structs
 另外需要注意，对于 struct 中 un export 的属性,需要经过 options 来进行修饰定制
*/

type data struct {
	Num    int               //ok
	Checks [10]func() bool   //not comparable
	Doit   func() bool       //not comparable
	M      map[string]string //not comparable
	Bytes  []byte            //not comparable
}

func TestCompare(t *testing.T) {
	v1 := data{}
	v2 := data{}
	fmt.Println("v1 == v2:", reflect.DeepEqual(v1, v2)) //prints: v1 == v2: true
	fmt.Println("m1 == m2:", cmp.Equal(v1, v2, cmp.Options{
		cmp.AllowUnexported(data{}, cmpopts.IgnoreTypes()),
	}))                                                 //prints: m1 == m2: true
	m1 := map[string]string{"one": "a", "two": "b"}
	m2 := map[string]string{"two": "b", "one": "a"}
	fmt.Println("m1 == m2:", reflect.DeepEqual(m1, m2)) //prints: m1 == m2: true
	fmt.Println("m1 == m2:", cmp.Equal(m1, m2))         //prints: m1 == m2: true

	s1 := []int{1, 2, 3}
	s2 := []int{1, 2, 3}
	fmt.Println("s1 == s2:", reflect.DeepEqual(s1, s2)) //prints: s1 == s2: true
	fmt.Println("s1 == s2:", cmp.Equal(s1, s2))         //prints: m1 == m2: true
}

/**
	recover 只有在defer 直接调用时才会起作用，通过类似调用函数的方式也是不行的！
 */

func doRecover() {
	fmt.Println("recovered =>", recover()) //prints: recovered => <nil>
}
func TestRecover(t *testing.T) {
	defer func() { fmt.Println("recovered :", recover()) }() //worked
	defer doRecover()                                        // worked !
	defer func() { //not work
		doRecover()
	}()
	recover() //doesn't do anything
	panic("not good")
	recover() //won't be executed :)
	fmt.Println("ok")
	panic("not good ")
}

/**
	range 是对slice ，map, array 中的值拷贝，所以不要尝试同步
 */

func TestRangeCopy(t *testing.T) {
	nums := []int{1,2,3,4,5}
	//这样子是不行的
	for _,value := range nums{
		value *= 10
	}
	for _,value := range nums {
		fmt.Println(value)
	}
	//但是可以通过指针的方式进行修改
	a,b,c,d,e := 1,2,3,4,5
 	numWithPointer := []*int{&a,&b,&c,&d,&e}
 	for _,value := range numWithPointer {
 		*value *= 10
	}

	for _,value := range numWithPointer {
		fmt.Println(*value)
	}
}

/**
	When you reslice a slice, the new slice will reference the array of the original slice.
 	slice的底层结构是array ,在你对一个 slice进行切割后,新的slice依旧指向的是原来slice的array,且不会被释放
	这样的场景在大slice中尤其要注意，不然会消耗很多不必要的内存

 */
func get() []byte{
	raw := make([]byte,10000)
	fmt.Println(len(raw),cap(raw),&raw[0])
	return raw[:3]
}

func rightGet() []byte{
	raw := make([]byte,1000)
	fmt.Println(len(raw),cap(raw),&raw[0])
	res := make([]byte,3)
	copy(res,raw[:3])
	return res
}
func TestReslice(t *testing.T){

	// data := get() 3 10000 0xc042114000
	data := rightGet()
	fmt.Println(len(data),cap(data),&data[0])
}
/**
  下面的问题其实和上面的一样，注意 通过 剪切slice获得的slice 其实指向的是同一个
  array 数组,例如下面这种操作是存在问题的。
------------
归根到底就是，不要修改一个从其他slice得到的slice
 */
 func TestSliceCorruption(t *testing.T){
	 path := []byte("AAAA/BBBBBBBBB")
	 sepIndex := bytes.IndexByte(path,'/')
	 dir1 := path[:sepIndex]
	 dir2 := path[sepIndex+1:]
	 fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAA
	 fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => BBBBBBBBB

	 dir1 = append(dir1,"suffix"...)
	 path = bytes.Join([][]byte{dir1,dir2},[]byte{'/'})

	 fmt.Println("dir1 =>",string(dir1)) //prints: dir1 => AAAAsuffix
	 fmt.Println("dir2 =>",string(dir2)) //prints: dir2 => uffixBBBB (not ok)

	 fmt.Println("new path =>",string(path))
 }

 /**

  */
type myMutex sync.Mutex
/**
//通过这种方式也可以实现声明即调用
 type myLocker struct {
    sync.Mutex
}
 */
func TestDeclaration(t *testing.T){
	/**
	这种方式是不能工作的
	var mtx myMutex
	mtx.Lock()
	mtx.UnLock()
	 */
	//这种可以工作
	mtx := new(sync.Mutex)
	mtx.Lock()
	mtx.Unlock()

	/**
	 也可以通过 type声明的方式解决
	 */
}

/**
	break from for switch  and for select
 */

 func TestBreakFromLoop(t *testing.T){
 loop:
	 for {
		 switch {
		 case true:
			 fmt.Println("breaking out...")
			 break loop
		 }
	 }

	 fmt.Println("out!")
 }

 /**
  在同一个goroutine 中的所有for 循环，循环内声明的同名的变量会被充用
  */
func TestVariableInForLoop(t *testing.T){
	data := []string{"one","two","three"}

	for _,v := range data {
		go func(value string) {
			fmt.Println(value)
		}(v)
	}

	time.Sleep(3 * time.Second)
	//goroutines print: three, three, three
}

func TestFailTypeAssertions(t *testing.T){
	//虽然这种方式类型可以断言失败,但是原变量 的类型是不变的，这是一个大坑！
	var data1 interface{} = "great"
	if data1, ok := data1.(int); ok {
		fmt.Println("[is an int] value =>",data1)
	} else {
		fmt.Println("[not an int] value =>",data1)
		//prints: [not an int] value => 0 (not "great")
	}

	//通过新设置变量的方式可以完成！
	var data2 interface{} = "great"
	if res, ok := data2.(int); ok {
		fmt.Println("[is an int] value =>",res)
	} else {
		fmt.Println("[not an int] value =>",data2)
		//prints: [not an int] value => 0 (not "great")
	}
}

/**

 */



func TestSameAddrForDiffZeroSizedVar(t *testing.T){
	type data struct {
	}
	a := &data{}
	b := &data{}
	//你会发现它们的地址是一样的,其实也好理解,结构体data没有东西.所以其对象的地址偏移量为0
	fmt.Printf("same address - a=%p b=%p\n",a,b)
}
const (
	azero = iota
	aone  = iota
)

const (
	info  = "processing"
	bzero = iota
	bone  = iota
)

//iota的具体值是在 const的类似枚举的状态的下使用的
func TestIotaUsage(t *testing.T){
	fmt.Println(azero,aone) //prints: 0 1
	fmt.Println(bzero,bone) //prints: 1 2
}

```

