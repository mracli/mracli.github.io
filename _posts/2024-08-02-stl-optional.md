---
title: stl-optional
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
date: 2024-08-02 17:13 +0800
---
最近使用 optional 容器，觉得其可以实现返回一个 T 类型或 nullopt 类型的操作挺有意思，这里参考一些内容重新复现该容器。

```c++
// 重新封装 optional 类的异常
class BadOptionalAccess : public std::exception {
public:
  BadOptionalAccess() = default;
  virtual ~BadOptionalAccess() = default;

  const char *what() const noexcept override { return "BadOptionalAccess"; }
};

// 作为无内容时的返回
struct nullopt_t {
  explicit nullopt_t() = default;
};

inline struct nullopt_t nullopt;

/* 
 构造 Optional 大部分都是将外部需要存储的 T 类对象/变量 移动到内部

 且构造过程中将移动进的变量进行 placement new 减少一次构造的成本，要记住这里 new 后需要手动析构

 这里面需要注意的是 Optional 重载了乘号运算符，导致直接类内直接使用*可能是地址取值操作，
 所以需要使用 std::addressof(·) 算子取地址

 */
template <class T> struct Optional {
private:
  bool m_has_value;
  union {
    T m_value;
  };

public:
  Optional() : m_has_value(false) {};

  Optional(nullopt_t) noexcept : m_has_value(false) {};

  Optional(T &&value) : m_has_value(true), m_value(std::move(value)) {};

  Optional(Optional &&that) noexcept : m_has_value(that.has_value()) {
    if (m_has_value) {
      new (std::addressof(m_value)) T(std::move(that.m_value));
    }
  }

  Optional(Optional const &that) : m_has_value(that.has_value()) {
    if (m_has_value) {
      new (std::addressof(m_value)) T(that.value());
    }
  }

  Optional &operator=(nullopt_t) noexcept {
    if (m_has_value) {
      m_value.~T();
      m_has_value = false;
    }
    return *this;
  }

  Optional &operator=(T &&that) noexcept {
    if (m_has_value) {
      m_value.~T();
    }
    new (std::addressof(m_value)) T(std::move(that));
    m_has_value = true;
    return *this;
  }

  Optional &operator=(Optional const &that) {
    if (this == &that)
      return *this;
    if (m_has_value) {
      m_value.~T();
      m_has_value = false;
    }
    m_has_value = that.has_value();
    m_value = that.value();
    return *this;
  }

  Optional &operator=(Optional &&that) noexcept {
    if (this == &that)
      return *this;
    if (m_has_value) {
      m_value.~T();
      m_has_value = false;
    }
    m_has_value = that.has_value();
    if (m_has_value) {
      new (std::addressof(m_value)) T(std::move(that.value()));
      that.m_value.~T();
      that.m_has_value = false;
    }
    return *this;
  }

  template <class... Args>
    requires(std::is_constructible_v<T, Args...>)
  void emplace(Args &&...args) {
    if (has_value()) {
      m_value.~T();
    }
    new (std::addressof(m_value)) T(std::forward<Args>(args)...);
    m_has_value = true;
  }

  void reset() noexcept {
    if (m_has_value) {
      m_value.~T();
      m_has_value = false;
    }
  }

  bool has_value() const noexcept { return m_has_value; }

  explicit operator bool(){
    return has_value();
  }

  T &&value() && {
    if (!has_value()) [[unlikely]]
      throw BadOptionalAccess();
    return std::move(m_value);
  }
  T const &&value() const && {
    if (!has_value()) [[unlikely]]
      throw BadOptionalAccess();
    return std::move(m_value);
  }
  T &value() & {
    if (!has_value()) [[unlikely]]
      throw BadOptionalAccess();
    return m_value;
  }

  T const &value() const & {
    if (!has_value()) [[unlikely]]
      throw BadOptionalAccess();
    return m_value;
  }

  T value_or(T default_value) const & noexcept(std::is_copy_assignable_v<T>) {
    if (!has_value())
      return default_value;
    return m_value;
  }

  T &&
  value_or(T default_value) && noexcept(std::is_nothrow_move_assignable_v<T>) {
    if (!has_value())
      return default_value;
    return std::move(value());
  }

  T const &operator*() const & noexcept { return m_value; }

  T &operator*() & noexcept { return m_value; }
  T const &&operator*() const && noexcept { return m_value; }
  T &&operator*() && noexcept { return m_value; }

  T const *operator->() const noexcept { return std::addressof(m_value); }
  T *operator->() noexcept { return std::addressof(m_value); }

  ~Optional() noexcept {
    if (has_value())
      m_value.~T();
  }
};
```