## VPPINFRA(基础设施)
VPP基础设施层的源文件位于./src/vppinfra目录。

VPPinfra是一系列基本的c库集合，足以构建可直接在裸机上运行的独立程序。它还提供高性能的动态数组(dynamic arrays)，哈希(hashes)，位图(bitmaps)，高精度实时时钟的支持，细粒度的事件记录(event-logging)和数据结构序列化(data structure serialization)。

关于vppinfra的一个合理的评论/合理的警告：您不能总是仅通过名称来告诉普通函数中的内联函数中的宏。在典型情况下，宏用于避免函数调用，并引起(有意的)副作用。

Vppinfra已经存在了将近20年，并且往往不会经常更改。VPP基础结构层包含以下功能：

### Vectors
Vppinfra Vectors是带有用户定义头部的动态调整大小的数组。许多vpppinfra数据结构(例如哈希(hash)，堆(heap)，池(pool))是具有各种不同头部的Vector。

内存布局是这样的:
```
                   User header (optional, uword aligned)
                   Alignment padding (if needed)
                   Vector length in elements
 User's pointer -> Vector element 0
                   Vector element 1
                   ...
                   Vector element N-1
```

如上所示，vector向量API处理指向vector第0个元素的指针。空指针是长度为零的有效向量。

为了避免破坏内存分配器，通常在保留内存分配的同时将向量的长度重置为零。通过vec_reset_length(v)宏将向量长度字段设置为零。[请使用宏！它比NULL指针更聪明。]

通常，用户头部不存在。用户头部允许将其他数据结构构建在vppinfra vector之上。用户可以通过[vec]()*_aligned宏指定向量的第一个数据元素的对齐方式。

向量元素可以是任何C类型，例如(int，double和struct bar)。对于建立在向量之上的数据类型(例如堆heap，池pool等)也是如此。许多宏具有支持向量元素对齐的_a变体和支持非零长度向量头的_h变体，_ha变体两者都支持。此外，可以使用[CLIB_CACHE_LINE_ALIGN_MARK]()宏指定vector元素结构内的缓存行对齐。

头部和/或对齐方式相关的宏变体用法不一致会导致延迟，和混乱的故障。

标准编程错误：记住一个指向向量ith元素的指针，然后展开向量。向量扩展了3/2，因此此类代码似乎可以工作一段时间。正确的代码几乎总是记住向量索引，这些向量索引在重新分配之间是不变的。

在典型的应用程序镜像中，提供了一组全局函数，这些函数旨在从gdb调用。这里有一些例子：

vl(v)-打印vec_len(v)

pe(p)-打印pool_elts(p)

pifi(p，index)-打印pool_is_free_index(p，index)

debug_hex_bytes(p，nbytes)-从p开始的十六进制内存dump nbytes

使用“show gdb”调试CLI命令来打印当前设置。

### Bitmaps
Vppinfra Bitmaps是动态的，是使用vppinfra Vector API构建的。非常适合各种工作。

### Pools
Vppinfra pools结合了vectors和bitmaps，可以快速分配和释放具有独立生存期的固定大小的数据结构。Pools非常适合分配每个会话结构。

### Hashes
Vppinfra提供了几种哈希样式。涉及数据包分类/会话查找的数据平面问题经常使用./src/vppinfra/bihash_template.[ch]有界索引可扩展哈希(bihash, bounded-index extensible hashes)。这些模板被实例化多次，以有效地服务于不同的固定键大小。

Bishhes是线程安全的。不需要读锁定。一个简单的自旋锁可确保一次仅一个线程写入一个条目。

原始的vppinfra哈希实现很容易使用(./src/vppinfra/hash.[ch])，并且经常用在需要精确字符串匹配(extract-string-matching)的控制平面代码中。

无论哪种情况，几乎总是在哈希表中查找键，以获得相关vector或pool中的索引。这些API非常简单，但是在使用不受管理的任意大小的键变量时，一定要小心。 Hash_set_mem(哈希表hash_table，键指针key_pointer，值value)存储key_pointer。将vector元素的地址作为第二个参数传递给hash_set_mem通常是一个严重的错误。最好在代码段(text segment)中存储常量字符串地址。

### 格式化输出(Format)
Vppinfra format大致等同于printf。

Format具有一些值得一提的属性。Format的第一个参数是(u8 *)vector，它在其中附加了当前format操作的结果。链式调用非常简单：

```
u8 * result;

result = format (0, "junk = %d, ", junk);
result = format (result, "more junk = %d\n", more_junk);
```

如前所述，NULL指针是完全正确的0长度vector。格式返回(u8 *)vector，而不是C字符串。如果要打印(u8 *)vector，请使用“%v”格式字符串。如果您需要一个(u8 *)vector，它也是一个适当的C字符串，则可以使用以下两种方案之一：

```
vec_add1 (result, 0)
or
result = format (result, "<whatever>%c", 0);
```

请记住在适当的时候对结果进行vec_free()。注意不要传递未初始化的格式(u8 *)。

format通过“%U”格式实现了一种用户自定义的格式化方案。例如：

