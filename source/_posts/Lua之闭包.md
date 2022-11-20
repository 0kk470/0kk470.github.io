---
title: Lua之闭包
date: 2022-11-20 11:53:53
tags: [Lua, Closure]
categories: Lua
---

# 前言

复习下Lua闭包相关的知识

# 正文

## 定义

* 闭包
闭包（closure）是一个函数以及其捆绑的周边环境状态（lexical environment，词法环境）的引用的组合。换而言之，闭包让开发者可以从内部函数访问外部函数的作用域。
以下为一个Lua闭包。
```Lua
local function func1()
    local x = 1
    local function func2()
        x = x + 1
        return x
    end
    return func2
end
```
* First-class Function

当一门编程语言的函数可以被当作变量一样用时，则称这门语言拥有头等函数， 这也是Lua通过闭包来支持的特性。
比如上面的函数也可以用以下形式来定义。
```Lua
local func1 = function ()
    local x = 1
    return function ()
        x = x + 1
        return x
    end
end
```

## 原理

### 闭包的结构

在Lua中，所有的函数其实都是一个闭包，如下图所示，GC代表的垃圾回收相关的数据，这里不赘述。

![closure](closure.png)

Lua的闭包为扁平闭包(```flat closure```)，它只保存它所需要的环境中的变量的引用，而不是保存整个环境的引用。
扁平闭包与“safe-for-space complexity”规则兼容，即“在某个局部变量的作用域内最后一次使用该变量后，该变量的绑定就不能被访问”。

定义代码如下

```C
typedef struct CClosure {
  ClosureHeader;
  lua_CFunction f;
  TValue upvalue[1];  /* list of upvalues */
} CClosure;

typedef struct LClosure {
  ClosureHeader;
  struct Proto *p;
  UpVal *upvals[1];  /* list of upvalues */
} LClosure;

typedef union Closure {
  CClosure c;
  LClosure l;
} Closure;

```
代码中的```CClosure```和```LClosure```分别代表C闭包和Lua闭包。
其中```LClosure```的```Proto```为函数原型，包含了函数定义需要的所有信息，结构如下

```C
typedef struct Proto {
  CommonHeader;
  lu_byte numparams;  /* number of fixed (named) parameters */
  lu_byte is_vararg;  
  lu_byte maxstacksize;  /* number of registers needed by this function */
  int sizeupvalues;  /* size of 'upvalues' */
  int sizek;  /* size of 'k' */
  int sizecode;
  int sizelineinfo;
  int sizep;  /* size of 'p' */
  int sizelocvars;
  int sizeabslineinfo;  /* size of 'abslineinfo' */
  int linedefined;  /* debug information  */
  int lastlinedefined;  /* debug information  */
  TValue *k;  /* constants used by the function */
  Instruction *code;  /* opcodes */
  struct Proto **p;  /* functions defined inside the function */
  Upvaldesc *upvalues;  /* upvalue information */
  ls_byte *lineinfo;  /* information about source lines (debug information) */
  AbsLineInfo *abslineinfo;  /* idem */
  LocVar *locvars;  /* information about local variables (debug information) */
  TString  *source;  /* used for debug information */
  GCObject *gclist;
} Proto;
```
一些未注释或者不太好理解的变量意义如下:
* numparams代表固定参数数量。
* is_varargb 标识是否有可变参数
* maxstacksize 是该函数需要的寄存器数量(Lua虚拟机是基于寄存器执行的),每个函数最多分配```250```个寄存器。比如下面的函数执行时, ```maxstacksize```为7。

![register](register.png)

* sizeupvalues， 函数闭包的上值数量。
* sizek, 函数内部的常量数量。
* sizecode, 执行的指令数量
* sizelineinfo, 函数行数信息。
* sizep, 内部定义的函数数量，即指针```**p```指向的函数列表大小。
* sizelocvars, 局部变量数量。
* 指针```**p```为所有定义在该函数内部的函数原型列表，通过该指针，Lua可以用递归的方式向上查找某个函数的外部函数变量上值。

