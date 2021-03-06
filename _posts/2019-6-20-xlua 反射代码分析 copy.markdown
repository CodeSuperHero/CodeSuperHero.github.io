---
layout:     post
title:      "xlua 反射调用C#代码分析"
subtitle:   ""
date:       2019-06-20
author:     "CodeSuperHero"
header-img: "img/lua1.jpg"
tags:
    - lua
    - xlua
    - Unity
---

# xlua 反射调用C#代码分析

## 前言

- 本文以xlua官方工程 09_GenericMethod 示例demo代码为例.
- 本文只分析xlua反射调用.

xlua 比之其它lua的最大优势，就是可以不需要导出C#代码，直接用lua代码替代C#代码，做到逻辑上的热修复。

我们先来看下lua代码

```
    local foo1 = CS.XLuaTest.Foo1Child()
    local foo2 = CS.XLuaTest.Foo2Child()
    local obj = CS.UnityEngine.GameObject() 
    foo1:PlainExtension()   
    foo1:Extension1()   
    foo1:Extension2(obj) -- overload1   
    foo1:Extension2(foo2) -- overload2  
    local foo = CS.XLuaTest.Foo()   
    foo:Test1(foo1) 
    foo:Test2(foo1,foo2,obj)    
```

在unity中，lua代码与C#代码之间的通信，都是通过C语言来中转，C#通过C语言的接口，把代码注册到Lua状态机里，供lua代码调用，所以一般使用Lua，都会生成注册代码。
但上述代码中 “CS.XLuaTest.Foo1Child()” 调用的C#类"Foo1Child"，却没有预先生成代码注册。那么xlua是如何做到不注册也能调用，就是本文讨论的知识点了。

## 分析

查看 LuaEnv 类中的 init_xlua 代码：

```
function metatable:__index(key) 
                local fqn = rawget(self,'.fqn')
                fqn = ((fqn and fqn .. '.') or '') .. key

                local obj = import_type(fqn)

                if obj == nil then
                    -- It might be an assembly, so we load it too.
                    obj = { ['.fqn'] = fqn }
                    setmetatable(obj, metatable)
                elseif obj == true then
                    return rawget(self, key)
                end

                -- Cache this lookup
                rawset(self, key, obj)
                return obj
            end
```

可以发现，当我们第一次调用 CS.XLuaTest 的时候， xlua会调用 “import_type” 函数。我们搜索这个函数，找到代码如下

```
 ublic void OpenLib(RealStatePtr L)
{
    ...
    LuaAPI.xlua_pushasciistring(L, "import_type");
	LuaAPI.lua_pushstdcallcfunction(L,importTypeFunction);
    ...
}
```

xlua在初始化的时候，会默认注册这个委托，当调用的 "CS.XLuaTest" 里的“XLuaTest”第一次被调用时，就会触发这个委托

```
public static int ImportType(RealStatePtr L)
{
try
{
    ...
    string className = LuaAPI.lua_tostring(L, 1);
    Type type = translator.FindType(className);
    if (translator.GetTypeId(L, type) >= 0)
    {
        LuaAPI.lua_pushboolean(L, true);
    }
    ...
}
```

从 translator.GetTypeId 一路查看下去，我们终于找到了最终的方法 Utils.ReflectionWrap ,这个方法会根据传递进来的类型type生成lua原表，并通过C语言API注入到Lua全局表里面。
所以当我们调用 “CS.XLuaTest.Foo1Child()” 时，会触发原表的 "__call" 方法，从而调用到预先诸如的构造函数，从而返回C#示例。

## 结论

所以当我们调用 “CS.XLuaTest.Foo1Child()" 时， xlua 会先访问初始化时生成的的 “CS” 表，从而生成 “XLuaTest” 表，因为 “XLuaTest” 在 C# 是命名空间，而不是具体的类，所以会再生成 “Foo1Child” 表，因为是具体的类型，就会调用预先注入的 import_type, 给 Foo1Child 反射导出一份 lua 表，供lua调用。