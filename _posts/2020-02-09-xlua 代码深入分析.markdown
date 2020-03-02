---
layout:     post
title:      "xlua 代码深入分析"
subtitle:   ""
date:       2020-02-09
author:     "CodeSuperHero"
header-img: "img/lua1.jpg"
tags:
    - lua
    - xlua
    - Unity
---

# xlua 相关知识点分享

## c/c++ 拓展

### 基础知识
[xlua增删第三方库](https://github.com/Tencent/xLua/blob/master/Assets/XLua/Doc/XLua%E5%A2%9E%E5%8A%A0%E5%88%A0%E9%99%A4%E7%AC%AC%E4%B8%89%E6%96%B9lua%E5%BA%93.md)

[CMake简单教程](https://www.hahack.com/codes/cmake/)

### 拓展步骤

1. 编写一个符合lua拓展的c/c++库
2. 编辑 cmakelist.txt 文件，填入编写的库
3. 编译后导入 unity plugin，修改luaEnv c#代码，初始化库
4. 在lua中require，然后调用接口

#### 编写符合Lua拓展的 c/c++ 库

这里以c++为例

``` lua-extendTest/ExtendTest.h
#pragma once

#ifdef __cplusplus  //兼容.cpp 和 .c 文件
extern "C"          //指定编译和连接规则。但不影响代码的语义
{
#endif
    int Add(const int& a, const int& b);
#ifdef __cplusplus
}
#endif
```

``` lua-extendTest/ExtendTest.cpp
#include "lua.hpp"      // lua 提供的 c++ 代码头文件
#include "ExtendTest.h" 
extern "C"
{
#include "i64lib.h"
}

int Add(const int& a, const int& b) 
{
    return a + b;
}

static int add(lua_State* L) { ... }

static int addTable(lua_State* L) { ... }

static const luaL_Reg methods[] = // 写入lua table 的函数数组
{
    {"addnumber", add},
    {"addnumberTable", addTable},
    {NULL, NULL}
};

extern "C"
{
    LUALIB_API int luaopen_luatestlib(lua_State *L) //标准语法，导出初始化函数
    {
        luaL_newlib(L, methods); //新建table, lua require 语法会返回这个 table，methods 里面存储的就是table的方法
        return 1;
    }
}
```

lua 堆栈索引有两种取值方式，一种是整数，从1开始递增。一种是负数，从 -1 开始递减。

```
Top         -1, 5
...         -2, 4
...         -3, 3
...         -4, 2
Bottom      -5, 1
```

- 正数索引可以直接取到栈底，匹配方法调用时的参数入栈顺序
- 负数索引可以直接取到栈顶，在获取table数据时非常方便
- lua_gettop 可以获取参数个数，方便做异常检查

```
addnumber(a, b) //lua 代码调用

static int add(lua_State* L)// lua堆栈操作函数，作为lua函数和c++代码的中间代码
{
    // lua_Integer lua.h 定义类型，用以表示Lua堆栈中的 int 类型数据
    // luaL_checkinteger 从lua堆栈中获取 lua_Integer 数据，每个类型都有对应方法. 尽量使用luaL_check_xx方法操作堆栈，避免搞乱堆栈
    // 1, 2 代表数据在堆栈中的索引位置
    lua_Integer a = luaL_checkinteger(L, 1); 
    lua_Integer b = luaL_checkinteger(L, 2); 
    lua_Integer c = Add(a, b);
    //lua_pushinteger 数据压入lua堆栈，存放在栈顶。尽量使用lua_push_xx方法操作堆栈，这个操作不会，避免搞乱堆栈
    lua_pushinteger(L, c); 
    //返回参数个数,lua支持多返回值
    return 1;
}

local x = {a = 1, b = 2}
addnumberTable(x)

static int addTable(lua_State* L)
{
    lua_getfield(L, 1, "a");
    lua_Integer a = luaL_checkinteger(L, -1);
    lua_getfield(L, 1, "b");
    lua_Integer b = luaL_checkinteger(L, -1);
    lua_pushinteger(L, Add(a, b));
    return 1;
}
```

#### 修改 cmakelist.txt 编译修改后的xlua库

```
#begin lua-extendTest
set (EXTENDTEST_SRC lua-extendTest/ExtendTest.cpp)
set_property(
    SOURCE ${EXTENDTEST_SRC} 
    APPEND
    PROPERTY COMPILE_DEFINITIONS 
    LUA_LIB 
)
list(APPEND THIRDPART_INC lua-extendTest)
set (THIRDPART_SRC ${THIRDPART_SRC} ${EXTENDTEST_SRC})
#end lua-extendTest
```

命令行调用 make 文件，讲编译后的库导入 unity.
注意需要重启 unity，才能加载新导入的库

#### 修改c#的初始化方法

1. 修改 XLua.LuaDLL.Lua 类。

```
[DllImport(LUADLL, CallingConvention = CallingConvention.Cdecl)]
public static extern int luaopen_luatestlib(System.IntPtr L);

[MonoPInvokeCallback(typeof(LuaDLL.lua_CSFunction))]
public static int LoadLuaTest(System.IntPtr L)
{
    return luaopen_luatestlib(L);
}
```

2.修改 DH.IM.LuaMain 类，在 InitLuaEnv 方法添加代码

```
luaEnv.AddBuildin("extendtest", Lua.LoadLuaTest);
```

#### 在lua中使用

```
test = require "extendtest"
local a = 1
local b = 2
local c = { a = 1, b = 2 }
Debug.Log("a :"..tostring(a))
Debug.LogError(tostring(test.addnumber(a, b)))
Debug.LogError(tostring(test.addnumberTable(c)))
```

<img src="img/lua_call.png" alt="lua_call" title="" width="371" height="68" />


## Lua与c#数据传递

lua是一种内嵌语言，运行时依赖于宿主，xlua是通过修改原生的 lua c 源码实现，所以c#和lua是通过c语言通信。

### 基础知识

- IntPtr 存储非托管堆指针的类，c#与非托管内存交互时，通过IntPtr调用指针对象
- Marshal 托管堆内存和非托管内存转换类，通过类的方法可以在托管和非托管内存上进行互相拷贝
- C# 通过上述对象与 lua 的 c 虚拟机进行交互，从而产生和 lua 代码的互相调用

### c# 调用 lua

1. 获取全局的字段方法。

luaEnv.Global 变量，是对lua 的 "_G" 全局表的映射
在lua中声明的全局方法，变量，均可以直接通过 luaEnv.Global.Get 函数获取。

```
global.lua
luaStart = function()
    Game:Start()
end
```

```
LuaMain.cs
luaEnv.Global.Get("luaStart", out luaStart);
luaStart();
```

luaEnv.Global 变量，是对lua 的 "_G" 全局表的映射，通过 LuaEnv.Global 可以获取到 lua 全局表的所有字段。

```
public void Get<TKey, TValue>(TKey key, out TValue value)
{
#if THREAD_SAFE || HOTFIX_ENABLE
    lock (luaEnv.luaEnvLock)
    {
#endif
        var L = luaEnv.L; // 获取 LuaState 指针
        var translator = luaEnv.translator; // xlua 用于c#对象和lua对象转换的操作类
        int oldTop = LuaAPI.lua_gettop(L);  // 操作之前记录 lua 栈顶，方便还原
        LuaAPI.lua_getref(L, luaReference); // 将 "_G" 表推入 lua 栈顶
        translator.PushByType(L, key);      // 压入将要获取的字段名

        if (0 != LuaAPI.xlua_pgettable(L, -2))  // 检查"_G"表
        {
            string err = LuaAPI.lua_tostring(L, -1);
            LuaAPI.lua_settop(L, oldTop);
            throw new Exception("get field [" + key + "] error:" + err);
        }

        LuaTypes lua_type = LuaAPI.lua_type(L, -1); // 获取栈顶字段的类型
        Type type_of_value = typeof(TValue);        // 获取c#字段的类型
        if (lua_type == LuaTypes.LUA_TNIL && type_of_value.IsValueType()) // 栈顶字段为空且c#是值类型，则报错。
        {
            throw new InvalidCastException("can not assign nil to " + type_of_value.GetFriendlyName());
        }

        try
        {
            //函数调用前，栈顶已经是 luaStart function
            translator.Get(L, -1, out value); // 尝试建立栈顶对象和传入对象的映射
        }
        catch (Exception e)
        {
            throw e;
        }
        finally
        {
            LuaAPI.lua_settop(L, oldTop);   // 还原堆栈
        }
#if THREAD_SAFE || HOTFIX_ENABLE
    }
#endif
}
```

ObjectTranslator.Get 函数的实现如下
``` 
public void Get<T>(RealStatePtr L, int index, out T v)
{
    Func<RealStatePtr, int, T> get_func;
    if (tryGetGetFuncByType(typeof(T), out get_func))
    {
        v = get_func(L, index);
    }
    else
    {
        v = (T)GetObject(L, index, typeof(T));
    }
}
```

ObjectTranslator.GetObject 函数实现如下
```
public object GetObject(RealStatePtr L, int index, Type type)
{
    ...
    //return (objectCasters.GetCaster(type)(L, index, null));
    //ObjectCaster : public delegate object ObjectCast(RealStatePtr L, int idx, object target); 
    ObjectCaster oc = objectCasters.GetCaster(type); 
    object obj = oc(L, index, null);
    return obj;
}
```

这里由于传入的 luaStart 字段是 System.Action 类型，最终调用 ObjectTranslator.CreateDelegateBridge(RealStatePtr L, Type delegateType, int idx)
CreateDelegateBridge 根据传入参数生成新的 DelegateBridge，通过获取的 luaRefrence 字段关联 lua 方法

```
public object CreateDelegateBridge(RealStatePtr L, Type delegateType, int idx)
{
    ...
    LuaAPI.lua_pushvalue(L, idx);
    int reference = LuaAPI.luaL_ref(L);
    LuaAPI.lua_pushvalue(L, idx);
    LuaAPI.lua_pushnumber(L, reference);
    LuaAPI.lua_rawset(L, LuaIndexes.LUA_REGISTRYINDEX);
    DelegateBridgeBase bridge;
    try
    {
#if (UNITY_EDITOR || XLUA_GENERAL) && !NET_STANDARD_2_0
        if (!DelegateBridge.Gen_Flag)
        {
            bridge = Activator.CreateInstance(delegate_birdge_type, new object[] { reference, luaEnv }) as DelegateBridgeBase;
        }
        else
#endif
        {
            bridge = new DelegateBridge(reference, luaEnv);
        }
    }
    ...
    if (delegateType == null)
    {
        delegate_bridges[reference] = new WeakReference(bridge);
        return bridge;
    }
    try
    {
        var ret = getDelegate(bridge, delegateType);
        bridge.AddDelegate(delegateType, ret);
        delegate_bridges[reference] = new WeakReference(bridge);
        return ret;
    }
    ...
}
```

最终通过 ObjectTranslator.getDelegate(DelegateBridgeBase bridge, Type delegateType) 返回给 LuaMain 的 luaStart 字段

```
Delegate getDelegate(DelegateBridgeBase bridge, Type delegateType)
{
    Delegate ret = bridge.GetDelegateByType(delegateType);
    if (ret != null)
    {
        return ret;
    }
    ...
}

public partial class DelegateBridge : DelegateBridgeBase
{
	public void __Gen_Delegate_Imp0()
	{
#if THREAD_SAFE || HOTFIX_ENABLE
        lock (luaEnv.luaEnvLock)
        {
#endif
            RealStatePtr L = luaEnv.rawL;
            int errFunc = LuaAPI.pcall_prepare(L, errorFuncRef, luaReference);
            PCall(L, 0, 0, errFunc);
            LuaAPI.lua_settop(L, errFunc - 1);
#if THREAD_SAFE || HOTFIX_ENABLE
            }
#endif
		}
        
```
luaStart 最终引用的是 DelegateBridge.__Gen_Delegate_Imp0() 方法

2. 获取全局table

获取Lua中全局 Table 的数据，则需要将Table映射为C#中相对应的数据结构，可选的方式有：class，interface，Dictionary，List和LuaTable
如果只获取 table 中的字段，则使用class就可以，如果需要获取table中的方法，则必须使用 interface,且必须导出 interface

```
class t
{
    public int x;
    public int y;
}

[CSharpCallLua]
public interface it
{
    int Add(int x, int y);
}

private string script = 
@"
    t = 
    { 
        x = 1,
        y = 2,
    }
    function t:Add (a, b)
        return a + b
    end
";

private string printScr = 
@"
    print(tostring(t.x))
    print(tostring(t.y))
";

private void Start() {
    LuaEnv luaenv = new LuaEnv();
    luaenv.DoString(script);
    var t = luaenv.Global.Get<t>("t");
    var it = luaenv.Global.Get<it>("t");
    Debug.LogError(it.Add(t.x, t.y));
}
```
注意这种方式调用方法，在 lua 中需要以对象形式定义，c#中的 interface 必须导出注册代码

### lua调用c#

lua 调用 c# 是通过调用 c# 通过 c 的 api 注册到虚拟机的方法，让 c# 的方法通过 c 的 api 主动获取 lua 的数据

lua 代码调用 c# 代码的形式通常是

```
Time = CS.UnityEngine.Time
```

在 LuaEnv 有一段初始化的 lua 代码

```
local metatable = {}
local rawget = rawget
local setmetatable = setmetatable
local import_type = xlua.import_type
local import_generic_type = xlua.import_generic_type
local load_assembly = xlua.load_assembly
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
...
CS = CS or {}
setmetatable(CS, metatable)
```

在Lua中调用 "CS.xx"代码时会调用 "metatable:__index(key) ",其中 xlua.import_type(fqn) 函数会寻找对应的c#类型

XLua.ObjectTranslator.OpenLib

```
public void OpenLib(RealStatePtr L)
{
    if (0 != LuaAPI.xlua_getglobal(L, "xlua"))
    {
        throw new Exception("call xlua_getglobal fail!" + LuaAPI.lua_tostring(L, -1));
    }
    LuaAPI.xlua_pushasciistring(L, "import_type");
    LuaAPI.lua_pushstdcallcfunction(L, importTypeFunction);
    ...
}
```

XLua.StaticLuaCallbacks.ImportType

```
public static int ImportType(RealStatePtr L)
{
    ObjectTranslator translator = ObjectTranslatorPool.Instance.Find(L);
    string className = LuaAPI.lua_tostring(L, 1);
    Type type = translator.FindType(className);
    if (type != null)
    {
        if (translator.GetTypeId(L, type) >= 0)
        {
            LuaAPI.lua_pushboolean(L, true);
        }
        else
        {
            return LuaAPI.luaL_error(L, "can not load type " + type);
        }
    }
    else
    {
        LuaAPI.lua_pushnil(L);
    }
    return 1;
}
```

这里关键代码是 GetTypeId，最终调用到 ObjectTranslator.getTypeId

```
internal int getTypeId(RealStatePtr L, Type type, out bool is_first, LOGLEVEL log_level = LOGLEVEL.WARN)
{
    ...
    if (TryDelayWrapLoader(L, alias_type == null ? type : alias_type))
    {
        LuaAPI.luaL_getmetatable(L, alias_type == null ? type.FullName : alias_type.FullName);
    }
    ...
}

public bool TryDelayWrapLoader(RealStatePtr L, Type type)
{
    ...
    if (delayWrap.TryGetValue(type, out loader))
    {
        delayWrap.Remove(type);
        loader(L);
    }
    else
    {
#if !GEN_CODE_MINIMIZE && !ENABLE_IL2CPP && (UNITY_EDITOR || XLUA_GENERAL) && !FORCE_REFLECTION && !NET_STANDARD_2_0
        if (!DelegateBridge.Gen_Flag && !type.IsEnum() && !typeof(Delegate).IsAssignableFrom(type) && Utils.IsPublic(type))
        {
            Type wrap = ce.EmitTypeWrap(type);
            MethodInfo method = wrap.GetMethod("__Register", BindingFlags.Static | BindingFlags.Public);
            method.Invoke(null, new object[] { L });
        }
        else
        {
            Utils.ReflectionWrap(L, type, privateAccessibleFlags.Contains(type));
        }
#else
        Utils.ReflectionWrap(L, type, privateAccessibleFlags.Contains(type));
#endif
#if NOT_GEN_WARNING
        if (!typeof(Delegate).IsAssignableFrom(type))
        {
#if !XLUA_GENERAL
            UnityEngine.Debug.LogWarning(string.Format("{0} not gen, using reflection instead", type));
#else
            System.Console.WriteLine(string.Format("Warning: {0} not gen, using reflection instead", type));
#endif
        }
#endif
    }
}
```

- delayWrap 存储了c#导出代码的初始化方法，第一次 lua 调用时才会注册到lua虚拟机
- 如果是没有导出的代码，则会调用 Utils.ReflectionWrap 方法，通过反射动态注册到lua虚拟机

### string 传递

lua 传递给 c#

```
[DllImport(LUADLL, CallingConvention = CallingConvention.Cdecl)]
public static extern IntPtr lua_tolstring(IntPtr L, int index, out IntPtr strLen); //c 接口

public static string lua_tostring(IntPtr L, int index)
{
    IntPtr strlen;
    IntPtr str = lua_tolstring(L, index, out strlen);
    if (str != IntPtr.Zero)
    {
#if XLUA_GENERAL || (UNITY_WSA && !UNITY_EDITOR)
        int len = strlen.ToInt32();
        byte[] buffer = new byte[len];
        Marshal.Copy(str, buffer, 0, len);
        return Encoding.UTF8.GetString(buffer);
#else
        string ret = Marshal.PtrToStringAnsi(str, strlen.ToInt32());
        if (ret == null)
        {
            int len = strlen.ToInt32();
            byte[] buffer = new byte[len];
            Marshal.Copy(str, buffer, 0, len);
            return Encoding.UTF8.GetString(buffer);
        }
        return ret;
#endif
    }
    else
    {
        return null;
    }
}

LUA_API const char *lua_tolstring (lua_State *L, int idx, size_t *len) {
  StkId o = index2addr(L, idx);
  if (!ttisstring(o)) {
    if (!cvt2str(o)) {  /* not convertible? */
      if (len != NULL) *len = 0;
      return NULL;
    }
    lua_lock(L);  /* 'luaO_tostring' may create a new string */
    luaO_tostring(L, o);
    luaC_checkGC(L);
    o = index2addr(L, idx);  /* previous call may reallocate the stack */
    lua_unlock(L);
  }
  if (len != NULL)
    *len = vslen(o);
  return svalue(o);
}
```

每次 lua 到 c# 的字符串传递，会在 c# new 一次char[],理论上需要减少string传递的场景。
这里的c#代码看起来是可以优化一下，减少 new byte[] 的次数。

### int

```
[DllImport(LUADLL, CallingConvention = CallingConvention.Cdecl)]
public static extern bool xlua_tointeger(IntPtr L, int index);

LUA_API int xlua_tointeger (lua_State *L, int idx) {
	return (int)lua_tointeger(L, idx);
}

#define lua_tointeger(L,i)	lua_tointegerx(L,(i),NULL)

LUA_API lua_Integer lua_tointegerx (lua_State *L, int idx, int *pisnum) {
    lua_Integer res;
    const TValue *o = index2addr(L, idx);
    int isnum = tointeger(o, &res);
    if (!isnum)
        res = 0;  /* call to 'tointeger' may change 'n' even if it fails */
    if (pisnum) *pisnum = isnum;
    return res;
}

```