构建Lua闭包的代码如下
```C
static void pushclosure (lua_State *L, Proto *p, UpVal **encup, StkId base,
                         StkId ra) {
  int nup = p->sizeupvalues;
  Upvaldesc *uv = p->upvalues;
  int i;
  LClosure *ncl = luaF_newLclosure(L, nup);
  ncl->p = p;
  setclLvalue2s(L, ra, ncl);  /* anchor new closure in stack */
  for (i = 0; i < nup; i++) {  /* fill in its upvalues */
    if (uv[i].instack)  /* upvalue refers to local variable? */
      ncl->upvals[i] = luaF_findupval(L, base + uv[i].idx);
    else  /* get upvalue from enclosing function */
      ncl->upvals[i] = encup[uv[i].idx];
    luaC_objbarrier(L, ncl, ncl->upvals[i]);
  }
}
```

#### 上值Upvalue
Lua闭包实现的核心构成部分是```upvalue```结构，它表示了一个闭包和一个变量的连接，结构如下。

```C
typedef struct UpVal {
  CommonHeader;
  lu_byte tbc;  /* true if it represents a to-be-closed variable */
  TValue *v;  /* points to stack or to its own value */
  union {
    struct {  /* (when open) */
      struct UpVal *next;  /* linked list */
      struct UpVal **previous;
    } open;
    TValue value;  /* the value (when closed) */
  } u;
} UpVal;
```

上值```UpValue```有两种状态:open和closed，如下图所示。

![upvalue_structure](upvalue_structure.png)

当一个upvalue创建时，它处于open状态，这时候通过指针```*v```指向栈上的值，当upvalue被关闭时, 值会被拷贝到```value```字段中, 然后指针会指向为这个```value```。
以下列函数为例:
```Lua
function add3 (x) -- f1
    return function (y) -- f2
      return function (z) return x + y + z end -- f3
    end
end
print(add3(1)(2)(3)) --打印 6
```
内层函数f3需要3个变量：```x```、```y```和```z```。其中```y```为```f3```的上值, 当执行到```f3```内部时, 变量```x```可能已经销毁了，为了避免这种情况，```f2```的闭包会将```x```存为```upvalue```。因此```x + y + z```执行时，实际上是f3的两个```upvalue```与```z```计算。等同于下面这个函数定义。
```Lua
function add3 (x)
  return function (y)
    local dummy = x
    return function (z) return dummy + y + z end
  end
end
```
在上述描述的机制下，会有个问题：当```f3```闭包创建时，会将```f2```的上值拷贝过去，这样相当于```f3```多存了一份数据。

为了解决一个问题，```UpVal```结构中引入了一个```open```链表。该链表中upvalue顺序与栈中对应变量的顺序相同。当解释器需要一个变量的upvalue时，它首先遍历这个链表：如果找到变量对应的upvalue，则复用它，因此确保了共享，否则它会创建一个新的upvalue并将其插入到链表中正确的位置。

查找与新建上值得代码如下
```C
UpVal *luaF_findupval (lua_State *L, StkId level) {
  UpVal **pp = &L->openupval;
  UpVal *p;
  lua_assert(isintwups(L) || L->openupval == NULL);
  while ((p = *pp) != NULL && uplevel(p) >= level) {  /* search for it */
    lua_assert(!isdead(G(L), p));
    if (uplevel(p) == level)  /* corresponding upvalue? */
      return p;  /* return it */
    pp = &p->u.open.next;
  }
  /* not found: create a new upvalue after 'pp' */
  return newupval(L, 0, level, pp);
}find
```

构建一个新的```open upvalue```之后的栈结构如下。
![open_upvalues](open_upvalues.png)

当一个变量离开作用域不再被引用时，它链接的upvalue会尝试close自身，然后将栈上的值拷贝到内部, 从open链表上断掉链接，结构如图所示。

