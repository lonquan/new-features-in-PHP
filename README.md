# new-features-in-PHP
PHP 近期版本的新特性汇总


## PHP 8.5


### **New Features**
```php
<?php
/*
* Pipe Operator / 管道操作符
* 从左到右链式调用可调用项, 使数在多个函数间顺畅传递, 无需中间变量
*/
$result = "Hello World"
    |> 'strtolower'
    |> 'str_split'
    |> fn($x) => array_map('strtoupper', $x)
    |> fn($x) => array_filter($x, fn($v) => $v !== 'O');

/*
* Clone With / 克隆并更新属性
* 在克隆对象的同时更新属性, 简化了 readonly 类的 "with-er" 模式
*/
readonly class Point
{
    public function __construct(
        public int $x,
        public int $y,
    ) {}
}

$point = new Point(x: 1, y: 2);
$newPoint = clone($point) with {
    x: 5,
};
// $newPoint->x === 5, $newPoint->y === 2

/*
* #[\NoDiscard] 属性
* 当函数/方法的返回值未被使用时发出警告, 避免错误, 提高 API 的安全性
*/
#[\NoDiscard]
function calculateImportantValue(): int
{
    return 42;
}

calculateImportantValue(); // Warning: Return value of calculateImportantValue() must be used

/*
* 常量表达式中的闭包和 First-class 可调用
* 静态闭包和 First-class 可调用现在可以用于常量表达式, 例如属性参数
*/
#[Attribute]
class MyAttribute
{
    public function __construct(
        public Closure $callback = static fn() => 'default',
    ) {}
}

/*
* URI 扩展
* 新增内置的 URI 扩展, 支持按照 RFC 3986 和 WHATWG URL 标准解析、规范化和处理 URL
*/
$uri = Uri\Url::fromString('https://example.com/path?query=value#fragment');
echo $uri->getHost(); // example.com
echo $uri->getPath(); // /path

/*
* 持久化 cURL Share 句柄
* 句柄现在可以在多个 PHP 请求之间保持, 避免重复初始化到同一主机的连接
*/
```

### New Functions
```php
<?php

// 返回数组的第一个值, 若数组为空则返回 null (方便与 ?? 操作符组合)
array_first()

// 返回数组的最后一个值, 若数组为空则返回 null (方便与 ?? 操作符组合)
array_last()

$array = [1, 2, 3, 4];
$first = array_first($array); // 1
$last = array_last($array); // 4

$empty = [];
$value = array_first($empty) ?? 'default'; // 'default'
```

### 弃用

```php
<?php
/*
* 弃用 switch 语句中 case 后的分号
* 建议使用冒号
*/
// Deprecated
switch ($x) {
    case 1;
        break;
}
// Recommended
switch ($x) {
    case 1:
        break;
}

/*
* 弃用反引号作为 shell_exec 的别名
* 建议使用 shell_exec() 函数
*/
// Deprecated
`ls -la`;
// Recommended
shell_exec('ls -la');

/*
* 弃用 __sleep() 和 __wakeup() 魔术方法
* 建议使用 __serialize() 和 __unserialize() 方法
*/

/*
* 弃用非标准强制转换名称
* (boolean)、(integer)、(double) 和 (binary) 已弃用
* 建议使用 (bool)、(int)、(float) 和 (string)
*/
// Deprecated
$value = (boolean) $var;
$num = (integer) $str;
$float = (double) $val;
$binary = (binary) $data;

// Recommended
$value = (bool) $var;
$num = (int) $str;
$float = (float) $val;
$binary = (string) $data;

/*
* disable_classes INI 选项已被移除
* 该选项会破坏引擎的一些基本假设
*/

/*
* class_alias() 中不再允许将 "array" 和 "callable" 用作类别名
*/
// No longer allowed
class_alias(MyClass::class, 'array'); // Error
class_alias(MyClass::class, 'callable'); // Error

/*
* 弃用将 null 作为数组偏移量
* 弃用 __debugInfo() 返回 null
* 弃用常量重定义
* 弃用 Reflection*::setAccessible()
* 弃用 curl_close() / imagedestroy() / finfo_close() / xml_parser_free()
* 弃用 mysqli_execute()
* 弃用 $http_response_header 预定义变量
*/
```


## PHP 8.4


