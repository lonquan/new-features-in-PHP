# PHP 新特性汇总

按版本倒序整理 PHP 的主要新特性、常用 API，以及升级时需要关注的弃用和不兼容变更。新版本附有示例与迁移建议，旧版本保留简明索引。

## 目录

- PHP 8.x：[8.6](#php-86) · [8.5](#php-85) · [8.4](#php-84) · [8.3](#php-83) · [8.2](#php-82) · [8.1](#php-81) · [8.0](#php-80)
- PHP 7.x：[7.4](#php-74) · [7.2](#php-72) · [7.1](#php-71) · [7.0](#php-70)

## PHP 8.6

发布日期：2026 年 11 月 19 日。

官方资料：[发布说明](https://www.php.net/releases/8.6/en.php) · [新特性](https://www.php.net/manual/en/migration86.new-features.php) · [不兼容变更](https://www.php.net/manual/en/migration86.incompatible.php) · [弃用清单](https://www.php.net/manual/en/migration86.deprecated.php)

### 主要特性

#### 部分函数应用（Partial Function Application）

使用 `?` 预留一个参数，或使用 `...` 预留剩余参数，即可创建接收缺失参数的闭包。原函数要等到闭包被调用时才执行。

```php
<?php

function add(int $a, int $b, int $c = 0): int
{
    return $a + $b + $c;
}

$addOne = add(1, ?);
var_dump($addOne(5)); // int(6)

$addToOne = add(1, ...);
var_dump($addToOne(2, 3)); // int(6)

$versions = ['8.4', '8.5', '8.6', '8.7', '9.0'];
$supported = array_filter(
    $versions,
    version_compare(?, '8.6', '>='),
);
```

#### `Time\Duration`：精确表示时长

`Duration` 表示与日期和时区无关的固定时长，支持纳秒精度、比较和算术运算，适合表达超时、重试间隔和基准测试耗时。它不包含 `DateInterval` 中天、月等长度不固定的日历单位。

```php
<?php

use Time\Duration;

$timeout = Duration::fromSeconds(2)
    ->add(Duration::fromMilliseconds(500));

printf("%d.%09d seconds\n", $timeout->seconds, $timeout->nanoseconds);
// 2.500000000 seconds

var_dump($timeout > Duration::fromSeconds(2)); // bool(true)
```

#### `clamp()`：将值限制在范围内

值超出范围时返回最近的边界值，也可用于其他可比较的值。

```php
<?php

var_dump(clamp(-10, 0, 100)); // int(0)
var_dump(clamp(50, 0, 100));  // int(50)
var_dump(clamp(110, 0, 100)); // int(100)
```

#### `SortDirection`：统一的排序方向枚举

内置枚举提供 `Ascending` 和 `Descending` 两个 case，让库之间可以用类型安全的方式共享排序方向，而不必各自约定整数、字符串或布尔值。

```php
<?php

function orderBy(
    string $column,
    SortDirection $direction = SortDirection::Ascending,
): string {
    $order = match ($direction) {
        SortDirection::Ascending => 'ASC',
        SortDirection::Descending => 'DESC',
    };

    return $column . ' ' . $order;
}

echo orderBy('name', SortDirection::Descending); // name DESC
```

#### Readonly 属性支持默认值

实例 readonly 属性（包括 readonly 类中的属性）现在可以声明默认值。默认值会完成属性初始化，因此之后不能再次赋值，构造函数也不例外。

```php
<?php

class Config
{
    public readonly int $timeout = 30;
}

$config = new Config();
var_dump($config->timeout); // int(30)
```

#### 结构化流错误处理（Stream Error Handling）

流操作可以配置为抛出 `StreamException` 或静默失败，而不再只能依赖警告。结构化 `StreamError` 包含错误码、包装器名称及相关路径或 URL。

```php
<?php

$context = stream_context_create([
    'stream' => [
        'error_mode' => StreamErrorMode::Exception,
    ],
]);

try {
    $config = file_get_contents('/etc/app/config.json', context: $context);
} catch (StreamException $e) {
    foreach ($e->getErrors() as $error) {
        printf("%s: %s\n", $error->code->name, $error->message);
    }
}
```

也可通过 `stream_last_errors()` 读取错误，通过 `stream_clear_errors()` 清除错误。

#### Polling API：可扩展的 I/O 多路复用

`Io\Poll` 根据平台使用 epoll、kqueue、event ports 或 WSAPoll 等机制，避免 `stream_select()` 的连接数扩展限制，主要为 ReactPHP、AMPHP 等事件循环库提供统一基础。

```php
<?php

use Io\Poll\{Context, Event};

$poll = new Context();
$poll->add(new StreamPollHandle($stream), [Event::Read]);

foreach ($poll->wait() as $watcher) {
    $stream = $watcher->getHandle()->getStream();
    // 处理就绪的流
}
```

#### 参数 DocComment 与属性读写反射

- 函数和方法参数可以拥有独立 DocComment，并通过 `ReflectionParameter::getDocComment()` 获取。
- `ReflectionProperty::isReadable()` 和 `ReflectionProperty::isWritable()` 会结合调用作用域、对象状态、属性 Hook、可见性及魔术方法判断属性能否读写。

```php
<?php

class DB
{
    public function search(
        /** 搜索关键词 */
        string $query,
        /** 最大结果数 */
        int $limit = 10,
    ): array {
        return [];
    }
}

$parameter = new ReflectionParameter([DB::class, 'search'], 'query');
var_dump($parameter->getDocComment());
```

### 新增函数与扩展能力

| 分类 | 代表 API 或能力 | 用途 |
| --- | --- | --- |
| URI | `Uri\Rfc3986\UriBuilder`、`Uri\WhatWg\UrlBuilder` | 从组件构建 URI；另新增 URI/主机类型检测方法 |
| URI | `Uri\WhatWg\url_percent_encode()` | 按 WHATWG 规则进行 URL 百分号编码 |
| Streams | `stream_last_errors()`、`stream_clear_errors()` | 获取或清除结构化流错误 |
| Intl | `grapheme_strrev()` | 按字素簇反转字符串，避免拆散组合字符 |
| Intl | `IntlNumberRangeFormatter` | 按区域和 skeleton 格式化数字区间 |
| Zip | `ZipArchive::openString()`、`ZipArchive::closeString()` | 直接处理内存中的 ZIP 压缩包 |
| MySQLi | `mysqli::quote_string()` | 转义字符串并添加引号，行为类似 `PDO::quote()` |
| GMP | `gmp_powm_sec()`、`gmp_prevprime()` | 侧信道静默模幂运算、查找前一个素数；依赖相应 GMP 版本 |
| SNMP | `snmp_init_mib()` 及 MIB/输出控制 API | 重置 MIB 树并控制解析和输出 |

### 其他值得关注的改进

- `#[\Override]` 现在可用于类常量和枚举 case；父类或接口中不存在对应常量时会产生编译错误。
- 枚举可以实现 `__debugInfo()`；常量或类常量中保存的对象现在可以直接修改属性。
- `WeakMap` 支持在键尚不存在时进行引用赋值。
- `pack()` 和 `unpack()` 的整数、浮点格式码支持 `<`（小端）和 `>`（大端）修饰符。
- Sodium 增加后量子密钥封装、IP 地址加密及 SHAKE/TurboSHAKE；TLS 流支持会话恢复、外部 PSK 和 TLS 1.3 early data（0-RTT）。
- JSON 错误消息包含输入中的错误位置；`error_include_args` INI 设置控制错误消息是否一致地包含函数参数。
- 性能改进包括将特定 `array_map()` 回调编译为循环、加速 `printf()`、缓存无状态闭包及加快 JSON 编码。

### 弃用与迁移建议

下表列出常用或影响较大的项目，完整清单见官方[弃用说明](https://www.php.net/manual/en/migration86.deprecated.php)。

| 已弃用用法 | 推荐替代或处理 |
| --- | --- |
| 在 `finally` 块中使用 `return` | 将 `return` 移到 `finally` 外，避免覆盖返回值或吞掉异常 |
| 从 `__construct()` / `__destruct()` 返回值，或将其声明为生成器 | 不再返回有意义的值；无值的 `return;` 仍允许 |
| `__debugInfo(): ?array` | 改为 `__debugInfo(): array`；PHP 8.5 已弃用返回 `null` |
| 类常量名 `namespace`、函数名 `readonly`，以及使用 `let` / `is` 命名类、函数或常量 | 重命名；`_` 作为常量名也已弃用，但 `_()` 函数不受影响 |
| `define()` 的第三个参数 | 省略该参数；大小写不敏感常量已在 PHP 8.0 移除 |
| 字符串类名传给 `is_a()` / `is_subclass_of()`，同时 `$allow_string = false` | 先按需要检查 `class_exists()`，再显式允许字符串，避免依赖始终返回 `false` 的调用 |
| `is_double()`、`is_long()`、`is_integer()`、`doubleval()` | 分别改用 `is_float()`、`is_int()`、`is_int()`、`floatval()` |
| `mb_ereg()`、`mb_eregi*()`、`mb_split()` 等 mbregex API | 通常改用带 `u` 修饰符的 PCRE 函数；这些 API 将在 PHP 9.0 移除 |
| `strcoll()` 和 `SORT_LOCALE_STRING` | 使用 intl 扩展的 `Collator::compare()`、`sort()`、`asort()` 等 |
| 将对象传给 `array_walk()`、`array_walk_recursive()`、`http_build_query()`、`mb_convert_variables()` 或压缩过滤器 | 先用 `get_object_vars()` 显式转换，必要时递归处理 |
| `spl_object_hash()`、`spl_classes()` | 分别改用 `spl_object_id()`、`ReflectionExtension::getClassNames()` |
| 自定义 Session Handler 未同时实现 `create_sid()` 和 `validateId()` | 实现 `SessionIdInterface` 与 `SessionUpdateTimestampHandlerInterface` 要求的这两个方法 |
| `mysqli_get_charset()` / `mysqli::get_charset()` | 获取字符集名称时改用 `mysqli_character_set_name()` / `mysqli::character_set_name()` |
| `mysqli_stmt_init()` / `mysqli::stmt_init()`，以及不传查询构造 statement | 使用 `mysqli_prepare()` / `mysqli::prepare()`，或直接传入查询 |
| `ArrayIterator` 的排序、flag 和序列化方法 | 需要这些能力时使用 `ArrayObject`；对应 `ArrayObject` 方法未弃用 |
| `SplFileObject` 的 CSV 方法 | 对流使用 `fgetcsv()` 和 `fputcsv()` |
| `metaphone()` | 使用适合目标语言的 phonetic algorithm 用户态库 |
| 未配置上限时在 `php://filter` URL 中使用超过 16 个过滤器 | 优先缩短过滤链，或使用 `stream_filter_append()` / `stream_filter_prepend()`；确有需要时配置 `filter.max_filter_count` |

### 升级注意事项

这些属于行为变化，不是弃用提示：

| 变化 | 升级影响 |
| --- | --- |
| Session 默认值更安全 | `session.use_strict_mode=1`、`session.cookie_httponly=1`、`session.cookie_samesite=Lax`；依赖 JavaScript 读取会话 Cookie 或跨站 POST 携带会话的应用需显式调整配置 |
| `trim()`、`ltrim()`、`rtrim()` 默认去除 `\f` | 换页符现在与其他默认空白字符一样会被移除 |
| 流关闭或刷新失败会反馈失败 | `fclose()`、`file_put_contents()`、`copy()` 在相应失败时会返回 `false`，调用方应检查返回值 |
| 非法参数校验更严格 | 多个函数对越界值、非法 flag 或含 NUL 字节的字符串改为抛出 `ValueError`，不再静默截断、钳制或忽略 |

## PHP 8.5

### 主要特性

#### 管道操作符（Pipe Operator）

从左到右链式调用可调用项，让数据在多个函数间传递，无需中间变量。

```php
<?php

$result = 'Hello World'
    |> 'strtolower'
    |> 'str_split'
    |> (fn($x) => array_map('strtoupper', $x))
    |> (fn($x) => array_filter($x, fn($v) => $v !== 'O'));
```

#### Clone With：克隆并更新属性

在克隆对象的同时更新属性，简化 readonly 类的 with-er 模式。

```php
<?php

readonly class Point
{
    public function __construct(
        public int $x,
        public int $y,
    ) {
    }

    public function withX(int $x): static
    {
        return clone($this, ['x' => $x]);
    }
}

$point = new Point(x: 1, y: 2);
$newPoint = $point->withX(5);

// $newPoint->x === 5，$newPoint->y === 2
```

#### `#[\NoDiscard]` 属性

当函数或方法的返回值未被使用时发出警告，避免调用方意外忽略重要结果。

```php
<?php

#[\NoDiscard]
function calculateImportantValue(): int
{
    return 42;
}

calculateImportantValue();
// Warning: Return value of calculateImportantValue() must be used
```

#### 常量表达式中的闭包和 First-class Callable

静态闭包和 First-class Callable 现在可用于常量表达式，例如属性参数。

```php
<?php

#[Attribute]
class MyAttribute
{
    public function __construct(
        public Closure $callback = static function (): string {
            return 'default';
        },
    ) {
    }
}
```

#### URI 扩展

新增内置 URI 扩展，支持按照 RFC 3986 和 WHATWG URL 标准解析、规范化和处理 URL。

```php
<?php

$uri = Uri\WhatWg\Url::parse('https://example.com/path?query=value#fragment');
echo $uri?->getAsciiHost(); // example.com
echo $uri?->getPath();      // /path
```

#### 持久化 cURL Share 句柄

句柄可以在多个 PHP 请求之间保持，避免重复初始化到同一主机的连接。

### 新增函数

| 函数 | 说明 |
| --- | --- |
| `array_first()` | 返回数组的第一个值；数组为空时返回 `null`，方便与 `??` 组合 |
| `array_last()` | 返回数组的最后一个值；数组为空时返回 `null` |

```php
<?php

$array = [1, 2, 3, 4];
$first = array_first($array); // 1
$last = array_last($array);   // 4

$value = array_first([]) ?? 'default'; // 'default'
```

### 弃用与迁移建议

| 已弃用用法 | 推荐替代或处理 |
| --- | --- |
| `switch` 语句中 `case` 后的分号 | 使用冒号：`case 1:` |
| 反引号作为 `shell_exec()` 的别名 | 显式调用 `shell_exec()` |
| `__sleep()` 和 `__wakeup()` | 使用 `__serialize()` 和 `__unserialize()` |
| `(boolean)`、`(integer)`、`(double)`、`(binary)` | 分别使用 `(bool)`、`(int)`、`(float)`、`(string)` |
| 将 `null` 用作数组偏移量 | 根据意图显式转换为键，避免依赖隐式转换 |
| `__debugInfo()` 返回 `null` | 返回数组 |
| 重定义常量 | 确保每个常量只定义一次 |
| `Reflection*::setAccessible()` | PHP 8.1 起属性、方法默认可通过反射访问，移除多余调用 |
| `curl_close()`、`imagedestroy()`、`finfo_close()`、`xml_parser_free()` | 移除不再必要的显式释放调用 |
| `mysqli_execute()` | 使用对应的 statement 方法 |
| `$http_response_header` | 使用 `http_get_last_response_headers()` |

### 移除与不兼容变更

- `disable_classes` INI 选项已移除，因为它会破坏引擎的一些基本假设。
- `class_alias()` 不再允许将 `array` 和 `callable` 用作类别名。

## PHP 8.4

### 主要特性

#### 属性 Hook（Property Hooks）

为属性附加 getter 和 setter，也可以定义不直接对应存储值的虚拟属性。

```php
<?php

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
```

#### 不对称属性可见性（Asymmetric Property Visibility）

分别控制属性的读取和写入可见性。

```php
<?php

class Example
{
    public protected(set) string $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }
}
```

#### 惰性对象（Lazy Objects）

将对象的实际初始化延迟到访问时，可创建 lazy ghost 或 lazy proxy。

```php
<?php

class Example
{
    public function __construct(private int $data)
    {
    }
}

$initializer = static function (Example $ghost): void {
    // 获取数据或依赖后完成初始化
    $data = 42;
    $ghost->__construct($data);
};

$reflector = new ReflectionClass(Example::class);
$object = $reflector->newLazyGhost($initializer);
```

相关 API 包括：

- `ReflectionClass::newLazyGhost()`、`newLazyProxy()`、`resetAsLazyGhost()`、`resetAsLazyProxy()`。
- `ReflectionClass::isUninitializedLazyObject()`、`initializeLazyObject()`、`markLazyObjectAsInitialized()`、`getLazyInitializer()`。
- `ReflectionProperty::skipLazyInitialization()`、`setRawValueWithoutLazyInitialization()`。
- `ReflectionClass::SKIP_INITIALIZATION_ON_SERIALIZE`、`SKIP_DESTRUCTOR`。

#### 其他核心特性

- `#[\Deprecated]` 属性可将函数、方法和类常量标记为弃用，并在使用时触发 `E_USER_DEPRECATED`。
- `request_parse_body()` 可在非 POST HTTP 请求中解析 RFC 1867（multipart）请求体。
- `new` 表达式后可以直接进行方法调用或属性访问，不再需要用括号包裹：

```php
<?php

new Hello()->world();
```

### 新增函数与方法

| 分类 | API | 说明 |
| --- | --- | --- |
| BCMath | `bcceil()`、`bcfloor()` | 向上、向下取整 |
| BCMath | `bcdivmod()` | 返回商和余数 |
| BCMath | `bcround()` | 四舍五入 |
| DateTime | `createFromTimestamp()` | 根据 Unix 时间戳创建对象 |
| DateTime | `getMicrosecond()`、`setMicrosecond()` | 获取、设置微秒；同时适用于 `DateTime` 和 `DateTimeImmutable` |
| MBString | `mb_trim()`、`mb_ltrim()`、`mb_rtrim()` | 多字节安全的空白裁剪 |
| MBString | `mb_ucfirst()`、`mb_lcfirst()` | 转换首字母大小写 |
| XML | `XMLReader::fromStream()`、`fromUri()`、`fromString()` | 从不同来源创建读取器 |
| XML | `XMLWriter::toStream()`、`toUri()`、`toMemory()` | 向不同目标创建写入器 |
| HTTP | `http_get_last_response_headers()` | 获取最近一次 HTTP 响应头 |
| HTTP | `http_clear_last_response_headers()` | 清除最近一次 HTTP 响应头 |
| Math | `fpow()` | 计算 x 的 y 次幂 |
| Array | `array_all()` | 检查所有值是否都通过回调测试 |
| Array | `array_any()` | 检查是否至少一个值通过回调测试 |
| Array | `array_find()` | 返回通过回调测试的第一个元素 |
| Array | `array_find_key()` | 返回通过回调测试的第一个元素的键 |

### 弃用与迁移建议

隐式可空参数已弃用。类型声明应明确包含 `null`：

```php
<?php

// PHP 8.4 起弃用
function foo(T2 $value = null) {}

// 推荐：两种显式可空写法均可
function fooShort(?T2 $value = null) {}
function fooUnion(T2|null $value = null) {}
```

## PHP 8.3

### 主要特性

1. **类常量支持类型声明**

   ```php
   <?php

   interface I
   {
       const string PHP = 'PHP 8.3';
   }

   class Foo implements I
   {
       const string PHP = [];
   }

   // Fatal error: Cannot use array as value for class constant
   // Foo::PHP of type string
   ```

2. **类常量支持动态获取**

   ```php
   <?php

   class Foo
   {
       const PHP = 'PHP 8.3';
   }

   $searchableConstant = 'PHP';
   var_dump(Foo::{$searchableConstant});
   ```

3. **`#[\Override]` 属性**

   向方法添加该属性后，PHP 会确保父类或实现的接口中存在同名方法，明确表达重写父方法的意图。

4. **Readonly amendments：克隆时重新初始化 readonly 属性**

   ```php
   <?php

   class PHP
   {
       public string $version = '8.2';
   }

   readonly class Foo
   {
       public function __construct(public PHP $php)
       {
       }

       public function __clone(): void
       {
           $this->php = clone $this->php;
       }
   }

   $instance = new Foo(new PHP());
   $cloned = clone $instance;
   $cloned->php->version = '8.3';
   ```

5. **`json_validate()`**

   ```php
   <?php

   var_dump(json_validate('{ "test": { "foo": "bar" } }')); // true
   ```

6. **`Randomizer::getBytesFromString()`**：根据给定字符集生成随机字节序列，适合随机字符串、域名和数字字符串。
7. **`Randomizer::getFloat()` 和 `nextFloat()`**：以无偏方式生成随机浮点数。
8. **命令行 linter 支持多个文件名**：一次执行即可检查多个 PHP 文件。
9. **其他新增方法**：`posix_sysconf()`、`mb_str_pad()`、`str_increment()`、`str_decrement()`、`stream_context_set_options()`。
10. **匿名类可以声明为 readonly**。

## PHP 8.2

### 主要特性

1. 类可以声明为 `readonly`。
2. 支持析取范式（DNF）类型，可组合 union 和 intersection 类型；交集类型必须用括号分组：

   ```php
   <?php

   class Foo
   {
       public function bar((A&B)|null $entity)
       {
           return $entity;
       }
   }
   ```

3. 允许 `null`、`false` 和 `true` 作为独立类型。
4. 新增随机扩展 `Randomizer`。
5. Trait 中可声明常量，并在使用 Trait 的类中访问。
6. 新增 `memory_reset_peak_usage()`。

### 弃用与迁移建议

动态属性已弃用，以减少错误和拼写问题：

```php
<?php

class User
{
    public $name;
}

$user = new User();
$user->last_name = 'Doe'; // Deprecated

$user = new stdClass();
$user->last_name = 'Doe'; // 仍然允许
```

类可以使用 `#[\AllowDynamicProperties]` 显式允许动态属性；`stdClass` 不受影响，`__get()` / `__set()` 魔术方法的使用也不受影响。

此外，`${}` 字符串插值、`utf8_encode()` 和 `utf8_decode()` 已弃用。

## PHP 8.1

### 主要特性

1. 新增 `array_is_list()`，判断数组是否为从 0 开始连续编号的列表。
2. 参数解包后可以使用命名参数：

   ```php
   <?php

   foo(...$args, named: $arg);
   ```

3. 新增枚举类型 `enum`：

   ```php
   <?php

   enum Suit
   {
       case Hearts;
       case Diamonds;
       case Clubs;
       case Spades;
   }

   $suit = Suit::Hearts;
   ```

4. 新增 readonly 属性：`public readonly string $prop`。
5. 类常量支持 `final`：`final public const X = 'foo'`。
6. `new ClassName()` 可用于参数默认值、静态变量、全局常量初始值设定项、属性参数和 `define()`。
7. 新增 First-class Callable 语法：`CallableExpr(...)`。

## PHP 8.0

### 主要特性

1. **命名参数**：`myFunction(paramName: $value)`，传参可以不按声明顺序。
2. **构造器属性提升**：`public function __construct(protected int $x, protected int $y = 0) {}`。
3. **联合类型**：例如 `T1|T2|null`。
4. **`match` 表达式**：

   ```php
   <?php

   $expressionResult = match ($condition) {
       1, 2 => foo(),
       3, 4 => bar(),
       default => baz(),
   };
   ```

   没有匹配项且未提供 `default` 时会抛出 `UnhandledMatchError`。
5. **Nullsafe 运算符**：使用 `?->` 安全地进行方法和属性访问。

## PHP 7.4

### 主要特性

1. 属性支持限定类型。
2. 箭头函数：`array_map(fn($n) => $n * $factor, [1, 2, 3, 4])`。
3. 支持有限的返回类型协变与参数类型逆变。
4. 空合并赋值运算符：`$array['key'] ??= computeDefault()`。
5. 数组内支持展开操作：`...$array`。

## PHP 7.2

### 主要特性

1. 新增 `object` 类型，可用于逆变参数和协变返回值。
2. 允许重写抽象方法（abstract method）。
3. Argon2 加入密码散列（password hashing）支持。
4. 允许分组命名空间声明使用尾部逗号。

## PHP 7.1

### 主要特性

1. 可空（nullable）类型：`?string`。
2. `void` 返回类型。
3. 短数组语法 `[]` 可用于解构：`[$id1, $name1] = $data`。
4. 类常量支持可见性。
5. 新增 `iterable` 伪类型。
6. 多异常捕获：`catch (FirstException | SecondException $e)`。
7. `list()` 和 `[]` 解构支持指定键名，例如 `list('id' => $id1, 'name' => $name1) = $data`。
8. 支持负字符串偏移量。

## PHP 7.0

### 主要特性

1. 标量类型声明。
2. 返回值类型声明。
3. null 合并运算符 `??`。
4. 太空船操作符 `<=>`。
5. 可通过 `define()` 定义常量数组。
6. 匿名类：`new class`。
