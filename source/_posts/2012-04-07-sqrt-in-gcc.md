---
layout: post
title: gcc中sqrt實現
author: MaskRay
tags: [c++, haskell]
---

對於`sqrt(3);`，`Visual Studio`會報告有歧義的重載函數調用，而`gcc`卻能把這個解釋爲對`sqrt(double)`的調用。研究了一下`gcc`的實現，發現`sqrt`其實是《C++ Templates: The Complete Guide》中提到的`type function`，使用了名爲`SFINAE (substitution failure is not an error)`的技術。這裏是給出一個使用該技術的`sqrt`的簡易實現：

<!-- more -->

    #include <cmath>
    #include <cstdio>
    using namespace std;

    template<typename T>
    struct is_int { enum {type = 0}; };

    template<>
    struct is_int<int> { enum {type = 1}; };

    template<typename T, bool>
    struct enable_if {};

    template<typename T>
    struct enable_if<T, true> { typedef T type; };

    template<typename T>
    typename enable_if<double, is_int<T>::type>::type sqr(T a)
    { return sqrt(double(a)); }

    double sqr(double a) { return sqrt(a); }
    long double sqr(long double a) { return sqrt(a); }
    float sqr(float a) { return sqrt(a); }

    int main()
    {
        printf("%lf\n", sqr(3));
        // compile error: printf("%lf\n", sqr((char)3));
        printf("%lf\n", sqr(3.));
        printf("%lf\n", sqr(3.f));
    }

可見C++的實現相當繁瑣。這也算是`partial template specialization`的應用，`Haskell`的`type class`使用`closed-world assumption`，所以沒有直接的對應物，但我們可以給`Integral`做一個wrapper：

```haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
class Sqrt a where
  sqrt_ :: a -> a

instance Sqrt Double where
  sqrt_ = sqrt

newtype WrappedIntegral a = WrappedIntegral a deriving (Num, Show)

instance Integral a => Sqrt (WrappedIntegral a) where
  sqrt_ = const 17
```