### **New Features**
```php
<?php
/*
* Property Hooks / 属性 Hook
* 附加 getter 和 setter 到属性, 该属性可以是未声明的
*/
class Person
{
    public string $fullName {
        get => $this->firstName . ' ' . $this->lastName;
    }

    public string $firstName {
        set => ucfirst(strtolower($value));
    }

    public string $lastName {
        set {
            if (strlen($value) < 2) {
                throw new \InvalidArgumentException('Too short');
            }
            $this->lastName = $value;
        }
    }
}

/*
* Asymmetric Property Visibility / 不对称的属性可见性
* 将对象属性的 set 可见性和 get 可见性分开控制
*/
class Example
{
    public protected(set) string $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }
}

/**
* Lazy Objects / 惰性对象
* 将对象构造方法延迟到访问时
*/
class Example
{
    public function __construct(private int $data)
    {
    }

    // ...
}

$initializer = static function (Example $ghost): void {
    // Fetch data or dependencies
    $data = ...;
    // Initialize
    $ghost->__construct($data);
};

$reflector = new ReflectionClass(Example::class);
$object = $reflector->newLazyGhost($initializer);
// 相关方式和常量
// ReflectionClass::newLazyGhost()
// ReflectionClass::newLazyProxy()
// ReflectionClass::resetAsLazyGhost()
// ReflectionClass::resetAsLazyProxy()
// ReflectionClass::isUninitializedLazyObject()
// ReflectionClass::initializeLazyObject()
// ReflectionClass::markLazyObjectAsInitialized()
// ReflectionClass::getLazyInitializer()
// ReflectionProperty::skipLazyInitialization()
// ReflectionProperty::setRawValueWithoutLazyInitialization()
// ReflectionClass::SKIP_INITIALIZATION_ON_SERIALIZE
// ReflectionClass::SKIP_DESTRUCTOR

/*
* #[\Deprecated] attribute 
* 新的 Deprecated 属性可用于将函数、方法和类常量标记为已弃用, throw E_USER_DEPRECATED error
*/

/* 
* Added request_parse_body()
* 该函数允许在非 POST HTTP 请求中解析 RFC1867（multipart）请求
*/

/*
* new 表达式链式调用不再需要括号
* 🎉 new class 后可以直接链接方法调用、属性访问, 无需在使用括号包裹 🎉
*/
new Hello->world()
```

### New Functions
```php
<?php

/*
* BCMath
*/ 
bcceil() // 向上取整
bcdivmod() // 求商
bcfloor() // 向下取整
bcround() // 四舍五入

/*
* BCMath
*/ 
DateTime::createFromTimestamp()
DateTime::getMicrosecond()
DateTime::setMicrosecond()
DateTimeImmutable::createFromTimestamp()
DateTimeImmutable::getMicrosecond()
DateTimeImmutable::setMicrosecond()

/*
* MBString
*/
mb_trim()
mb_ltrim()
mb_rtrim()
mb_ucfirst()
mb_lcfirst()

/*
* XML
*/
XMLReader::fromStream()
XMLReader::fromUri()
XMLReader::fromString()
XMLWriter::toStream()
XMLWriter::toUri()
XMLWriter::toMemory()

// 获取最近一次 HTTP 响应的头部信息
http_get_last_response_headers()
  
// 清除最近一次 HTTP 响应的头部信息
http_clear_last_response_headers()

// 计算 x 的 y 次幂  
fpow()
  
// 检查数组中的值是否都能通过 callback 的测试
array_all()

// 检查数组中是否存在至少一个元素满足 callbacl 的测试
array_any()

// 返回满足 callback 测试的第一个元素
array_find()
  
// 返回满足 callback 测试的第一个元素的 key
array_find_key()
```
### 弃用

```php
<?php
/*
* Implicitly nullable parameter
* 隐式的可空参数
*/

// after
function foo(T1 $a, T2 $b = null, T3 $c) {}
// before 
function foo(T1 $a, T2|null $b, T3 $c) {}
// or
function foo(T1 $a, ?T2 $b, T3 $c) {}
```



## PHP 8.3
1. 类常量可声明类型

```php
interface I {
    const string PHP = 'PHP 8.3';
}

class Foo implements I {
    const string PHP = [];
}

// Fatal error: Cannot use array as value for class constant
// Foo::PHP of type string
    ```

2. 类常量可动态获取
```php
lass Foo {
    const PHP = 'PHP 8.3';
}

$searchableConstant = 'PHP';

var_dump(Foo::{$searchableConstant});
```


3. #[\Override] 属性
通过向方法添加 #[\Override] 属性，PHP 将确保父类或实现的接口中存在同名方法。添加该属性可以清楚地表明重写父方法是有意为之


