# Some C++ metaprogramming tips&tricks
 

# 1. Compile part of code if template parameter is true

enable_if Implementation:
```cpp
template<bool, typename U = void>
struct enable_if
{};

template<typename T>
struct enable_if<true, T> 	// partial specialization
{
    using type = T;
};
```
 
 
# 2. Type checking at compile time

IsInt implementation:

```cpp
template<typename T>
struct isInt
{
    static constexpr bool value = false;
};

template<>
struct isInt <int>	// specialization
{
    static constexpr bool value = true;
};
```
Let's mix some code above and create something useful...

**Usage:**
```cpp
template<typename U>
using Type = typename enable_if< isInt<U>::value, U >::type;

int main()
{
	Type<int> test;		// OK
	Type<float> test;	// Compile error!
}
```


# 3. Generate sequences of numbers at compile time

Index sequence implementation:
```cpp
template <std::size_t ...>
struct indexSequence
{ };

template <std::size_t ... Next>
struct indexSequenceHelper<0U, Next ... >   // partial specialization
{ 
    using type = indexSequence<Next ... >; 
};

template <std::size_t N, std::size_t ... Next>
struct indexSequenceHelper : public indexSequenceHelper<N-1U, N-1U, Next...>
{ };

template <std::size_t N>
using makeIndexSeq = typename indexSequenceHelper<N>::type;
```

**Usage:**

Let's modify final indexSequence struct for better debug output:
```cpp
template <std::size_t ...>
struct indexSequence
 { 
    indexSequence()
    { 
    	std::cout << __PRETTY_FUNCTION__ << std::endl; 
    }
 };
 
 ...
 
 ```
 ... and test it
```
int main()
{    
    makeIndexSeq<5> test;
}
```
**Output:**

*indexSequence<<anonymous> >::indexSequence() [with long unsigned int ...\<anonymous\> = {0ul, 1ul, 2ul, 3ul, 4ul}]*
	

We can see, point to point, how makeIndexSequence<3> become index_sequenxe<0, 1, 2>.

We have that makeIndexSequence<3> is defined as typename indexSequenceHelper<3>::type [N is 3]


- indexSequenceHelper<3> match only the general case so

  inherit from indexSequenceHelper<2, 2> [N is 3 and Next... is empty]


- indexSequenceHelper<2, 2> match only the general case so 

  inherit from indexSequenceHelper<1, 1, 2> [N is 2 and Next... is 2]


- indexSequenceHelper<1, 1, 2> match only the general case so 

  inherit from indexSequenceHelper<0, 0, 1, 2> [N is 1 and Next... is 1, 2]


indexSequenceHelper<0, 0, 1, 2> match both case (general an partial specialization) so 

the partial specialization is applied and define type = indexSequence<0, 1, 2> [Next... is 0, 1, 2]


[ Description taken from: https://stackoverflow.com/a/49672613 ]



# 4. Determine if a type contains a certain func  (SFINAE)

**Example from Fedor Pikus CppCon 2015 presentation**
[ Source: https://www.youtube.com/watch?v=CZi6QqZSbFg ]


When T has the sort function defined, the instantiation of the first test works and the null pointer constant is successfully passed. (And the resulting type of the expression is yes.) If it does not work, the only available function is the second test, and the resulting type of the expression is no
```cpp
template<typename T>
struct HasSort
{
    using yes = char;
    using no = char[2];
    
    template<typename U>
    static auto test(void*) -> decltype(&U::sort, yes());    	// C++11
    //static yes& test( char(*)[sizeof(&U::sort)] );		// C++98

    template<typename U>
    static no& test(...);
    
    static constexpr bool value = ( sizeof( HasSort::test<T>(nullptr) ) == sizeof( HasSort::yes ) );

};
```
**Usage:**
```cpp
    struct Sortable
    {
   	void sort();
    };
    
    ...
    
    std::cout << HasSort<int>::value << std::endl;
    std::cout << HasSort<Sortable>::value << std::endl;
```

**Output:**

0 <- **no conversion, lack of T::sort(), accept any argument test(...) function**

1 <- **conversion rank is lowest, so a call to the first test() function will be preferred if it is possible**

# 4a SFINAE vs argument deduction

Consider another exmple and its result.

```cpp
struct Test
{
    template<typename U>
    static int func( char(*)[sizeof(U)] )
    { std::cout << __PRETTY_FUNCTION__ << std::endl; }
    
    template<typename U>
    static int func( ...)
    { std::cout << __PRETTY_FUNCTION__ << std::endl; }
};

...

int main()
{
    Test::func<int>(nullptr);
    Test::func<int>(5);
}
```

**Output:**

*static int Test::func(char (*)[sizeof (U)]) [with U = int]*

*static int Test::func(...) [with U = int]* 


# 4b Determine if a class contains a certain type
We can use helper types from type_traits
```cpp
#include <type_traits>


template<typename T>
struct Void
{
    using type = void;   
};

template<typename T, typename = void>
struct has_type : std::false_type { };

template<typename T>
struct has_type<T, typename Void<typename T::foobar>::type > : std::true_type { };

struct foo {
  using foobar = float;
};

int main() 
{   
  std::cout << std::boolalpha;
  std::cout << has_type<int>::value << std::endl;	// Output: false
  std::cout << has_type<foo>::value << std::endl;	// Output: true
}
```
**Explanation 1 [ Source: https://stackoverflow.com/a/27687803 ]**
```cpp
template<typename T>
struct has_type<T, typename Void<typename T::foobar>::type > : std::true_type { };
```

That above specialization exists only when it is well formed, so when decltype( T::member ) is valid and not ambiguous. the specialization is so for has_member<T , void> as state in the comment.

When you write has_type<A>, it is has_member<A, void> because of default template argument.

And we have specialization for has_type<A, void> (so inherit from true_type) but we don't have specialization for has_type<B, void> (so we use the default definition : inherit from false_type

**Explanation 2**
