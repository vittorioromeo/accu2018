<!-- $size: 16:9 -->

<style>

.live-link
{
    position: absolute; 
    float: left; 
    font-size: 18px; 
    right: 0px; 
    margin-top: -15px; 
    text-align: right;
}

.s 
{
	font-size: 28pt;
    color: black;
}

</style>

<div style="height:100px">

</div>

<span style="text-align: center; font-size: 35.5px; padding: 0; margin: 0">

# `function_ref`

<div style="margin-top: -30px">
  
## (a non-owning reference to a `Callable`)

</div>

</span>

<div style="text-align: center; font-size: 24px; margin-top: 100px">

## Vittorio Romeo

<div style="margin-top: -30px; font-size: 18px">

##### https://vittorioromeo.info

</div>

<div style="margin-top: -25px; font-size: 16px">

##### vittorio.romeo@outlook.com

</div>

</div>

<div style="margin-top: -120px !important">

<span style="text-align: left; padding-top: 30px; padding-left: 13px; zoom:17%">

![](./bloomberg.png)


</span>

<span style="text-align: right; position: absolute; margin-top:-50px; padding-left: 1065px">

**ACCU 2018**

<div style="font-size: 16px; margin-top: -20px !important">

April 2018

</div>

</span>

</div>

<br> <br>

---

<!-- page_number: true -->

<!-- footer: vittorioromeo.info | vittorio.romeo@outlook.com | vromeo5@bloomberg.net | @supahvee1234 -->

<div class='s'>

# C++ is getting *more functional*

</div>

---

<div class='s'>

* C++11 $\rightarrow$ *lambda expressions* and `std::function`

* C++14 $\rightarrow$ *generic lambdas*

* C++17 $\rightarrow$ *`constexpr` lambdas*

</div>

---

<div class='s'>

*Lambda expressions* are syntactic sugar for the definition of *anonymous closure types*
  
</div>

---

<div class='s'>
  
```cpp
auto l = []{ std::cout << "hi!\n"; };
```

$$\downarrow$$

```cpp
struct
{
    auto operator()() const
    {
        std::cout << "hi!\n";
    }
} l;
```
  
</div>

---

<div class='s'>

Even though they're just *syntactic sugar*, lambdas **changed the way we think about code**
  
</div>

---

<div class='s'>

```cpp
const auto benchmark = [](auto f)
{
    const auto time = clock::now();
    f();
    return clock::now() - time;
};
```

```cpp
const auto t = benchmark([]
{
    some_algorithm(/* ... */);
});
```

</div>

---

<div class='s'>
  
```cpp
synchronized<widget> sw;
sw.access([](widget& w)
{
    w.foo();
    w.bar();
});
```
  
</div>

---

<div class='s'>

* *Lambda expressions* make *higher-order functions* **viable** in C++

	* *E.g.* accepting a function as a parameter

	* *E.g.* returning a function from a function

</div>

---

<div class='s'>

> What options do we have to implement *higher-order functions*?

</div>

---

<div class='s'>
  
##### Pointers to functions

```cpp
int operation(int(*f)(int, int)) 
{ 
    return f(1, 2); 
}
```

* Works with *non-member functions* and *stateless closures*

* Doesn't work with *stateful `Callable` objects*

* Small run-time overhead (easily inlined in the same TU)

* Constrained, with obvious signature

</div>

---

<div class='s'>
  
##### Template parameters

```cpp
template <typename T>
auto operation(F&& f) -> decltype(std::forward<F>(f)(1, 2))
{ 
    return std::forward<F>(f)(1, 2); 
}
```

* Works with *any `FunctionObject` or `Callable` with `std::invoke`*

* Zero-cost abstraction

* Hard to constrain

* Might degrade compilation time

</div>

---

<div class='s'>
  
##### `std::function`  

```cpp
int operation(const std::function<int(int, int)>& f) 
{ 
    return f(1, 2); 
}
```

* Works with *any `FunctionObject` or `Callable`*

* Significant run-time overhead (hard to inline/optimize)

* Constrained, with obvious signature

* Unclear semantics: can be both *owning* or *non-owning*

</div>

---

<div class='s'>
  
##### `function_ref`

```cpp
int operation(function_ref<int(int, int)> f) 
{ 
    return f(1, 2); 
}
```

* Works with *any `FunctionObject` or `Callable`*

* Small run-time overhead (easily inlined in the same TU)

* Constrained, with obvious signature

* Clear *non-owning* semantics

* Lightweight - think of "`string_view` for `Callable` objects"

</div>

---

<div class='s'>

## I proposed `function_ref` to LEWG ([P0792](https://wg21.link/p0792))

* **https://wg21.link/p0792**

<br>

## It was sent to LWG without opposition in Jacksonville

* Yay
  
</div>

---

<div class='s'>

> How does it work?

</div>

---

<div class='s'>

"Match" a signature though template specialization:

```cpp
template <typename Signature>
class function_ref;

template <typename Return, typename... Args>
class function_ref<Return(Args...)>
{
    // ...
}
```
  
</div>

---

<div class='s'>

Store *pointer to `Callable` object* and *pointer to erased function*:

```cpp
template <typename Return, typename... Args>
class function_ref<Return(Args...)>
{
private:
    void* _ptr; 
    Return (*_erased_fn)(void*, Args...);

public:
    // ...
};
```

</div>

---

<div class='s'>

On construction, set the pointers:

```cpp
template <typename F>
function_ref(F&& f) noexcept : _ptr{&f}
{
    _erased_fn = [](void* ptr, Args... xs) -> Return 
    {
        return (*reinterpret_cast<F*>(ptr))(
            std::forward<Args>(xs)...);
    };
}
```
  
</div>

---

<div class='s'>

On invocation, go through `_erased_fn`:

```cpp
Return operator()(Args... xs) const
{
    return _erased_fn(_ptr, std::forward<Args>(xs)...);
}
```
  
</div>

---

<!-- footer: -->
<!-- page_number: false -->

```cpp
template <typename Return, typename... Args>
class function_ref<Return(Args...)>
{
    void* _ptr;
    Return (*_erased_fn)(void*, Args...);

public:
    template <typename F, /* ...some constraints... */>
    function_ref(F&& x) noexcept : _ptr{&f}
    {
        _erased_fn = [](void* ptr, Args... xs) -> Return {
            return (*reinterpret_cast<F*>(ptr))(
                std::forward<Args>(xs)...);
        };
    }

    Return operator()(Args... xs) const noexcept(/* ... */)
    {
        return _erased_fn(_ptr, std::forward<Args>(xs)...);
    }
};
```

---

<div class='s'>

In the proposal (**https://wg21.link/p0792**):

* In-depth analysis of the covered techniques' pros/cons

* Synopsis and specification of `function_ref`

* Existing practice *(e.g. LLVM, Folly, `gdb`, ...)*

* Possible issues and open questions

<br>

Article on my blog (**https://vittorioromeo.info**):

* [*"Passing functions to functions"*](https://vittorioromeo.info/index/blog/passing_functions_to_functions.html)

</div>

---

<div class='s'>
<center>
  
# Thanks!

https://wg21.link/p0792
https://vittorioromeo.info

<br>

vittorio.romeo@outlook.com
vromeo5@bloomberg.net

<br>

https://github.com/SuperV1234/accu2018
  
</center>
</div>