```
u8 * format_junk (u8 * s, va_list *va)
{
    junk = va_arg (va, u32);
    s = format (s, "%s", junk);
    return s;
}

result = format (0, "junk = %U, format_junk, "This is some junk");
```

如果需要，format_junk()可以调用其他用户自定义的格式化函数。程序员负责参数类型检查。如果va_arg(va，type)宏与调用者的现实想法不符，通常用户格式函数会急剧膨胀。

### 格式化输入(Unformat)
Vppinfra unformat与scanf相当，但更为笼统。

典型的用法涉及从C字符串或(u8 *)向量初始化unformat_input_t，然后通过unformat()进行解析，如下所示：
```
    unformat_input_t input;

    unformat_init_string (&input, "<some-C-string>");
    /* or */
    unformat_init_vector (&input, <u8-vector>);
```

然后循环解析单个元素:

```
    while (unformat_check_input (&input) != UNFORMAT_END_OF_INPUT)
    {
      if (unformat (&input, "value1 %d", &value1))
        ;/* unformat sets value1 */
      else if (unformat (&input, "value2 %d", &value2)
        ;/* unformat sets value2 */
      else
        return clib_error_return (0, "unknown input '%U'",
                                  format_unformat_error, input);
    }
```
与格式化一样，unformat通过“%U”实现了用户自定义unformat的方案。通常，人们可以轻松地转换“format(s，“foo%d”，foo)->“unformat(input，“foo%d”，&foo)”。

Unformat实现了一些方便的非类似于scanf的格式说明符：
```
    unformat (input, "enable %=", &enable, 1 /* defaults to 1 */);
    unformat (input, "bitzero %|", &mask, (1<<0));
    unformat (input, "bitone %|", &mask, (1<<1));
    <etc>
```

如果unformat本身全部解析“enable”关键字，则短语“enable%=”表示“将提供的变量设置为默认值”。如果unformat解析“enable 123”，则将提供的变量设置为123。

我们可以使用“%=”清除一些手动输入的“冗长”+“冗长%d”参数解析代码。

如果unformat解析“bitzero”，则短语“bitzero%|”表示“设置提供的位掩码中的指定位”。尽管看起来很方便，但是在代码库中很少使用。

#### 如何解析单个输入行
调试(Debug) CLI命令功能一定不能意外消费属于其他调试CLI命令的输入。否则，将无法编写一组调试CLI命令的脚本，这些命令一次发出一个就可以“正常工作”。

此代码段不正确：
```
/* Eats script input NOT beloging to it, and chokes! */
while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
{
    if (unformat (input, ...))
    ;
    else if (unformat (input, ...))
    ;
    else
    return clib_error_return (0, "parse error: '%U'",
                            format_unformat_error, input);
    }
}
```

当作为脚本的一部分执行时，此类函数每次都会返回“parse error：‘’”，除非它恰好是脚本中的最后一个命令。

相反，请使用“unformat_line_input”消费一行中其余的输入内容-超出VLIB_CLI_COMMAND声明中指定的路径的所有内容。

例如，如下所示设置unformat_line_input为“my_command”，那么用户输入“my path is clear”将生成一个包含“is clear”的unformat_input_t。

```
VLIB_CLI_COMMAND (...) = {
    .path = "my path",
};
```

这是一些代码，完整显示了所需的机制：

```
static clib_error_t *
my_command_fn (vlib_main_t * vm,
                unformat_input_t * input,
                vlib_cli_command_t * cmd)
{
    unformat_input_t _line_input, *line_input = &_line_input;
    u32 this, that;
    clib_error_t *error = 0;

    if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

    /*
    * Here, UNFORMAT_END_OF_INPUT is at the end of the line we consumed,
    * not at the end of the script...
    */
    while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
    {
        if (unformat (line_input, "this %u", &this))
            ;
        else if (unformat (line_input, "that %u", &that))
            ;
        else
            {
            error = clib_error_return (0, "parse error: '%U'",
                                format_unformat_error, line_input);
            goto done;
            }
        }

<do something based on "this" and "that", etc>

done:
    unformat_free (line_input);
    return error;
}
/* *INDENT-OFF* */
VLIB_CLI_COMMAND (my_command, static) = {
    .path = "my path",
    .function = my_command_fn",
};
/* *INDENT-ON* */
```

### Vppinfra错误和警告
vpp数据平面内的许多函数的返回值类型为clib_error_t *。 Clib_error_t是任意字符串，带有一些元数据(致命fatal，警告warning)，并且易于声明。返回NULL clib_error_t *表示“A-OK，没有错误。”

Clib_warning(format-args)是添加调试输出的便捷方法。clib警告优先于function:line info以明确定位消息源。 Clib_unix_warning()添加了perror()样式的Linux系统调用信息。在生产镜像中，clib_warnings会生成syslog条目。

### 序列化(Serialization)
Vppinfra序列化支持使程序员可以轻松地序列化和反序列化复杂的数据结构。

基本的原始序列化/反序列化函数使用网络字节顺序，因此在小端字节序的主机上序列化和在大端字节序的主机上反序列化没有结构性问题。