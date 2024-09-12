---
title: stl-function 实现
description: 剪短的描述
author: momo
categories:
- C++
- stl容器
tags:
- C++
- stl容器
layout: post
pin: false
math: true
date: 2024-08-10 15:37 +0800
---
在C++中一个装饰器类，可承载任何可调用对象，包括 lambda表达式, func, func ptr, class with operator(), 且这样在编译时仅按照相同参数和返回来实例化一份模板，并不会因为是否为函数指针，函数以及可调用对象导致实例化多份. 实现一个这样的容器类最关键的地方在于`类型擦除`，即将任何可调用对象转为内部的FuncBase类的调用目标，通过 invoke 实现只关心参数及返回，并通过模板子类实现自动实例化需要被包装的可调用对象，这样一个最基本的结构应该是：

```c++
// 对外暴露模板参数应只关注调用对象的 `签名`,  也即返回值类型和参数包
template<class Ret, class ...Args>
struct function {

};
```

那么如果传入的模板参数非法怎么办? 这里可以重新定义模板，使得编译器不能通过重载决议的情况退回默认模板：

```c++
// 默认情况，满足上述的模板会变为下述情况，并直接编译器报错
// 明确我们的容器只接受指定为返回值和参数的函数，其他的情况都为错误情况
template<class Fn>
struct function {
  static_assert(!std::is_function_v<Fn>, "Not a valid Function");
}

// 而我们实现的版本则使用偏特化，表示我们接收的是一种函数
template<class Ret, class ...Args>
struct function<Ret(Args...)> {
  
}
```

为了实现类型擦除，我们应该采用虚函数的方式实现，或者说多态技术:
```c++
// ...

template<class Ret, class ...Args>
struct function<Ret(Args)> {
  // 这一类可调用对象都使用FuncBase call 进行实际调用
  struct FuncBase {
    virtual _call(Args...) = 0;
    virtual ~FuncBase() = default;
  }

  // 加入额外模板参数，用来存储外部的可调用对象
  template<class = std::enable_if_t<
        std::is_constructible_v<F, Args...> && 
        std::is_same_v<std::invoke_result_t<std::decay_t<F>, Args...>, Ret>
    >>
  struct FuncImpl : FuncBase {
    F m_func;
    Ret _call(Args &&...args) override {
      return std::invoke(m_func, std::forward<Args>(args)...);
    }
    FuncImpl(F &&f) : m_func(std::forward(f)) {}
  }

  // 对于function来说，任务是接收外部可调用对象，同时提供调用接口
  template <class F, class = std::enable_if_t<
                       std::is_invocable_r_v<Ret, F, Args...> &&
                       !std::is_same_v<std::decay_t<F>, Function>>>
  function(F &&f) : m_func(std::make_shared<FuncImpl<std::decay_t(F)>>(std::forward<std::decay_t(F)>(f))) {}

  Ret operator()(Args &&...args) {
    return m_func->_call(std::forward<Args>(args)...);
  }

  std::shared_ptr<FuncBase> m_func;
}
```

这样，一个支持可调用对象的function容器就可以完成基本的任务了, 现在我们可以测试一下常见的集中可调用对象:

```c++
int func1(int a) {
  cout << "func what = " << a << '\n';
  return a;
}

struct obj {
  static int obj_func(int a) {
    cout << "obj func what = " << a << '\n';
    return a;
  }
};

int main() {
  // func
  function<int(int)> func = func1;
  auto ret = func(1);
  cout << "output = \"" << ret << "\"\n";

  // func ptr
  function<int(int)> func_ptr = &func1;
  ret = func_ptr(2);
  cout << "output = \"" << ret << "\"\n";

  // obj func
  function<int(int)> func_obj = obj::obj_func;
  ret = func_obj(3);
  cout << "output = \"" << ret << "\"\n";

  // obj func ptr
  function<int(int)> func_obj_ptr = &obj::obj_func;
  ret = func_obj_ptr(4);
  cout << "output = \"" << ret << "\"\n";

  function<int(int)> func_lambda = [](int a) {
    cout << "lambda what = " << a << '\n';
    return a;
  };
  ret = func_lambda(5);
  cout << "output = \"" << ret << "\"\n";
  return 0;
}

/* 
输出结果: 
> func what = 1
> output = "1"
> func what = 2
> output = "2"
> obj func what = 3
> output = "3"
> obj func what = 4
> output = "4"
> lambda what = 5
> output = "5"
 */
```

