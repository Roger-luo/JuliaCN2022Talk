# Configurations

配置文件（configuration）和数据模式（schema）工具箱

## 功能

- 定义一个数据模式（schema）或者配置文件
- 解析数据模式
- 利用类型转换来验证数据模式
- 从数据模式到代码生成

---
layout: two-cols
---

## 定义一个数据模式（schema）或者配置文件

我们这里以TOML为例（Julia标准库自带了一个解析器），便携一个
关于Julia编译器选项的配置文件，尖括号 `<>` 里表示对应参数的
类型和名称。

```toml
[julia]
compile="<yes|no>"
threads="<nthreads::Int>"
optimize="<level::Int>"
startup="<file::String>"
```

<v-click>

为了方便处理并检测输入是否正确，我们往往会定义其**对应的类型**
从而能够在语言里表示这样一个配置文件

```julia
struct JuliaOptions
    compile::String # yes or no
    threads::Int
    optimize::Int
    startup::String
end
```

</v-click>

::right::

<v-click>

接下来我们将需要把从解析器里的输出结果转换到我们的
结构体 `JuliaOptions` 并且检查输入的值是否符合
要求，例如 `compile` 的值只能是 `"yes"` 或者 `"no"`,

```julia
function from_toml(compile, threads, optimize, startup)
    compile in ("yes", "no") || error("expect yes or no for compile")
    JuliaOptions(compile, threads, optimize, startup)
end
```

并且我们需要实现一个关键字函数来支持有默认参数的项。

```julia
function from_toml(;compile="yes", threads=2, optimize=3, startup="no")
    compile in ("yes", "no") || error("expect yes or no for compile")
    JuliaOptions(compile, threads, optimize, startup)
end
```

</v-click>

---
layout: two-cols
---

更进一步，由于我们的工程有大量的配置信息，并且有一些工程
会**共享相同的配置信息**，我们需要定义有更加复杂结构的
配置文件，并且让不同的模块能够**互相组合**

```julia
struct LargeOptions
    # ...
    julia::JuliaOptions
    # ...
end
```

::right::

<v-click>

现在以上步骤以及更多的功能只需要

```julia
using Configurations
@option struct JuliaOptions
    compile::String = "yes"
    threads::Int = 1
    optimize::Int = 3
    startup::String = "no"
end

@option struct LargeOptions
    # ...
    julia::JuliaOptions
    # ...
end
```

即可解锁

</v-click>

---
layout: two-cols
---

## 利用类型转换来验证数据模式

由于Julia有强类型，我们可以利用类型来实现对值的验证

```julia
julia> @option struct MyOption
           a::Int
           b::Symbol
       end


julia> d = Dict{String, Any}(
           "a" => 1,
           "b" => "ccc"
       )
Dict{String, Any} with 2 entries:
  "b" => "ccc"
  "a" => 1
```

::right::

我们可以为特定的参数定义其类型转换

```julia
julia> from_dict(MyOption, d)
ERROR: FieldTypeConversionError: conversion from String to type Symbol for field b in type Main.MyOption failed

julia> function Configurations.from_dict(::Type{MyOption}, ::OptionField{:b}, ::Type{Symbol}, s)
            s isa String && s == "ccc" || error("expect ccc")
            return Symbol(s)
        end
```

---
layout: two-cols
---

## 可组合的配置类型

`@option` 在定义相关数据模式解析方法和类型的时候也会定义一系列
操作接口使得这一系列类型变得可以组合。

```julia
"Option A"
@option "option_a" struct OptionA
    name::String
    int::Int = 1
end

"Option B"
@option "option_b" struct OptionB
    opt::OptionA = OptionA(; name="Sam")
    float::Float64 = 0.3
end

@option struct OptionC
    num::Float64

    function OptionC(num::Float64)
        num > 0 || error("not positive")
        return new(num)
    end
end
```

::right::

<v-click>

## 方法一览

我们的接口非常之简单，以下是操作一个 `@option` 标记的类型所需要知道的所有方法

```julia
@option
field_default
type_alias
is_option
from_dict
from_kwargs
from_toml
from_toml_if_exists
to_dict
```

他们都有详细的文档说明


</v-click>

---

一些使用Configurations的工程

<img width=200 src="https://raw.githubusercontent.com/fonsp/Pluto.jl/dd0ead4caa2d29a3a2cfa1196d31e3114782d363/frontend/img/logo_white_contour.svg"/>

|                   |                 |
| ----------------- | --------------- |
| TerminalClock     | QuantumESPRESSO |
| Workflows         | Polymer         |
| Bloqade           | Workflows       |
| PlutoSliderServer | Comonicon       |
