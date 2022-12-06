---
layout: section
---

# Expronicon
## 基于[MLStyle](https://github.com/thautwarm/MLStyle.jl)的元编程工具箱

---
layout: section
---

# 什么是元编程
## Meta Programming

---
layout: quote
---

# “程序即数据”

将程序作为数据处理的函数就是元编程里宏（macro）

---
layout: two-cols
---

## 需求

（`Base.@kwdef`) 从以下语法表达式作为输入

```julia
struct MyKwType
    a::Int = 1
    b::Vector{Int} = 2
end
```

产生

```julia
struct MyKwType
    a::Int
    b::Vector{Int}
end

function MyKwType(;a::Int = 1, b::Vector{Int} = 2)
    MyKwType(a, b)
end
```

::right::

## 思路

<v-click>

- 扫描表达式，分离结构体表达式和各个元素的赋值表达式

</v-click>

<v-click>

- 分析结构体表达式的类型定义信息（类型参数，子类型关系）

</v-click>


<v-click>

- 将结构体表达式复制进新的表达式

</v-click>

<v-click>

- 分析结构体表达式，构造关键字构造函数

</v-click>

<v-click>

- 将以上产生的表达式按照顺序组合在一起

</v-click>


---
layout: section
---

# Expronicon的功能概览

---
layout: statement
---

# 方便的
# 表达式操作

---
layout: two-cols
---

## 函数中间格式

```julia
julia> JLFunction(
           head=:function,
           name=:(object::MyObject),
           args=[:x, :(y::Int)],
           body=quote
               x + y
           end
       )
function object::MyObject(x, y::Int)
    #= REPL[19]:6 =#
    x + y
end
```

::right::

## 结构体中间格式

```julia
julia> JLStruct(
           name=:MyType,
           supertype=:MySuperType,
           fields=[
               JLField(name=:x, type=:Int),
               JLField(name=:y, type=:T),
           ],
           doc="a demo type",
       )
"""
a demo type
"""
struct MyType <: MySuperType
    x::Int
    y::T
end
JLKwStruct
```

---
layout: two-cols
---

## 条件分支（ifelse）中间格式

```julia
julia> jl = JLIfElse(ex);


julia> JLIfElse(ex)
if x > 100
    x + 1
elseif x > 90
    x + 2
elseif x > 80
    x + 3
else
    error("some error msg")
end
```

::right::

```julia
julia> jl.otherwise
quote
    #= /Users/roger/Code/Julia/Expronicon/test/print/multi.jl:243 =#
    error("some error msg")
end

julia> jl[:(x > 100)]
quote
    #= /Users/roger/Code/Julia/Expronicon/test/print/multi.jl:237 =#
    x + 1
end
```

---
layout: section
---

# 海量表达式分析工具

---
layout: two-cols
---

# 常用表达式分析工具

- 超过20种表达式分析函数
- 详细的文档，几乎每个函数都有对应的用例
- 直观且统一的接口
- 经过实践检验

::right::

```julia
compare_expr
is_function
is_kw_function
is_struct
is_tuple
is_splat
is_ifelse
is_for
is_field
is_field_default
is_datatype_expr
is_matrix_expr
split_function
split_function_head
split_struct
split_struct_name
split_ifelse
uninferrable_typevars
has_symbol
is_literal
is_gensym
alias_gensym
has_kwfn_constructor
has_plain_constructor
guess_type
```

---
layout: two-cols
---

## 解析函数表达式

```julia
julia> ex = :(function f(x::T; a=10)::Int where T
           return x
       end)
:(function (f(x::T; a = 10)::Int) where T
      return x
  end)

julia> jl = JLFunction(ex)
function f(x::T; a = 10)::Int where {T}
    return x
end

julia> jl.head, jl.doc, jl.name, jl.rettype, jl.whereparams
(:function, nothing, :f, :Int, Any[:T])
```

::right::

## 解析结构类型定义表达式

```julia
julia> ex = :(struct Foo1{N, T}
               x::T = 1
           end)
:(struct Foo1{N, T}
      x::T = 1
  end)

julia> JLKwStruct(ex)
begin
    struct Foo1{N, T}
        x::T
    end
    begin
        function Foo1{N, T}(; x = 1) where {N, T}
            Foo1{N, T}(x)
        end
        function Foo1{N}(; x = 1) where {N}
            Foo1{N}(x)
        end
    end
    nothing
end
```

---
layout: two-cols
---

## Julia表达式反射

```julia
julia> @expr if x > 100
           x + 1
       elseif x > 90
           x + 2
       elseif x > 80
           x + 3
       else
           error("some error msg")
       end
:(if x > 100
      #= REPL[1]:2 =#
      x + 1
  elseif #= REPL[1]:3 =# x > 90
      #= REPL[1]:4 =#
      x + 2
  elseif #= REPL[1]:5 =# x > 80
      #= REPL[1]:6 =#
      x + 3
  else
      #= REPL[1]:8 =#
      error("some error msg")
  end)
```


::right::

## 中间格式反射

```julia
julia> def = @expr JLKwStruct struct Foo1{N, T}
            x::T = 1
        end
begin
    struct Foo1{N, T}
        x::T
    end
    begin
        function Foo1{N, T}(; x = 1) where {N, T}
            Foo1{N, T}(x)
        end
        function Foo1{N}(; x = 1) where {N}
            Foo1{N}(x)
        end
    end
    nothing
end
```

---
layout: two-cols
---

# 常用表达式构造工具

构造对应表达式的`x`函数，帮助减少
表达式构造时容易出现的常见错误

```julia
xtuple(1, :x)
# :((1, x))
xnamedtuple(;x=2, y=3)
# :((x = 2, y = 3))
xcall(Base, :sin, 1; x=2)
# :($Base.sin(1; x = 2))
xpush(:coll, :x)
# :($Base.push!(coll, x))
xfirst(:coll)
# :($Base.first(coll))
xlast(:coll)
# :($Base.last(coll))
xprint(:coll)
# :($Base.print(coll))
```

等20种构造工具

::right::

# 方便的构造器

```julia {all|3|4|6|7|15-22|all}
julia> using Expronicon

julia> jl = JLIfElse()
nothing

julia> jl[:(foo(x))] = :(x = 1 + 1)
:(x = 1 + 1)

julia> jl[:(goo(x))] = :(y = 1 + 2)
:(y = 1 + 2)

julia> jl.otherwise = :(error("abc"))
:(error("abc"))

julia> jl
if foo(x)
    x = 1 + 1
elseif goo(x)
    y = 1 + 2
else
    error("abc")
end
```

---
layout: two-cols
---

## 测试表达式并不方便

```julia
julia> lhs = Expr(:block, :(1 + 1))
quote
    1 + 1
end

julia> rhs = quote 1 + 1 end
quote
    #= REPL[15]:1 =#
    1 + 1
end

julia> lhs == rhs
false
```

::right::

<v-click>

## 表达式测试工具

大部分时候我们只关心 **语义** 上表达式是否相等

```julia
julia> using Test

julia> @test_expr lhs == rhs
Test Passed
```

</v-click>

<v-click>

`@test_expr` 忽略不影响语义的表达式，然后进行比较，
毋需浪费资源对表达式求值就可以进行测试。

</v-click>

---

## 总结

- 方便操作的表达式中间格式
- 常用表达式分析工具
- 常用表达式构造工具
- 表达式测试工具

<v-click>

实战检验： Configurations， Comonicon

</v-click>
