# sail 内置类型与用户自定义类型
## 一些基本类型
| 类型 | sail中的含义 | 代码示例 |
| --- | --- | --- |
| unit | 在函数声明中表示无返回值()  | function main():unit->unit={}  |
| bool | true/false  | let a:bool=true; |
| int | 表示任意精度整数  | let a:int=1; |
| int('n) | 'n是sail中的类型约束 | let number:int(1)=1; |
| nat | 任意精度自然数 | let number: nat =0; |
| range('m,'n) | 闭区间'm和'n之间的数字 | let r: range(1,3)=2; |
| {n1,n2,...} | 一个数字集合，变量值只能是集合中的某个元素 | let s: {32,64}=64; |
| bits('n) | 位向量，'n限制了位向量的长度 | let v:bits(8)=0b00000001; |
| vector(n,type) | 向量：n为元素个数，type为元素类型，例如int | let v:vector(3,int)=[1,2,3]; |
| list | 列表 | 两种写法：//    let l: list(int)=[|1,2,3|];let l: list(int)=1::2::3::[||]; |
| tuple | 元祖,必须有两个或更多元素，每个元素的类型可以不同 | let tuple1 : (string, int) = ("Hello, World!", 3) |
| string | 字符串,包裹在两个双引号之间的字符序列 | let s:string="hello world!"; |
## basic.sail 代码示例
以下是一段可直接编译运行的代码，其中包含了上面提到的sail内置类型:
```sail

// =========================================
// basic.sail
// Sail 基础语法演示
// =========================================

default Order dec
$include <prelude.sail>

function main(): unit -> unit = {

    // =====================================
    // 1. 基本类型：布尔与整数
    // =====================================

    let b : bool = true;

    // Sail 有三种基本数字类型：int, nat, range()
    let number1 : int        = 1;
    let number2 : nat        = 0;
    let number3 : range(1,3) = 2;

    // number4 被限制为只能等于集合中的某个元素
    let number4 : {32, 64} = 64;

    print_int("number1 = ", number1);
    print_int("number2 = ", number2);
    print_int("number3 = ", number3);
    print_int("number4 = ", number4);

    // =====================================
    // 2. 比特向量（bits）
    // =====================================

    let bv1 : bits(8) = 0b00000001;

    // bitzero 和 bitone 代表 0 和 1
    let bv2 : bits(2) = [bitzero, bitone];

    // 使用 @ 运算符连接两个 bitvector
    let bv3 : bits(16) = 0xFFFF;
    let bv4 : bits(16) = 0x0000;
    let bv5 = bv3 @ bv4;   // 拼接结果为 32 位 bitvector

    print_bits("bv1 = ", bv1);
    print_bits("bv2 = ", bv2);
    print_bits("bv3 = ", bv3);
    print_bits("bv4 = ", bv4);
    print_bits("bv5 = ", bv5);

    // =====================================
    // 3. 向量（vector）
    // =====================================

    let v1 : vector(3, int) = [1, 2, 3];

    // 由于 order 是 dec，因此索引 0 的元素是 3
    let v_element1 : int = v1[0];

print_int("v_element1 = ", v_element1);

    // =====================================
    // 4. 列表（list）
    // =====================================

    // 两种创建列表的方式
    let l1 : list(int) = [|1, 2, 3|];
    let l2 : list(int) = 1 :: 2 :: 3 :: [||];


    // =====================================
    // 5. 元组（tuple）与字符串
    // =====================================

    let tuple1 : (string, int) = ("Hello, World!", 3);

    //解包元祖
    let (str,number)=tuple1;
    print_int("tuple[1]]=",number);
    //字符串
    let s : string = "hello world!\n";

    print(s);
    print("hello sail!\n");

}
```
## 用户自定义类型
sail中提供了三种用户自定义类型，union，struct,enum.  
### Struct
与c语言相同，struct用于保存多个值，使用struct关键字创建struct,后面跟struct的名称。存储在struct中的每条数据称为一个字段，每个字段有自己唯一的名称，且每个字段可以是不同的类型。以下示例结构定义了三个字段。第一个包含长度为5的位向量称为field1。第二个字段field2包含一个整数。第三个字段field3包含一个字符串。  
```sail
struct My_struct = {
  field1 : bits(5),
  field2 : int,
  field3 : string,
}
```
现在，我们可以使用struct关键字并为所有字段提供值的方式创建一个My_struct的对象。同时，sail也支持使用 . 运算符访问struct中的字段。另外，如想修改一个不可修改的struct对象(用let关键字定义的对象)，可以用with关键字，它会基于旧的struct对象创建一个新的struct对象。就像这样:  
```sail
    let s : My_struct = struct {
        field1 = 0b11111,
        field2 = 5,
        field3 = "test",
    };

    // 访问结构体字段, 打印 "field1 is 0b11111"
    print_bits("field1 is ", s.field1);

    // 使用with更新struct的字段，创建新的结构体变量s2
    var s2 = { s with field1 = 0b00000, field3 = "a string" };

    // 打印 "field1 is 0b00000"
    print_bits("field1 is ", s2.field1);
    // 打印 "field2 is 5", 因为field2取自's',且未被修改
    print_int("field2 is ", s2.field2);

    //  因为s2是可变的，所以我们可以给它赋值
    s2.field3 = "some string";
    // 打印 "some string"
    print_endline(s2.field3);

```
struct中的字段类型也可以是用户自定义类型:  
```sail

// struct可以具有用户定义的类型参数。以下示例有一个整数参数“n”和一个类型参数“a”。
struct My_other_struct('n, 'a) = {
    a_bitvector : bits('n),
    something : 'a
}

    var s3 : My_other_struct(2, string) = struct {
        a_bitvector = 0b00,
        something = "a string"
    };

    var s4 : My_other_struct(32, My_struct) = struct {
        a_bitvector = 0xFFFF_FFFF,
        something = struct {
            field1 = 0b11111,
            field2 = 6,
            field3 = "nested structs!",
        }
    };

    s3.a_bitvector = 0b11;

    // 请注意，一旦创建，我们就不能更改变量的类型，因此禁止以下操作：
    //s3.a_bitvector =0xFFFF_FFFF;

    // 在with表达式中也禁止更改类型：
    //let s4 : My_other_struct(32, string) = { s3 with a_bitvector = 0xFFFF_FFFF};

    //  如果结构是嵌套的，那么字段访问也可以嵌套
    print_endline(s4.something.field3);
    // 分配值
    s4.something.field3 = "another string";

```

### Enum
sail中的enum也与c中的类似。使用enum关键字创建枚举，后面跟枚举的名称，然后是花括号包裹的以逗号分隔的枚举值。示例：  
```sail
enum My_enum = {
  Foo,
  Bar,
  Baz,
  Quux,
}
```
还有一种简洁的定义enum的方式,类似Haskell：  
```sail
enum My_short_enum = A | B | C
```
Sail将自动生成用于将枚举成员转换为数字的函数，枚举的第一个元素从0开始。  
```sail
function enum_example1() = {
  assert(num_of_My_enum(Foo) == 0);
  assert(num_of_My_enum(Bar) == 1);
  assert(num_of_My_enum(Baz) == 2);
  assert(num_of_My_enum(Quux) == 3);
}
```
### Union
struct给我们提供了一种将多个数据组合起来的方式，但有时候我们会面临另一种情况，例如描述一个形状，其在某个时刻可能是举行也可能是圆形，这时就用到了Union.示例：  
```sail

struct rectangle = {
    width : int,
    height : int,
}
struct circle = { radius : int }
union Shape = {
    Rectangle : rectangle,
    Circle : circle,
}

function example() = {
    // 使用Rectangle构造函数构造Shape
    let r : Shape = Rectangle(struct { width = 30, height = 20 });

    // 使用Circle构造函数构造Shape
    // 我们允许推断Shaple类型
    let c = Circle(struct { radius = 15 });

    mat(r);
}
```
访问union的值，需要用到模式匹配：  

```sail

function mat(shape: Shape):Shape -> unit={
    match(shape){
        Circle(ci) => print_int("radius = ", ci.radius),
        Rectangle(re) =>{
            print_int("width=",re.width);
            print_int("height=",re.height);
        }
    }
}

```
##总结
