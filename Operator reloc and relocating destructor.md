# Operator reloc and relocating destructor

**Authors:**
Qiuyi Li

**Audience:**
EWG, LEWG

**Project:**

**Abstract**

This paper is talking about a different way from [P2785R3](https://wg21.link/p2785R3), [D2839R1](https://wg21.link/d2839r1) and other similar proposals to enable real relocation in C++. This way cause less change to the core feature than the proposals mentioned above,and is as flexible and powerful as them.To achieve this,we just need a new destructor `T ~T(int)` and an operator `reloc`.

## 1. Overview

In C++11, we have rvalue reference, move constructor and move-assignment operators. Thus, if we want to relocate an object `t` from `&t` to `T* ptr`, we just need to `new(ptr) T(std::move(t))`. Then call the destructor of `T` on `t`. For the object with automatic storage duration this happen automatically.

However, in many cases, the call of the destructor is just useless, and the call of the move constructor is just like a simple memcpy. For the sake of the Zero-overhead rule, we need a new way to solve this problem.

Moreover,sometimes we may want to have a not-null type,that means,the type can’t copy and move,but it’s OK to relocate it.For example,we can define a `not_null_unique_ptr`.<!-- JA: WHY do we want this? At least I (Jiang An) don't want it. -->

The proposal introduces:

- a new destructor `T ~T(int);`

- a new operator `reloc`.

## 2. Proposed language changes

### 2.1 Relocating destructor `T ~T(int)`

Please recall the object model of C++ and think about what will happen when we “relocate” an object from one place to another. Obviously, an object’s lifetime is over and another object’s lifetime begin.What will cause the end of the lifetime of an object? The answer is the calling of its destructor.So now,we introduce the relocating destructor.

For a type `T`, the relocating destructor will return a prvalue with type `T`. Thanks to the copy elision, this prvalue, as a temporary, will be materialized just at where we want “relocate” to,as long as we use placement new or initialized expression to catch this return value.

You may notice that there is a `int` parameter in the declaration of this destructor. That is not important. Just as we use `operator++()` and `operator++(int)` to distinguish between the prefix increment and the postfix increment, we use int to distinguish this from the normal destructor. Typically, we don’t give this `int` a name.

The detail of the definition of the relocating will be talked later in this paper.

### 2.2 Operator `reloc`

In many other proposals, this operator is also defined.

In this paper, the expression reloc(expr) where expr is an unparenthesized id-expression with type T will be treat like this:

- `expr.~T(0)`, if `T` has a relocating destructor,or `T` is trivially-copyable (this will just return expr itself.).

- `[](T& t) {  T ret(std::move(t)) /* used for NRVO, but guaranteed */;  t.~T();  return ret; }(expr)`, if `T` doesn’t have this destructor,and is not trivially-copyable.
  The lambda expression is just for explain.

Normally, identifier `expr` must be an object (not reference,or the “reference” create by structured binding) with automatic storage duration or an array with a [converted constant expression](https://en.cppreference.com/w/cpp/language/constant_expression) of type [`std::size_t`](https://en.cppreference.com/w/cpp/types/size_t) index (for multi-dimensional array, there must be enough indexes to guarantee the type of expr is not an array). But, if in the destructor or relocating destructor of a class, `expr` can also be the member of this class or the directly base class subobject of this class (by using expression like `*(Base*)this` which is obviously point to this subobject).

After using the operator `reloc` on an identifier, the compiler needs to "remember" this variable is destructed and will not destruct it again at the end of the function body. In fact, unless this identifier is an array, the scope of this identifier will be ended.

The rule of the usage of reloc is similar to the chapter 5.1.4 _Early end-of-scope_ in [P2785R3](https://wg21.link/p2785R3).

However, not same to the chapter 5.1.5 _Conditional relocation_ [P2785R3](https://wg21.link/p2785R3), this will result in the calling of the normal destructor of the relocated variable in the unrelocated path.Unless there is another usage of the relocated variable after the `if` statement, the complier will not check whether the destructor should be called later.

### 2.3 The detail of the relocating destructor

#### 2.3.1 The special feature of the relocating destructor

1. In the function body of the relocating destructor or the normal destructor, the (non-static) data member subobject and the base class subobject will just work like the automatic variable and you can relocate it.

2. In the function body of the relocating destructor,we can(but not must) use Designated initialization to construct an object with the same type to try our best to enjoy the benefit of the copy elision. That is, we can `return {/*base class expr*/...,.member1{...},.member2{...},...};`. Even in the non-aggregate class. In addition, we require the compiler to try its best to use NRVO in the relocating destructor.At least, if there is just one declaration of the variable with the same type (ignoring _cv_) before the `return` statement at the same time, and will never return any unnamed value after it,the NRVO will be guaranteed.Here are some examples:

```C++
class T
{
    //...
public:
    T ~T(int)
    {
        if(/*...*/)
        {
            return {/*...*/};//RVO
        }
        if(/*...*/)
        {
            T t{/*...*/};
            //todo,no any other return expressions
            return t;//guarantee NRVO
        }
        if(/*...*/)
        {
            T t{/*...*/}
            if(/*...*/)
            {
                //...
                return t;
            }
            else
            {
                //...
                return t;
            }
            //unreachable,guarantee NRVO
        }
        if(/*...*/)
        {
            T t{/*...*/}
            if(/*...*/)
            {
                //...
                return t;
            }
            else
            {
                //...
                return {/*...*/};
            }
            //no NRVO
        }
        if(/*...*/)
        {
            T t1{/*...*/},t2{/*...*/};
            //...
            return t1;//decided by the complier
        }
    }
}
```

3. In any class `T`, we can define the relocating destructor as `T ~T(int) = default;`. That means it will just use the operator `reloc` to relocate every of its subobject to the new address. Here is an example.

```C++
class A;
class B;
class C;
class D;
class T: A, B
{
    C c;
    D d[3];
    T ~T(int) = default;
    /* This just work like:
    T ~T(int) noexcept(noexcept(std::declval<A>().~A(0))&&...)
    {
        return {reloc(*(A*)this), reloc(*(B*)this), .c = reloc(c), .d = {reloc(d[0]), reloc(d[1]), reloc(d[2])}};
    }
    */
};
```

4. Just like the normal destructor,relocating destructor is `noexcept(true)` by default.

#### 2.3.2 Trivially relocatable

We call a type `T` is trivially relocatable when one of these requirement is satisfied.

- `T` is trivially copyable.

- `T` is a class and all of the subobject of the object with type `T` is trivially relocable and `T` has a default relocating destructor.

- `T` is a union of trivially relocable classes.

Relocating of trivially relocatable class object can be replaced to a `memcpy`.

#### 2.3.3 Virtual relocating destructor

Relocating destructor can be virtual.That means the call of relocating destructor will check the dynamic type,and will throw a `std::bad_relocating` exception when the type is wrong.

### 2.4 Object model

Let `T` be a trivially relocatable but not trivially copyable class, `t` be an object with type `T`, `ptr` be a `T*` point to a storage suitable for creating a `T` object, but not `T` object at the storage is within its lifetime yet. The expression `memcpy(&t, ptr, sizeof(T))` will create a `T` object at `ptr`. But if we use `t` or change the byte between `ptr` and `ptr + 1` not through the pointer to `T` or `T`’s subobject, this object will disappear just as we didn’t create it. If we use this object, the lifetime of `t` will end, then we use t will cause undefined behavior. But use placement new or other method to create object at `&t` is OK,and then we can use `t` again.

When we call the relocating destructor at an object, the lifetime of this object will end after the return value is materialized. That means expression like `new (ptr) T(ptr->~T(42))` is undefined behavior.<!-- JA: Can we just say: the relocation is considered to be happen if it can change any subsequent operation from undefined to well-defined, without introducing new UB? -->

## 2.5 Return statement

Except in a relocating destructor, a return statement uses relocating destructor first if possible.

## 3. Proposed standard library changes

### 3.1 Type traits

The `<type_traits>` library provides some _UnaryTypeTraits_ to tell us the information of a type about relocating.

#### 3.1.1 `std::is_relocatable`

This tells us whether a type has a relocating destructor or is not only move constructible but also destructible.

#### 3.1.2 `std::is_nothrow_relocatable`

This tells us whether a type `T` has a `noexcept` relocating destructor or `std::is_nothrow_move_constructible_v<T> && std::is_nothrow_destructible_v<T> == true`.

#### 3.1.3 `std::is_trivially_relocatable`

This tells us whether a type is trivially relocatable.

#### 3.1.4 `std::has_specialized_relocating_destructor`

This tells us whether a type has a relocating_destructor or is trivially relocatable.

## 3.2 Utility library

There are some changes in `<utility>`.

### 3.2.1 `std::relocating_tag_t`

In relocating destructor, we may define private constructor to make life easier. For convenient, the `<utility>` provide a struct like this.

```C++
struct relocating_tag_t {};
constexpr auto relocating_tag = relocating_tag_t {};
```

### 3.2.2 `std::relocate_to`

The `<utility>` header will provide a function template whose declaration is

```C++
template<class T>
constexpr void relocate_to(T* src, void* dst)
    noexcept(std::is_nothrow_relocatable_v<T> ||
             (std::is_nothrow_move_constructible_v<T> && std::is_nothrow_destructible_v<T>));
```

in `namespace` `std`. The purpose of this function is to relocate an object from `src` to `dst`, choose between the relocate destructor and move-then-destruct automatically.

#### 3.2.3 `std::swap` and `std::exchange`

`std::swap<T>` and `std::exchange<T>` will use relocating destructor and operator `reloc` when `std::is_nothrow_relocatable_v<T> == true`.

#### 3.2.4 `std::pair` and `std::tuple`

`std::pair` and `std::tuple` will have a defaulted relocating destructor.

#### 3.2.5 `std::optional`

If `std::is_trivially_relocatable_v<T> == true`, `std::optional<T>` will be trivially relocatable. In this case, it will have a trivial relocating destructor. If `std::is_trivially_relocatable_v<T> != true`, it will also have a (not defaulted) relocating destructor.
`std::optional<T>` will have a new member function whose declaration is

```C++
constexpr T relocate() noexcept(std::is_nothrow_relocatable_v<T>);
```

if `std::is_relocatable_v<T> == true`. This function will call the relocating destructor on the value inside without check. After the calling of this function, this `optional` will not have value.

#### 3.2.6 `std::variant` and `std::excepted`

For `std::variant<Ty...>`, if `(std::is_trivially_relocatable_v<Ty> && ...) == true`, it will have a trivial relocating destructor, or else, it will have a non-trivial relocating destructor. `std::excepted` is similar.

### 3.3 Double relocating

In the later part of this paper, I will use the statement double relocating frequently. Here is what the meaning of it.

- Double relocate when passing the parameter. For example,we have a function defined like this:

```C++
void d_reloc(T t)
{
    //...
    new (/*dst-ptr*/) T(reloc(t));
}
```

Then, we call this function like `d_reloc(reloc(expr));` or `d_reloc(expr.~T(0));` to pass the parameter. In this way, the relocating will always happen twice.

- Double relocate when return a value. For example, we have a function defined like this:

```C++
T d_reloc()
{
    T tmp = ptr->reloc(0);
    // ...
    return tmp;
}
```

The return statement may cause another relocation if NRVO not happened.

### 3.4 Smart pointers

#### 3.4.1 Relocating destructor of themselves

`std::unique_ptr`, `std::shared_ptr`, and `std::weak_ptr` will have default relocating destructor. For `std::unique_ptr`, if the deleter of it is trivially relocatable, it will also be trivially relocatable. `std::shared_ptr` and `std::weak_ptr` are always trivially relocatable.

#### 3.4.2 Relocating value from `std::unique_ptr`

Firstly, `std::default_delete` will have a new overload of `operator()`, whose declaration is

```C++
void operator()(T*, std::destroy_delete_t).
```

It will choose proper overload of `operator delete` and use it to deallocate the storage without calling the destructor.
Then, if the deleter of `std::unique_ptr` have the overload `void operator(T*, std::destroy_delete_t)`, it will provide a member function called `relocate_out` who use double relocating to relocate the object out and deallocate the storage then set itself empty.

### 3.5 Containers

If possible, every container it self will be trivially relocatable (at least, they will have a relocating destructor), and will provide member function called `relocate_in_xxx` or `relocate_out_xxx` which uses iterator or other way to relocate object into or out of the container by double relocating.

#### 3.5.1 `std::vector`

Let `T` is the element type of the `std::vector`. When `std::is_trivially_relocatable_v<T> == true`, std::vector can use `memcpy` or `memmove` to relocate its elements. If that trait is not `true` but `std::is_nothrow_relocatable_v<T> && std::has_specialized_relocating_destructor_v<T> == true`, `std::vector` will use `std::relocate_to` as possible as it can.<!-- JA: std::deque sometimes also benefits from relocation of elements. -->

<!-- JA: Please be precise about self-referencing containers, e.g., libstdc++'s basic_string and list. -->
`std::vector` will provide a member function called `relocate_out_back` which will relocate out the last element without double relocating.

#### 3.5.2 `std::deque`

`std::deque`’s member function `relocate_out_back` and `relocate_out_front` will never use double relocating.

Like `std::vector`, `std::deque` will also benefit from relocating destructor when the element type is nothrow relocable.

#### 3.5.3 `std::array`

`std::array` will have a default relocating destructor.

## 4. Simple examples

### 4.1 not_null_unique_ptr

```C++
template<class T>
class not_null_unique_ptr
{
    T* _ptr;
public:
    template<class... Args>
    not_null_unique_ptr(Args&&... args):_ptr{new T(std::forward(args...))}
    {
    }
    not_null_unique_ptr(const not_null_unique_ptr&)=delete;
    not_null_unique_ptr(not_null_unique_ptr&&)=delete;
    auto operator=(const not_null_unique_ptr&)=delete;
    auto operator=(not_null_unique_ptr&&)=delete;
    ~not_null_unique_ptr()
    {
        [[assume(_ptr!=nullptr)]];
        delete _ptr;
    }
    not_null_unique_ptr ~not_null_unique_ptr(int)=default;
    const T& operator*()const
    {return *_ptr;}
    T& operator*()
    {return *_ptr;}
    const T* operator->()const
    {return _ptr;}
    T* operator->()
    {return _ptr;}
    void swap(not_null_unique_ptr& other)const
    {
        std::swap(_ptr,other._ptr);
    }
};
```

This class can’t copy or move, but it can relocate and swap.There will never be a nullptr in this class.In fact, this is a strict RAII style smart pointer.We can even put it into the vector, however, we can only take out them by using the relocating member function.Here are some example.

```C++
struct point
{
    size_t x,y;
};
std::array<std::array<size_t,5>,10>screen{};
void show()
{
    for(size_t i=0;i<5;i++)
    {
        if(i!=0)
        {std::cout<<’\n’;}
        for(size_t j=0;j<10;j++)
        {
            std::cout<<screen[j][i]<<’ ’;
        }
    }
}
void display(not_null_unique_ptr<point> p)
{
    screen[p->x][p->y]++;
    //p will be destructed here
}
void consume(std::vector<not_null_unique_ptr<point>>& v)
{
    while(!v.empty())
    {
        //display(v.back());
        //Error!
        display(v.relocate_out_back());
    }
}
int main()
{
    std::vector<not_null_unique_ptr<point>>tmp(10);
    for(size_t i=0;i<10;i++)
    {
        tmp.emplace_back(i,1);
    }
    consume(tmp);
    show();
    std::cout<<”####################\n”;
    for(size_t i=0;i<5;i++)
    {
        not_null_unique_ptr<point> p{i,i};
        //tmp.push_back(p);
        //Error!
        tmp.relocate_in_back(reloc(p));
    }
    consume(tmp);
    show();
    std::cout<<"####################\n";
    std::cout<<"vector size: "<<tmp.size();
}
```

The result of the code is:

```unknown
0 0 0 0 0 0 0 0 0 0
1 1 1 1 1 1 1 1 1 1
0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0
####################
1 0 0 0 0 0 0 0 0 0
1 2 1 1 1 1 1 1 1 1
0 0 1 0 0 0 0 0 0 0
0 0 0 1 0 0 0 0 0 0
0 0 0 0 1 0 0 0 0 0
####################
vector size: 0
```

### 4.2 (simple) self-reference list

A list may use self-reference to implement the end iterator.Here is the destructor of it.

```C++
//We assume the list node is an array of the struct 
//that only contain a pointer.
//struct node_t{node_t* next};
//First two pointer in the array is the next and the previous pointer.
//The other pointer’s storage is reused to store object.
template<class T>
class my_list
{
    size_t _size;
    node_t _head_and_tail[3];
public:
    my_list():_head_and_tail{_head_and_tail+1,nullptr,_head_and_tail}
        //self-reference
    {/*...*/}
    //...
private:
    my_list(std::relocating_tag_t)noexcept
    {/*do nothing*/}
public:
    my_list ~my_list(int)
    {
        if(empty())
        {
            return {};//RVO
        }
        my_list m{std::relocating_tag};//NRVO
        memcpy(this,&m,sizeof(my_list));
        *(_head_and_tail[0]+1)=+m._head_and_tail;
        *(_head_and_tail[2])=m.head_and_tail+1;
        //reset the first and last node
        return m;
    }
};
```

### 4.3 (simple) self-reference string

Sometimes, self-reference also exist in the string object.

```C++
class my_string
{
    struct blk_t
    {
        char* _cap;
        char* _end;
    };
    union
    {
        char sso[16];
        blk_t blk;
    };
    char* _begin;
    my_string(std::relocating_tag_t)
    {
        //do nothing
    }
public:
    //...
    my_string ~my_string(int)
    {
        my_string ret{std::relocating_tag};//NRVO
        memcpy(this,&ret,sizeof(my_string));
        if(_begin==this)
        {
            ret._begin=&ret;
        }
        return ret;
    }
};
```

### 4.4 Register object

Here is a simple demo of a class register itself into a global `std::unordered_map` .

```C++
class my_class
{
    inline static std::unordered_map<size_t,my_class*> mp{};
    inline static size_t unused_id{};
    size_t id;
    //...
public:
    static const std::unordered_map<size_t,my_class*>& get_all_objects()
    {
        return mp;
    }
    my_class():id{unused_id}
    {
        unused_id++;
        mp[id]=this;
        //...
    }
    //...
    my_class(my_class&& other):id{unused_id}/*noexcept(false)*/
    {
        unused_id++;
        mp[id]=this;
        //...
    }
    ~my_class()
    {
        mp.erase(id);
        //...
    }
    my_class ~my_class(int)/*noexcept*/
    {
        my_class ret = {.id=id,/*...*/};
        mp[id]= &ret;
        return ret;
    }
}
```
