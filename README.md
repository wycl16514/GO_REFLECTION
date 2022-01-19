有一些高级语言提供了涉及到编程元素深层信息的接口，这些信息通常是运行时或编译器有用，但语言也通过接口将其暴露出来，这样开发者就能使用它们实现一些类似黑客的功能。这些能让开发者攫取到编程元素深层信息或者进行深度操作的接口就叫反射，在Go和Java都有提供，运用好反射功能可以开发出功能强大的程序，但是反射由于涉及到编译原理，因此比较抽象，在此我们用丰富的例子来说清楚GO的反射接口应用。

Go的反射接口来自于reflect包，其中大部分反射功能都来自3个对象，分别为reflect.Type, reflect.Value, reflect.Kind。其中Type用来获取变量对象的类型信息，Value用来获取变量的值，Kind跟Type类似，也是获得变量类型信息，但是跟Type有所不同，我们看个具体例子:

import (
    "fmt"
    "reflect"
)

func main() {
    var s string
    s = "hello"
    s_type := reflect.TypeOf(s) //返回 reflect.Type
    fmt.Println("s_type:", s_type)   //输出 string
    s_value := reflect.ValueOf(s) //返回reflect.Value
    fmt.Println("s_value:", s_value)  //输出 hello
    s_kind := s_value.Kind() //返回 reflect.Kind 
    fmt.Println("s_kind:", s_kind) //输出 string
}
reflect.TypeOf 返回的类型为reflect.Type对象，reflect.ValueOf返回reflect.Value对象，reflect.Value.Kind返回reflect.Kind对象。从上面代码的返回看不好分辨reflect.Type和reflect.Kind的区别，我们再看一个例子：

import (
    "fmt"
    "reflect"
)

type Employee struct {
    Name    string
    Age     uint32
    Salary  float32
    Has_pet bool
}

func main() {
    var e Employee
    e_value := reflect.ValueOf(e)
    fmt.Println("e_value: ", e_value)
    e_type := e_value.Type()
    fmt.Println("e_type: ", e_type)
    e_kind := e_value.Kind()
    fmt.Println("e_kind: ", e_kind)
}
上面代码运行后输出结果为：

e_value:  { 0 0 false}
e_type:  main.Employee
e_kind:  struct
从结果可以看到,reflect.Value对应变量在内存中的数据，在GO中，任何变量定义后会根据其类型初始化为0，所以结构体的每个字段初始化成了相应类型的nil，由于string对应的”nil”是空字符串，因此打印时没有输出任何内容。同时我们看到通过reflect.Value对象的Type接口能获取对应的reflect.Type对象，通过Kind接口能获取reflect.Kind对象。

我们再看Type与Kind的区别，Type对应关键字type后面的定义，从例子中看就是main.Employee, Kind对应变量的在GO语言中的基础类型，在GO也就有那么几种基础类型，分别为int, uint, float, slice, map, string, bool, ptr , 本质上reflect.Kind可以看做是一个枚举类型的对象，通过它我们可以辨别对象的基础类型然后采取相应的操作。

reflect.Type的作用很大，它最重要的作用在于解析复合类型，例如解析struct类型里面的各个字段，解读slice类型每个元素的Type信息等，我们看看如何通过reflect.Type读取struct内部各个字段的信息，例子如下：

package main

import (
    "fmt"
    "reflect"
)

type MyData struct {
    Name   string `csv:"name"`
    HasPet bool   `csv:"has_pet"`
    Age    int    `csv:"age"`
    Salary float32
}

func main() {
    myData := MyData{
        Name:   "Smith",
        Age:    45,
        HasPet: false,
        Salary: 5000.0,
    }

    myDataValue := reflect.ValueOf(myData)
    myDataType := myDataValue.Type()

    for i := 0; i < myDataType.NumField(); i++ { //NumField获得结构体里面字段的数量
        structField := myDataType.Field(i)                    //Field用于获得字段的reflect.StructField对象
        fmt.Println("field name : ", structField.Name)        //reflect.StructField.Name获得字段名
        fmt.Println("field type : ", structField.Type.Name()) //reflect.StructField.Type.Name ()获得字段类型
        if tag, ok := structField.Tag.Lookup("csv"); ok {
            fmt.Println("field tag: ", tag)
        }

        fieldVal := myDataValue.Field(i) //reflect.Value同样支持Field接口
        switch fieldVal.Kind() {         //判断其基础类型
        case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
            fmt.Println("field with value: ", fieldVal.Int())
        case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
            fmt.Println("field with value: ", fieldVal.Uint())
        case reflect.String:
            fmt.Println("field  with value: ", fieldVal.String())
        case reflect.Bool:
            fmt.Println("field with value: ", fieldVal.Bool())
        case reflect.Float32, reflect.Float64:
            fmt.Println("field with value: ", fieldVal.Float())
        }
    }

}
上面代码运行后输出结果为：

field name :  Name
field type :  string
field tag:  name
field  with value:  Smith
field name :  HasPet
field type :  bool
field tag:  has_pet
field with value:  false
field name :  Age
field type :  int
field tag:  age
field with value:  45
field name :  Salary
field type :  float32
field with value:  5000
从代码可以看到，如果对象类型为结构体，那么可以调用reflect.Type对象的NumField接口获得结构体里面的字段数量，然后通过它的Field接口返回reflect.StructField类型，reflect.StructField.Name 对应字段的名称，reflect.StructField.Type.Name()返回字段的数据定义类型，同时reflect.StructField.Tag.LookUp可以查找字段是否含有给定标签，如果有的话还能返回标签对应的内容.