![close_upvalues](close_upvalues.png)

核心代码如下

```C
int luaF_close (lua_State *L, StkId level, int status) {
  UpVal *uv;
  while ((uv = L->openupval) != NULL && uplevel(uv) >= level) {
    TValue *slot = &uv->u.value;  /* new position for value */
    lua_assert(uplevel(uv) < L->top);
    if (uv->tbc && status != NOCLOSINGMETH) {
      /* must run closing method, which may change the stack */
      ptrdiff_t levelrel = savestack(L, level);
      status = callclosemth(L, uplevel(uv), status);
      level = restorestack(L, levelrel);
    }
    luaF_unlinkupval(uv);
    setobj(L, slot, uv->v);  /* move value to upvalue slot */
    uv->v = slot;  /* now current value lives here */
    if (!iswhite(uv))
      gray2black(uv);  /* closed upvalues cannot be gray */
    luaC_barrier(L, uv, slot);
  }
  return status;
}

void luaF_unlinkupval (UpVal *uv) {
  lua_assert(upisopen(uv));
  *uv->u.open.previous = uv->u.open.next;
  if (uv->u.open.next)
    uv->u.open.next->u.open.previous = uv->u.open.previous;
}
```

### 编译器如何生成闭包

* 编译器递归地编译内层函数。当编译器发现一个内层函数时，它停止为当前函数生成指令，转而为新的函数生成完整的指令。在编译内层函数时，编译器为该函数需要的所有upvalue创建一个列表，并映射到最后一个创建的闭包中。当函数结束时，编译器返回外层函数，并生成一条创建带有所需upvalue的闭包的指令。

* 当编译器看到一个变量名时，它会查找该变量。查找的结果有三种：变量为函数内部的局部变量，变量为函数外部的局部变量，变量为全局变量。编译器首先查找函数内部的局部变量，若找到，则可确定变量保存在当前活动记录的某个固定的寄存器中。若没有找到，则递归地查找外层函数的变量；若没有找到变量名，则该变量是全局变量（与Scheme的某些实现相同，Lua的变量默认为全局)，否则该变量是某个外层函数的局部变量。在这种情况下，编译器会添加（或复制）一个该变量的引用到当前函数的upvalue表中。如果一个变量在upvalue表中，则编译器使用它在表中的位置生成```GETUPVAL```或```SETUPVAL```指令。

* 查找变量的过程是递归的，因此当一个变量被加入当前函数的upvalue表中时，该变量要么是来自外部函数的局部变量，要么是从已被加入外层函数的upvalue表复制引用过来。

以下方函数举个例子
```Lua
function A (a)
    local function B (b)
        local function C (c)
            return c + g + b + a
        end
    end
end
```

* 编译器首先在当前局部变量表找到变量```c```，因此将其作为局部变量处理。
* 然后编译器在当前函数找不到```g```，因此在它的外层函数```B```中查找，接着递归地在函数```A```和包裹程序块的匿名函数中查找。最终没有找到```g```，于是将其作为全局变量处理。
* 编译器在当前函数中找不到变量```b```，向外递归在函数```B```中查找；在函数```B```的局部变量表中找到```b```，于是在函数```C```的```upvalue```表中加入```b```。
* 最后，编译器在当前函数中找不到a，因此在函数```B```中查找。在```B```中也没有找到，紧接着在```A```中查找, 发现```a```为```A```的局部变量，于是将```a```加入```B```的upvalue表和```C```的upvalue表当中。

# 总结

Lua闭包其实就是一层层的函数套娃，某个执行过后的函数退出close后，如果被其内部函数引用了局部变量，那么会生成一个内闭的环境存下这些局部变量，然后内部函数通过upvalue列表去访问, 否则的话就去open共享链表上找上值。

# 参考资料

* [Roberto Ierusalimschy, Luiz Henrique de Figueiredo† , Waldemar Celes.  [Closures in Lua].      April 24, 2013](/download/closures-draft.pdf)