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

**Explanation 2** [Source: https://stackoverflow.com/a/27688405]

When you write has_member<A>::value, the compiler looks up the name has_member and finds the primary class template, that is, this declaration:
```
template< class , class = void >
struct has_member;
```

The template argument list <A> is compared to the template parameter list of this primary template. Since the primary template has two parameters, but you only supplied one, the remaining parameter is defaulted to the default template argument: void. It's as if you had written has_member<A, void>::value.

Now, the template parameter list is compared against any specializations of the template has_member. Only if no specialization matches, the definition of the primary template is used as a fall-back. So the partial specialization is taken into account:
```
template< class T >
struct has_member< T , void_t< decltype( T::member ) > > : true_type
{ };
```
The compiler tries to match the template arguments A, void with the patterns defined in the partial specialization: T and void_t<..> one by one. First, template argument deduction is performed. The partial specialization above is still a template with template-parameters that need to be "filled" by arguments.

The first pattern, T, allows the compiler to deduce the template-parameter T. This is a trivial deduction, but consider a pattern like T const&, where we could still deduce T. For the pattern T and the template argument A, we deduce T to be A.

In the second pattern void_t< decltype( T::member ) >, the template-parameter T appears in a context where it cannot be deduced from any template argument. There are two reasons for this:

The expression inside decltype is explicitly excluded from template argument deduction. I guess this is because it can be arbitrarily complex.

Even if we used a pattern without decltype like void_t< T >, then the deduction of T happens on the resolved alias template. That is, we resolve the alias template and then try to deduce the type T from the resulting pattern. The resulting pattern however is void, which is not dependent on T and therefore does not allow us to find a specific type for T. This is similar to the mathematical problem of trying to invert a constant function (in the mathematical sense of those terms).

Template argument deduction is finished(*), now the deduced template arguments are substituted. This creates a specialization that looks like this:
```
template<>
struct has_member< A, void_t< decltype( A::member ) > > : true_type
{ };
```
The type void_t< decltype( A::member ) > > can now be evaluated. It is well-formed after substitution, hence, no Substitution Failure occurs. We get:
```
template<>
struct has_member<A, void> : true_type
{ };
```
Now, we can compare the template parameter list of this specialization with the template arguments supplied to the original has_member<A>::value. Both types match exactly, so this partial specialization is chosen.

On the other hand, when we define the template as:
```
template< class , class = int > // <-- int here instead of void
struct has_member : false_type
{ };

template< class T >
struct has_member< T , void_t< decltype( T::member ) > > : true_type
{ };
```
We end up with the same specialization:
```
template<>
struct has_member<A, void> : true_type
{ };
```
but our template argument list for has_member<A>::value now is <A, int>. The arguments do not match the parameters of the specialization, and the primary template is chosen as a fall-back.

(*) The Standard, IMHO confusingly, includes the substitution process and the matching of explicitly specified template arguments in the template argument deduction process. For example (post-N4296) [temp.class.spec.match]/2:

A partial specialization matches a given actual template argument list if the template arguments of the partial specialization can be deduced from the actual template argument list.

But this does not just mean that all template-parameters of the partial specialization have to be deduced; it also means that substitution must succeed and (as it seems?) the template arguments have to match the (substituted) template parameters of the partial specialization. Note that I'm not completely aware of where the Standard specifies the comparison between the substituted argument list and the supplied argument list.

# 5. Unpacking tuples 
**[Source: http://aherrmann.github.io/programming/2016/02/28/unpacking-tuples-in-cpp14/]**

Suppose we wanted to implement a function that takes an arbitrary tuple and returns a new tuple that holds the first N elements of the original tuple. Let’s call it take_front. Since tuples have fixed size the parameter N will have to be a template parameter.
```
template <class Tuple, size_t... Is>
constexpr auto take_front_impl(Tuple t,
                               index_sequence<Is...>) {
    return make_tuple(get<Is>(t)...);
}

template <size_t N, class Tuple>
constexpr auto take_front(Tuple t) {
    return take_front_impl(t, make_index_sequence<N>{});
}
```
The func­tion take_front_impl takes the input tuple and an index_sequence. As be­fore that second pa­ra­meter is only there so that we can get our hands on a pa­ra­meter pack of in­dices. We then use these in­dices to get the el­e­ments of the input tuple and pass them to make_tuple which will con­struct the re­sult. How­ever, at that point we haven’t ac­tu­ally de­fined, yet, which el­e­ments should be put into that new tu­ple. This hap­pens within take_front, which con­structs an index-se­quence con­sisting of the in­dices 0 to N-1 and passes it to take_front_impl.

We can use that func­tion like so.
```
    auto t = take_front<2>(make_tuple(1, 2, 3, 4));
    assert(t == make_tuple(1, 2));
```