看起来一切都正常，那如果我们想让该容器实现模板参数自动推导呢，也就是CTAD的特性，按照我们使用常见容器的方式，只要把类型中的模板参数留空，就可以自动推导：

```c++
int func1(int a) {
  cout << "func what = " << a << '\n';
  return a;
}

int main(){
  function func = func1;
  // 编译出错 class template argument deduction failed:
}
```

出现了错误，看一下错误原因，是我们类模板参数推导不出来，这里我们试着给他提供一下`推导指南`吧:

```c++
template<class F> //我们传入的参数是什么来着？就是一个可调用对象
struct function(F) -> function<???>;
// 我们需要传入的是一个F对象，推导的是F对应的函数签名，需要使用额外的辅助萃取出对应类型

template <class F> struct function_traits;

template <class R, class... Args> struct function_traits<R(Args...)> {
  using type = R(Args...);
};

template <class R, class... Args> struct function_traits<R (*)(Args...)> {
  using type = R(Args...);
};

template <class C, class R, class... Args>
struct function_traits<R (C::*)(Args...)> {
  using type = R(Args...);
};

template <class C, class R, class... Args>
struct function_traits<R (C::*)(Args...) const> {
  using type = R(Args...);
};

// 可调用对象的类型也即 operator() 函数的类型
template <class F> struct function_traits {
  using fn_type = std::decay_t<decltype(&F::operator())>;
  using type = typename function_traits<fn_type>::type;
};

template <class F> function(F) -> function<typename function_traits<F>::type>;

template<class F> //我们传入的参数是什么来着？就是一个可调用对象，现在可以使用类型萃取的方式从 F 中获取我们需要的内容
struct function(F) -> function<function_traits<F>::type>;
```

正好再和标准库学习一下，给一个 `function_traits_t` 别名吧：

```c++
template<class F>
using function_traits_t = typename function_traits<F>::type;

// 这样我们的推导指南可以写成
template<class F>
function(F) -> function<function_traits_t<F>>;
```

提供了类型推导指南后，我们就可以开开心心地让编译器帮我们推导出来具体的模板参数，而不需要我们手动指明：

```c++
void test() {
  // func
  function func = func1;
  auto func_sig = func1;
  auto ret = func(1);
  cout << "output = \"" << ret << "\"\n";

  // func ptr
  function func_ptr = &func1;
  auto func_ptr_sig = &func1;
  ret = func_ptr(2);
  cout << "output = \"" << ret << "\"\n";

  obj obj_instance;

  // obj func
  function func_obj = obj::obj_func;
  auto func_obj_sig = &obj::obj_func;
  ret = func_obj(3);
  cout << "output = \"" << ret << "\"\n";

  // obj func ptr
  function func_obj_ptr = &obj::obj_func;
  auto func_obj_ptr_sig = &obj::obj_func;
  ret = func_obj_ptr(4);
  cout << "output = \"" << ret << "\"\n";

  function func_lambda = [](int a) {
    cout << "lambda what = " << a << '\n';
    return a;
  };
  ret = func_lambda(5);
  cout << "output = \"" << ret << "\"\n";
}

int main() {
  test();
  return 0;
}
```

可以看到，可以推导出常见的可调用对象。在本代码中没有提供移动和拷贝构造实现，因为function仅有 shared_ptr 管理的对象，这样我们使用默认的构造和移动函数就可以，全部交给 shared_ptr 管理即可。