同时我们还能看到，reflect.Value对象也支持Field接口，它得到各个字段对应的reflect.Value对象，然后通过Kind接口返回信息来判断字段的基础类型，根据基础类型调用不同方法来获得对应数据，这里要注意如果基础类型是string, 但是你调用reflect.Value.Int()来获取数据，那么就有可能造成panic。同时调用reflect.Type.NumField接口时，必须确保对象是结构体类型，要不然调用该接口也会导致panic。

这里我们可以进一步分析Go的interface类型，interface其实包含了两部分，一部分是reflect.Type,一部分是reflect.Value，如果一个interface对象是nil的话，它必须满足reflect.Type对应nil, 同时reflect.Value.IsValid返回false,例如：

import (
    "fmt"
    "reflect"
)

func main() {
    var i interface{}
    if i == nil {
        fmt.Println("type for nil interface: ", reflect.TypeOf(i))
        fmt.Println("is value valid for nil interface: ", reflect.ValueOf(i).IsValid())
    }

    strPtr := (*string)(nil)
    i = (interface{})(strPtr) //此时i不再是nil,因为它有了reflect.Type内容
    fmt.Println("is interface i is nil: ", i == nil)
    fmt.Println("type for interface i is: ", reflect.TypeOf(i))
}
上面代码运行后返回结果为：

type for nil interface:  <nil>
is value valid for nil interface:  false
is interface i is nil:  false
type for interface i is:  *string
注意到虽然我们把一个空的字符串指针赋值给i，但此时i不再是空接口，因为它的reflect.Type部分有了内容，现在我们可以明白，为何interface类型能指向所有其他类型呢，原因正是我们这里解读的反射原理，通过它的reflect.Type部分获得它指向对象的类型，通过reflect.Value部分来读取对象的内容。

还有两种“过渡”类型需要我们注意，那就是指针和数组或者说是切片，之所以说是“过渡”是因为指针类型对应的Kind是reflect.Ptr，它还需要做进一步操作才能获得指针指向对象的具体类型。同时切片类型对应的Kind是reflect.Slice，我们还需要进一步操作才能获得切片元素的类型。如果对象是指针或者切片类型，那么reflect.Type.Elem()就能获得指向元素类型或者是获得切片所包含元素的类型，同时reflect.Value.Elem()才能获得指针指向对象的数值，对于切片类型，要获得其包含元素的数值还需要一些额外操作，我们看下面例子：

package main

import (
    "fmt"
    "reflect"
)

func main() {
    i := 1234
    i_ptr := &i
    i_ptr_type := reflect.TypeOf(i_ptr)
    i_ptr_value := reflect.ValueOf(i_ptr)
    i_ptr_kind := i_ptr_value.Kind()
    fmt.Println("i_ptr_type :", i_ptr_type)
    fmt.Println("i_ptr_value :", i_ptr_value)
    fmt.Println("i_ptr_kind: ", i_ptr_kind)
    fmt.Println("element type for element pointed to by i_ptr : ", i_ptr_type.Elem())
    fmt.Println("element value for pointer i_ptr: ", i_ptr_value.Elem())
}
上面代码运行后输出结果为：

i_ptr_type : *int
i_ptr_value : 0xc000018030
i_ptr_kind:  ptr
element type for element pointed to by i_ptr :  int
element value for pointer i_ptr:  1234
可以看到指针类型是 *int，指针对应的值就是变量在内存中的地址，通过Elem调用获得了指针指向元素的类型和数值，要注意如果元素的类型存在“过渡”属性，那么调用Elem就会导致panic，接下来我们看看切片的解析：

import (
    "fmt"
    "reflect"
)

func main() {
    str_slice := []string{"hello", "world"}
    str_slice_value := reflect.ValueOf(str_slice)
    str_slice_type := reflect.TypeOf(str_slice)
    elem_type := str_slice_type.Elem() //获得数组元素的类型信息
    fmt.Println("elem type :", elem_type)
    for i := 0; i < str_slice_value.Len(); i++ { //reflect.Value.Len() 获得数组元素个数
        elem_value := str_slice_value.Index(i) //通过Index获得指定元素
        fmt.Println("elem_value: ", elem_value)
    }
}
上面代码运行后结果如下：

elem type : string
elem_value:  hello
elem_value:  world
代码中需要注意的是，如果元素类型不是切片或数组，那么调用reflect.Value.Len接口会导致panic。最后我们看看GO内存分配，reflect.New可以针对指针类型分配内存，我们看下面代码示例：

import (
    "fmt"
    "reflect"
)

func main() {
    var iPtr *int
    fmt.Println("value of iPtr : ", reflect.ValueOf(iPtr))
    elem_type := reflect.TypeOf(iPtr).Elem() //获得指针指向的元素类型
    elem_ptr := reflect.New(elem_type)       //这里得到一个int类型指针对应的reflect.Value
    fmt.Println("elem ptr value: ", elem_ptr)
    elem_value := elem_ptr.Elem() //获得指针指向的元素对于的reflect.Value对象
    elem_value.SetInt(20)         //调用reflect.Value 对应接口设置元素数值
    fmt.Println("elem value: ", elem_value)
}
上面代码运行后结果为：

value of iPtr :  <nil>
elem ptr value:  0xc000018040
elem value:  20
这里需要注意的是reflect.New返回的是指针类型对象对应的reflect.Value，由于它具有“过渡”属性，因此我们可以调用Elem获得指针指向对象的reflect.Value对象，然后调用相应的Set接口来设置对象的数值，如果对象类型是int那么就实现SetInt，如果是string，那就使用SetString,但如果对象类型是int,但使用SetString就会导致panic。

反射由于涉及到编译原理等因素，如果没有相应代码示例来辅助理解，那么我们学起来会感觉很抽象和烧脑，希望上面代码示例能帮助同学们对GO的反射机制有较好的理解。

