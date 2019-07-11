# cppmetamagic
Some general purpose metaprogramming tips&amp;tricks in C++
 

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

Let's modify final indexSequence struct for better debug output. Just like this:
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

# 4. Determine if a type contains a certain func  (SFINAE)

**Example from Fedor Pikus CppCon 2015 presentation**

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