4. Readonly amendments 深度克隆
```php
class PHP {
    public string $version = '8.2';
}

readonly class Foo {
    public function __construct(
        public PHP $php
    ) {}

    public function __clone(): void {
        $this->php = clone $this->php;
    }
}

$instance = new Foo(new PHP());
$cloned = clone $instance;

$cloned->php->version = '8.3';
```

5. json_validate()
```html
var_dump(json_validate('{ "test": { "foo": "bar" } }')); // true
```

6. Randomizer::getBytesFromString()
该方法允许开发人员轻松生成随机字符串，例如域名和任意长度的数字字符串。


7. Randomizer::getFloat() & nextFloat()
以无偏的方式生成随机浮点数


8. 命令行 linter 现在接受多个文件名进行 lint
9. 新增方法 posix_sysconf / mb_str_pad / str_increment / str_decrement / stream_context_set_options
10. 匿名类现在可以是只读的

## PHP 8.2

1. 可以把一个类声明为 readonly
2. 析取范式 （DNF）类型
```php
class Foo {
    public function bar((A&B)|null $entity) {
        return $entity;
    }
}
// DNF 类型允许我们组合 union 和 intersection类型，
// 遵循一个严格规则：组合并集和交集类型时，交集类型必须用括号进行分组。
```

3. 允许 null、false 和 true 作为独立类型
4. 新的随机扩展 Randomizer 
5. Traits 中可声明常量并在使用 trait 的类中进行访问
6. 弃用动态属性
```php
class User
{
    public $name;
}

$user = new User();
$user->last_name = 'Doe'; // Deprecated notice

$user = new stdClass();
$user->last_name = 'Doe'; // Still allowed

// 动态属性的创建已被弃用，以帮助避免错误和拼写错误，
// 除非该类通过使用 #[\AllowDynamicProperties] 属性来选择。stdClass 允许动态属性。
// __get/__set 魔术方法的使用不受此更改的影响。
```

7. 新增方法 memory_reset_peak_usage 
8. 弃用 ${} 字符串插值, 弃用 utf8_encode 和 utf8_decode 函数

## PHP 8.1

1. 函数新增  
- array_is_list() 是否是从 0 开始的索引数组
2. 参数解包后的命名参数
```php
 foo(...$args,named: $arg)
```

3. 增加枚举类型 enum
```php
enum Suit
{ 
  case Hearts; case Diamonds; case Clubs; case Spades; 
}
Suit::Hearts
```

4. readonly 只读属性关键字 public readonly string $prop;
5. 类中常量 final  关键字 final public const X = "foo";
6. new ClassName() 表达式用作参数的默认值、静态变量、全局常量初始值设定项和属性参数和 define()
7. First class callable syntax   CallableExpr(...)


----

## PHP 8.0

1. 命名参数 myFunction(paramName: $value) 传参可以不按照参数顺序
2. 构造器属性提升 public function __construct(protected int $x, protected int $y = 0) {}
3. 联合类型 T1|T2|null
4. match 语法
```php
$expressionResult = match ($condition) {
    1, 2 => foo(),
    3, 4 => bar(),
    default => baz(),
};
not match UnhandledMatchError
```

5. Nullsafe 方法和属性


----

## PHP 7.4

1. 属性添加限定类型
2. 箭头函数 array_map(fn($n) => $n * $factor, [1, 2, 3, 4])
3. 有限返回类型协变与参数类型逆变(class extends 后改变方法返回类型)
4. 空合并运算符赋值 $array['key'] ??= computeDefault();
5. 数组展开操作 ...$array


----

## PHP 7.2

1. 新的对象类型, object, 可用于逆变（contravariant）参数输入和协变（covariant）返回任何对象类型
2. 允许重写抽象方法(Abstract method)
3. Argon2 已经被加入到密码散列（password hashing)
4. 允许分组命名空间的尾部逗号


----

## PHP 7.1

1. 可为空（Nullable）类型 ?string
2. Void 函数
3. 短数组语法（[]） [$id1, $name1] = $data
4. 类常量可见性
5. iterable 伪类
6. 多异常捕获处理 catch (FirstException | SecondException $e)
7. list() 和它的新的 [] 语法支持在它内部去指定键名 list("id" => $id1, "name" => $name1) = $data[0]
8. 为负的字符串偏移量


----

## PHP 7.0

1. 标量类型声明
2. 返回值类型声明
3. null合并运算符 ??
4. 太空船操作符
5. 通过 define 定义常量数组
6. 匿名类 new class


----
