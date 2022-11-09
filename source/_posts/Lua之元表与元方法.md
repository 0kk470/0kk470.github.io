---
title: Lua之元表与元方法
date: 2022-11-09 22:13:36
tags: [Lua, table]
categories: Lua
---

# 前言

复习下Lua元表相关的知识

# 正文

## 定义

* 元表
    元表可以修改一个值在面对一个未知操作时的行为，是一个操作行为拓展的集合，常常用于拓展```table```的行为，也可用于```userdata```。
* 元方法
    可以理解为元表中的某个字段指向的拓展函数或者拓展表。

## 常用元方法字段

### __index

当访问一个表中不存在的字段时，如果该表存在元表，那么解释器会尝试去查找该元方法，并由这个元方法提供最终结果。

#### 应用：实现面向对象的继承
我们可以用元表的```__index```元方法来实现一个简单的单继承机制。

* 类的继承实现

```Lua
local definedClasses = {}

function class(_className, _baseClass)
    assert(type(_className) == "string")
    assert(type(_baseClass) == "table" or type(v) == "nil")
    assert(definedClasses[_className] == nil, "duplicate class define, class name:" .. _className)
    local _newclass = {__name=_className, super = _baseClass}
    _newclass.__index = _newclass
    definedClasses[_className] = _newclass
    setmetatable(_newclass, {__index = _baseClass})

    _newclass.ctor = function()

    end

    _newclass.new = function(...)
        local _instance = {}
        setmetatable(_instance, _newclass)
        _instance:ctor(...)
        return _instance
    end

    return _newclass
end
```
* 构建一个```a<-b<-c```的继承链。
```Lua
local a = class("parent")

function a:ctor()
    print("ctor a")
end

function a:Hello()
    print("parent Hello")
end

local b = class("child", a)
function b:Hello()

    print("Child Hello ")
end

function b:ctor(...)
    print("ctor b")
end

local c = class("grandson", b)
```
* 构建三个类的实例化对象进行测试。
```Lua
local insA = a:new()
local insB = b:new()
local insC = c:new()
insA:Hello()
insB:Hello()
insC:Hello()
```
* 输出如下结果，可以看到子类```C```发现自己的表中没有```Hello```这个字段，就通过元表索引到了```B```的同名方法进行了调用。
![class_print](1.png)

### __newindex

这个元方法会在向表内一个不存在的键索引赋值时触发。

#### 应用：使用```__newindex```跟踪表的赋值操作

测试代码
```Lua

local function track_func(t, key, val) 
   print(os.date("%c",os.time()) .. " insert a new key: " .. key)
   rawset(t, key, val)
end

local track = {}
local mt = {__newindex = track_func }
setmetatable(track, mt)
track.a = "a"
track.b = "b"
track[1] = 1
```

输出如下
![2](2.png)

需要注意的是在```track_func```部内部使用的```rawset```来替代```t[key] = val```，从而避免赋值操作重复触发```__newindex```的递归死循环。

### 其他元方法

其他的元方法示例可以参见[此处](https://github.com/0kk470/pil4/blob/master/chapter20/chapter20.lua)。

所有元方法字段参见Lua Wiki的[MetatableEvents章节](http://lua-users.org/wiki/MetatableEvents)。

### MetaTable的操作源码

```C
static int luaB_getmetatable (lua_State *L) {
  luaL_checkany(L, 1);
  if (!lua_getmetatable(L, 1)) {
    lua_pushnil(L);
    return 1;  /* no metatable */
  }
  luaL_getmetafield(L, 1, "__metatable");
  return 1;  /* returns either __metatable field (if present) or metatable */
}


static int luaB_setmetatable (lua_State *L) {
  int t = lua_type(L, 2);
  luaL_checktype(L, 1, LUA_TTABLE);
  luaL_argexpected(L, t == LUA_TNIL || t == LUA_TTABLE, 2, "nil or table");
  if (luaL_getmetafield(L, 1, "__metatable") != LUA_TNIL)
    return luaL_error(L, "cannot change a protected metatable");
  lua_settop(L, 2);
  lua_setmetatable(L, 1);
  return 1;
}

LUA_API int lua_setmetatable (lua_State *L, int objindex) {
  TValue *obj;
  Table *mt;
  lua_lock(L);
  api_checknelems(L, 1);
  obj = index2value(L, objindex);
  if (ttisnil(s2v(L->top - 1)))
    mt = NULL;
  else {
    api_check(L, ttistable(s2v(L->top - 1)), "table expected");
    mt = hvalue(s2v(L->top - 1));
  }
  switch (ttype(obj)) {
    case LUA_TTABLE: {
      hvalue(obj)->metatable = mt;
      if (mt) {
        luaC_objbarrier(L, gcvalue(obj), mt);
        luaC_checkfinalizer(L, gcvalue(obj), mt);
      }
      break;
    }
    case LUA_TUSERDATA: {
      uvalue(obj)->metatable = mt;
      if (mt) {
        luaC_objbarrier(L, uvalue(obj), mt);
        luaC_checkfinalizer(L, gcvalue(obj), mt);
      }
      break;
    }
    default: {
      G(L)->mt[ttype(obj)] = mt;
      break;
    }
  }
  L->top--;
  lua_unlock(L);
  return 1;
}

```
从源码中还可以发现一些元表的使用trick:
* 可以设置某张表的```__metatable```为非空字段来屏蔽元表功能或者保护已设置好的元表。
* Lua的```C API```除了可以对```Table```和```UserData```设置元表外，还可以对其他类型也设置一个全局的元表。
* 设置元表会触发```LuaGC```的屏障机制来避免对应的表在当前可能处于GC回收阶段的情况下被回收掉。
* 函数```setmetatable```是有返回值的，为设置了metatable的table。


# 总结

* 元表非常适合用来拓展自己想要的额外逻辑。```XLua```的```Lua侧访问Unity和C#的对象```机制就是通过```UserData```的元表机制来实现的。
* 元表的设计思路个人觉得非常类似设计模式的观察者模式，元表的各种字段就相当于各种操作事件，元表就相当于这一系列操作事件的订阅者，Lua虚拟机执行在代码特定位置时会触发相应的事件来调用元表中的元方法。