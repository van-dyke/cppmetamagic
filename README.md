# Some C++ metaprogramming tips&tricks
 

# 1a. Compile part of code if template parameter is true

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

# 1b. Conditional type selection
```cpp
template<bool, typename U, typename T>
struct condition
{
    using type = U;
};

template<typename U, typename T>
struct condition<true, U, T>
{
    using type = T;
};
```
another approach:
```cpp
template<bool>
struct cond
{
    template<typename T, typename U>
    using type = U;
};

template<>
struct cond<true>
{
    template<typename T, typename U>
    using type = T;
};
```
**Usage:**
```cpp
    std::cout<< sizeof( condition<true, int, char>::type ) << std::endl;	//Output: 1
    std::cout<< sizeof( condition<false, int, char>::type ) << std::endl;	//Output: 4
    
    std::cout<< sizeof( cond<true>::type<int, char> ) << std::endl;		//Output: 1
    std::cout<< sizeof( cond<false>::type<int, char> ) <<std::endl;		//Output: 4
```

# 1c. enable_if - a compile-time switch for templates
***enable_if*** is an extremely useful tool. There are hundreds of references to it in the C++11 standard template library. It's so useful because it's a key part in using type traits, a way to restrict templates to types that have certain properties. Without ***enable_if***, templates are a rather blunt "catch-all" tool. If we define a function with a template argument, this function will be invoked on all possible types. Type traits and enable_if let us create different functions that act on different kinds of types, while still remaining generic 

```cpp
#include <iostream>
#include <string>
#include <type_traits>

template <typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
t()
{ return 1; }

template <typename T>
typename std::enable_if<!std::is_integral<T>::value, T>::type
t()
{ return 0; }

template<typename T,
         typename std::enable_if<std::is_integral<T>::value, int>::type = 0>
auto f() { return 1; }

template<typename T,
         typename std::enable_if<!std::is_integral<T>::value, int>::type = 0>
auto f() { return 0; }


int main()
{
    std::cout << t<int>() << std::endl;		//Output: 1
    std::cout << t<float>() << std::endl;	//Output: 0
    
    std::cout << f<int>() << std::endl;		//Output: 1
    std::cout << f<float>() << std::endl;	//Output: 0
        
}
```
This line
```cpp
template<typename T,
         typename std::enable_if<std::is_integral<T>::value, int>::type = 0>
```
declares a named template type parameter T and a nameless template value parameter of type ***std::enable_if<std::is_integral<T>::value, int>::type***, i.e. the second parameter declarations is basically a more convoluted version of simple

```cpp
template <int N = 0> struct S {};
```

The type ***std::enable_if<std::is_integral<T>::value, int>::type*** resolves into type ***int*** when the enabling condition is met, meaning that the whole thing in such cases is equivalent to

```cpp
template<typename T, int = 0>
```

However, since type of that second template parameter is a nested type of a dependent template lfEnableIf, you are required to use the keyword typename to tell the compiler that member Type actually refers to a type and not to something else (i.e. to disambiguate the situation).

Again, the second parameter of the template is nameless, but you can give it a name, if you wish. It won't change anything

```cpp
template<typename T,
         typename std::enable_if<std::is_integral<T>::value, int>::type V = 0>
```

In the above example I called it V. The name of that parameter is not used anywhere in the template, which is why there's no real need to specify it explicitly. It is a dummy parameter, which is also the reason it has a dummy default value (you can replace 0 with 42 - it won't change anything either).

In this case keyword typename creates a misleading similarity between the two parameter declarations of your template. In reality in these parameter declarations the keyword typename serves two very very very different unrelated purposes.

In the first template parameter declaration - typename T - it declares T as a template type parameter. In this role keyword typename can be replaced with keyword class

```cpp
template <class T, ...
```

In the second declaration - ***typename std::enable_if<std::is_integral<T>::value, int>::type*** - it serves a secondary purpose - it just tells the compiler that ***std::enable_if<std::is_integral<T>::value, int>::type*** is a type and thus turns the whole thing into a value parameter declaration. In this role keyword typename cannot be replaced with keyword class.
	
 
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

# 4a Determine if a type contains a certain func  (SFINAE) v2.0

Playing with SFINAE for method detection in C++11 - another working version

This solution use std::decval<T>()
It converts any type T to a reference type, making it possible to use member functions in decltype expressions without the need to go through constructors.

```cpp
template <typename T, typename ENABLE = void>
struct Detect_My_Method : std::false_type
{
};

template <typename T>
struct Detect_My_Method<T, decltype(std::declval<T>().Method())> : std::true_type
{
};

// ...

class Foo
{
    public:
        void Method() {};
};

std::cout << Detect_My_Method<Foo>::value << std::endl;		//Output: 1
```


# 4b SFINAE vs argument deduction

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


# 4c Determine if a class contains a certain type
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
```cpp
template< class , class = void >
struct has_member;
```

The template argument list <A> is compared to the template parameter list of this primary template. Since the primary template has two parameters, but you only supplied one, the remaining parameter is defaulted to the default template argument: void. It's as if you had written has_member<A, void>::value.

Now, the template parameter list is compared against any specializations of the template has_member. Only if no specialization matches, the definition of the primary template is used as a fall-back. So the partial specialization is taken into account:
```cpp
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
```cpp
template<>
struct has_member< A, void_t< decltype( A::member ) > > : true_type
{ };
```
The type void_t< decltype( A::member ) > > can now be evaluated. It is well-formed after substitution, hence, no Substitution Failure occurs. We get:
```cpp
template<>
struct has_member<A, void> : true_type
{ };
```
Now, we can compare the template parameter list of this specialization with the template arguments supplied to the original has_member<A>::value. Both types match exactly, so this partial specialization is chosen.

On the other hand, when we define the template as:
```cpp
template< class , class = int > // <-- int here instead of void
struct has_member : false_type
{ };

template< class T >
struct has_member< T , void_t< decltype( T::member ) > > : true_type
{ };
```
We end up with the same specialization:
```cpp
template<>
struct has_member<A, void> : true_type
{ };
```
but our template argument list for has_member<A>::value now is <A, int>. The arguments do not match the parameters of the specialization, and the primary template is chosen as a fall-back.

(*) The Standard, IMHO confusingly, includes the substitution process and the matching of explicitly specified template arguments in the template argument deduction process. For example (post-N4296) [temp.class.spec.match]/2:

A partial specialization matches a given actual template argument list if the template arguments of the partial specialization can be deduced from the actual template argument list.

But this does not just mean that all template-parameters of the partial specialization have to be deduced; it also means that substitution must succeed and (as it seems?) the template arguments have to match the (substituted) template parameters of the partial specialization. Note that I'm not completely aware of where the Standard specifies the comparison between the substituted argument list and the supplied argument list.

# 5a. Unpacking tuples 
**[Source: http://aherrmann.github.io/programming/2016/02/28/unpacking-tuples-in-cpp14/]**

Suppose we wanted to implement a function that takes an arbitrary tuple and returns a new tuple that holds the first N elements of the original tuple. Let’s call it take_front. Since tuples have fixed size the parameter N will have to be a template parameter.
```cpp
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
```cpp
    auto t = take_front<2>(make_tuple(1, 2, 3, 4));
    assert(t == make_tuple(1, 2));
```
***Example of usage ( custom tuple printer ):***
```cpp
template<typename T>
void tprintf(T t)                   // base function
{
    std::cout << t << std::endl;
}

template<typename T, typename... Targs>
void tprintf(T value, Targs... Fargs)
{
     std::cout << value << std::endl;
     tprintf(Fargs...); 
}

template<typename Tup, size_t... Is >
void unpackHelper(Tup t, std::index_sequence<Is...>)
{
    tprintf(std::get<Is>(t)...);
}

template<typename Tup>
void printTuple(Tup t)
{
    unpackHelper(t, std::make_index_sequence<std::tuple_size<Tup>::value>() );
}
```
And invoke:
```cpp
    std::tuple<int, bool, float> t{100, 1, 15.0f};
    printTuple(t);
```    

# 5b. Unpacking tuples - another approach
**[Source: https://stackoverflow.com/questions/16387354/template-tuple-calling-a-function-on-each-element]**

Given a meta-function gen_seq for generating compile-time integer sequences (encapsulated by the seq class template):
```cpp
namespace detail
{
    template<int... Is>
    struct seq { };

    template<int N, int... Is>
    struct gen_seq : gen_seq<N - 1, N - 1, Is...> { };

    template<int... Is>
    struct gen_seq<0, Is...> : seq<Is...> { };
}
```

And the following function templates:

```cpp
namespace detail
{
    template<typename T, typename F, int... Is>
    void for_each(T&& t, F f, seq<Is...>)
    {
        auto l = { (f(std::get<Is>(t)), 0)... };
    }
}

template<typename... Ts, typename F>
void for_each_in_tuple(std::tuple<Ts...> const& t, F f)
{
    detail::for_each(t, f, detail::gen_seq<sizeof...(Ts)>());
}
```
You can use the for_each_in_tuple function above this way:

```cpp
struct my_functor
{
    template<typename T>
    void operator () (T&& t)
    {
        std::cout << t << std::endl;
    }
};

int main()
{
    std::tuple<int, double, std::string> t(42, 3.14, "Hello World!");
    for_each_in_tuple(t, my_functor());		//Output: 42 3.14 Hello World
}
```

# 5c. Unpacking tuples - another approach II
**[Source: https://eli.thegreenplace.net/2014/variadic-templates-in-c/]**

Custom data structures (structs since the times of C and classes in C++) have compile-time defined fields. They can represent types that grow at runtime (std::vector, for example) but if you want to add new fields, this is something the compiler has to see. Variadic templates make it possible to define data structures that could have an arbitrary number of fields, and have this number configured per use. The prime example of this is a tuple class, and here I want to show how to construct one [4].

For the full code that you can play with and compile on your own: variadic-tuple.cpp.

Let's start with the type definition:

```cpp
template <class... Ts> struct tuple {};

template <class T, class... Ts>
struct tuple<T, Ts...> : tuple<Ts...> {
  tuple(T t, Ts... ts) : tuple<Ts...>(ts...), tail(t) {}

  T tail;
};
```
We start with the base case - the definition of a class template named tuple, which is empty. The specialization that follows peels off the first type from the parameter pack, and defines a member of that type named tail. It also derives from the tuple instantiated with the rest of the pack. This is a recursive definition that stops when there are no more types to peel off, and the base of the hierarchy is an empty tuple. To get a better feel for the resulting data structure, let's use a concrete example:
```cpp
tuple<double, uint64_t, const char*> t1(12.2, 42, "big");
```
Ignoring the constructor, here's a pseudo-trace of the tuple structs created:

```cpp
struct tuple<double, uint64_t, const char*> : tuple<uint64_t, const char*> {
  double tail;
}

struct tuple<uint64_t, const char*> : tuple<const char*> {
  uint64_t tail;
}

struct tuple<const char*> : tuple {
  const char* tail;
}

struct tuple {
}
```
The layout of data members in the original 3-element tuple will be:

```cpp
[const char* tail, uint64_t tail, double tail]
```
Note that the empty base consumes no space, due to empty base optimization. Using Clang's layout dump feature, we can verify this:

Dumping AST Record Layout

   0 |-struct tuple<double, unsigned long, const char *>
   
   0 |----struct tuple<unsigned long, const char *> (base)
   
   0 |------struct tuple<const char *> (base)
   
   0 |---------struct tuple<> (base) (empty)
   
   0 |---------const char * tail
   
   8 |------unsigned long tail
   
  16 |----double tail
  
   sizeof=24, dsize=24, align=8
   nvsize=24, nvalign=8
     
Indeed, the size of the data structure and the internal layout of members is as expected.

So, the struct definition above lets us create tuples, but there's not much else we can do with them yet. The way to access tuples is with the get function template [5], so let's see how it works. First, we'll have to define a helper type that lets us access the type of the k-th element in a tuple:
```cpp
template <size_t, class> struct elem_type_holder;

template <class T, class... Ts>
struct elem_type_holder<0, tuple<T, Ts...>> {
  typedef T type;
};

template <size_t k, class T, class... Ts>
struct elem_type_holder<k, tuple<T, Ts...>> {
  typedef typename elem_type_holder<k - 1, tuple<Ts...>>::type type;
};
```
***elem_type_holder*** is yet another variadic class template. It takes a number k and the tuple type we're interested in as template parameters. Note that this is a compile-time template metaprogramming construct - it acts on constants and types, not on runtime objects. For example, given ***elem_type_holder<2, some_tuple_type>***, we'll get the following pseudo expansion:
```cpp
struct elem_type_holder<2, tuple<T, Ts...>> {
  typedef typename elem_type_holder<1, tuple<Ts...>>::type type;
}

struct elem_type_holder<1, tuple<T, Ts...>> {
  typedef typename elem_type_holder<0, tuple<Ts...>>::type type;
}

struct elem_type_holder<0, tuple<T, Ts...>> {
  typedef T type;
}
```
So the ***elem_type_holder<2, some_tuple_type>*** peels off two types from the beginning of the tuple, and sets its type to the type of the third one, which is what we need. Armed with this, we can implement get:
```cpp
template <size_t k, class... Ts>
typename std::enable_if<
    k == 0, typename elem_type_holder<0, tuple<Ts...>>::type&>::type
get(tuple<Ts...>& t) {
  return t.tail;
}

template <size_t k, class T, class... Ts>
typename std::enable_if<
    k != 0, typename elem_type_holder<k, tuple<T, Ts...>>::type&>::type
get(tuple<T, Ts...>& t) {
  tuple<Ts...>& base = t;
  return get<k - 1>(base);
}
```
Here, enable_if is used to select between two template overloads of get - one for when k is zero, and one for the general case which peels off the first type and recurses, as usual with variadic function templates.

Since it returns a reference, we can use get to both read tuple elements and write to them:

```cpp
tuple<double, uint64_t, const char*> t1(12.2, 42, "big");

std::cout << "0th elem is " << get<0>(t1) << "\n";
std::cout << "1th elem is " << get<1>(t1) << "\n";
std::cout << "2th elem is " << get<2>(t1) << "\n";

get<1>(t1) = 103;
std::cout << "1th elem is " << get<1>(t1) << "\n";
```


# 6. Getting the nth-arg 
[Source: https://medium.com/@LoopPerfect/c-17-vs-c-14-if-constexpr-b518982bb1e2]

Many template meta-programs operate on variadic-type-lists. In C++ 14, getting the nth-type of an argument lists is often implemented the following way:

```cpp
template<unsigned n>
struct Arg {
 template<class X, class…Xs>
 constexpr auto operator()(X x, Xs…xs) {
   return Arg<n-1>{}(xs…);
 }
};
template<>
struct Arg<0> {
 template<class X, class…Xs>
 constexpr auto operator()(X x, Xs…) {
   return x;
 }
};
template<unsigned n>
constexpr auto arg = Arg<n>{};

// arg<2>(0,1,2,3,4,5) == 2;

```

# 7. API — shimming 
[Source: https://medium.com/@LoopPerfect/c-17-vs-c-14-if-constexpr-b518982bb1e2] + my improvements...

[***Note:*** There is no need to pass object T into ***supportsAPI*** function and invoke later its method. ]

Sometimes you want to support an alternative API. C++ 14 provides an easy way to check if an object can be used in a certain way:

```cpp
template<class T>
constexpr auto supportsAPI(void*) -> decltype(&T::Method1,  std::true_type{}) 
{
	return {};
}

template<class T>
constexpr auto supportsAPI(...) -> std::false_type 
{
	return {};
}

// ...
struct One
{
	int Method1() { return 100; };
};

struct Two
{
    
};

std::cout << std::boolalpha;
std::cout << supportsAPI<One>(nullptr) << std::endl;	//Output: true
std::cout << supportsAPI<Two>(nullptr) << std::endl;	//Output: false

```
Implementing custom behaviour in C++ 14 can be done like this:

```cpp
template<class T>
auto compute(T x) -> decltype( std::enable_if_t< supportsAPI<T>(nullptr), int>{}) 
{
	return x.Method1();
}

template<class T>
auto compute(T x) -> decltype( std::enable_if_t<!supportsAPI<T>(nullptr), int>{}) 
{
	return 0;
}

// ...

std::cout << compute(One{}) << std::endl;	//Output: 100
std::cout << compute(Two{}) << std::endl;	//Output: 0

```

# 8. check if our type T has given constructor (matching the parameter list)

```cpp
template <typename U>
std::true_type test(U);

std::false_type test(...);

template <typename T, typename... Ts>
std::false_type test_has_ctor(...);

template <typename T, typename... Ts>
auto test_has_ctor(T*) -> decltype(test(std::declval< decltype(T(std::declval<Ts>()...)) >()));

class Test
{
    public:
    Test(int, int) {};
};

// ...
std::cout << std::boolalpha;
std::cout << decltype( test_has_ctor<Test, int, int>(nullptr) )::value		//Output: true
std::cout << decltype( test_has_ctor<Test, int, int, float>(nullptr) )::value	//Output: false
```
The core part is the matching:
```cpp
decltype(test(declval<decltype(T(declval<Ts>()...)) >()))
```
In this expression we try to build a real object using given set of parameters. We simply try to call its constructor. Let’s read this part by part:

The most outer decltype returns the type of the test function invocation. This might be true_type or false_type depending on what version will be chosen.

Inside we have:
```cpp
declval<decltype(T(declval<Ts>()...)) >()
```
Now, the most inner part ‘just’ calls the proper constructor. Then we take a type out of that (should be T) and create another value that can be passed to the test function.

***SFINAE in SFINAE…***

# 9a. Function to generate a tuple given a size N and a type T
[ Source: https://stackoverflow.com/questions/33511753/how-can-i-generate-a-tuple-of-n-type-ts ]

[ Source: https://stackoverflow.com/questions/37093920/function-to-generate-a-tuple-given-a-size-n-and-a-type-t ]

Case : write generate_tuple_type<int, 3> which would internally have a type alias type which would be std::tuple<int, int, int> in this case.

Fairly straightforward recursive formulation:

```cpp
template<typename T, unsigned N, typename... REST>
struct generate_tuple_type
{
 typedef typename generate_tuple_type<T, N-1, T, REST...>::type type;
};

template<typename T, typename... REST>
struct generate_tuple_type<T, 0, REST...>
{
  typedef std::tuple<REST...> type;
};

int main()
{
  using gen_tuple_t = generate_tuple_type<int, 3>::type;
  using hand_tuple_t = std::tuple<int, int, int>;
  static_assert( std::is_same<gen_tuple_t, hand_tuple_t>::value, "different types" );
}

```

Another possible example:

```cpp
template<size_t, class T>
using T_ = T;

template<class T, size_t... Is>
auto gen(std::index_sequence<Is...>) { return std::tuple<T_<Is, T>...>{}; }

template<class T, size_t N>
auto gen() { return gen<T>(std::make_index_sequence<N>{}); } 
```

And usage:

```cpp
auto tup = gen<MyType, N>();
```

# 9b. Produce std::tuple of same type in compile time given its length by a template argument
[ Source: https://stackoverflow.com/questions/38885406/produce-stdtuple-of-same-type-in-compile-time-given-its-length-by-a-template-a ]

How to implement a function with an int template argument indicating the tuple length and produce a std::tuple, with that length with usage below:
```cpp
func<2>() returns std::tuple<int, int>();
func<5>() returns std::tuple<int, int, int, int, int>().
```

A recursive solution with alias template and it's implementable in C++11:
```cpp
template <size_t I,typename T> 
struct tuple_n{
    template< typename...Args> using type = typename tuple_n<I-1, T>::template type<T, Args...>;
};

template <typename T> 
struct tuple_n<0, T> {
    template<typename...Args> using type = std::tuple<Args...>;   
};
template <size_t I,typename T>  using tuple_of = typename tuple_n<I,T>::template type<>;
```

Example of usage:

```cpp
tuple_of<3, double> t;
```

Using an index_sequence and a helper type alias you can generate the type you want:

```cpp
// Just something to take a size_t and give the type `int`
template <std::size_t>
using Integer = int;

// will get a sequence of Is = 0, 1, ..., N
template <std::size_t... Is>
auto func_impl(std::index_sequence<Is...>) {
    // Integer<Is>... becomes one `int` for each element in Is...
    return std::tuple<Integer<Is>...>{};
}

template <std::size_t N>
auto func() {
    return func_impl(std::make_index_sequence<N>{});
}
```

It is worth calling out that in the general case you would probably be better with a std::array, (in your case you can't use one), but a std::array can behave like a tuple, similarly to a std::pair.

Update: since you've made it clear you're working with c++11 and not 14+, you'll need to get an implementation of index_sequence and related from somewhere. Here is the C++11 version of func and func_impl with explicit return types:

```cpp
template <std::size_t... Is>
auto func_impl(std::index_sequence<Is...>) -> std::tuple<Integer<Is>...> {
  return std::tuple<Integer<Is>...>{};
}

template <std::size_t N>
auto func() -> decltype(func_impl(std::make_index_sequence<N>{})) {
  return func_impl(std::make_index_sequence<N>{});
}
```

And the easiest way:

```cpp
template<std::size_t N>
auto array_tuple() {
    return std::tuple_cat(std::tuple<int>{}, array_tuple<N-1>());
}

template<>
auto array_tuple<0>() {
    return std::tuple<>{};
}
```

# 10. Where and why do I have to put the “template” and “typename” keywords?
[ Source: https://stackoverflow.com/questions/610245/where-and-why-do-i-have-to-put-the-template-and-typename-keywords ]

In order to parse a C++ program, the compiler needs to know whether certain names are types or not. The following example demonstrates that:

```cpp
t * f;
```

How should this be parsed? For many languages a compiler doesn't need to know the meaning of a name in order to parse and basically know what action a line of code does. In C++, the above however can yield vastly different interpretations depending on what t means. If it's a type, then it will be a declaration of a pointer f. However if it's not a type, it will be a multiplication. So the C++ Standard says at paragraph (3/7):

	Some names denote types or templates. In general, whenever a name is encountered 
	it is necessary to determine whether that name denotes one of these entities before continuing to parse 	
	the program that contains it. The process that determines this is called name lookup.

How will the compiler find out what a name t::x refers to, if t refers to a template type parameter? x could be a static int data member that could be multiplied or could equally well be a nested class or typedef that could yield to a declaration. If a name has this property - that it can't be looked up until the actual template arguments are known - then it's called a dependent name (it "depends" on the template parameters).

You might recommend to just wait till the user instantiates the template:

	Let's wait until the user instantiates the template, 
	and then later find out the real meaning of t::x * f;.

This will work and actually is allowed by the Standard as a possible implementation approach. These compilers basically copy the template's text into an internal buffer, and only when an instantiation is needed, they parse the template and possibly detect errors in the definition. But instead of bothering the template's users (poor colleagues!) with errors made by a template's author, other implementations choose to check templates early on and give errors in the definition as soon as possible, before an instantiation even takes place.

So there has to be a way to tell the compiler that certain names are types and that certain names aren't.

***The "typename" keyword***

The answer is: We decide how the compiler should parse this. If t::x is a dependent name, then we need to prefix it by typename to tell the compiler to parse it in a certain way. The Standard says at (14.6/2):

A name used in a template declaration or definition and that is dependent on a template-parameter is assumed not to name a type unless the applicable name lookup finds a type name or the name is qualified by the keyword typename.

There are many names for which typename is not necessary, because the compiler can, with the applicable name lookup in the template definition, figure out how to parse a construct itself - for example with T *f;, when T is a type template parameter. But for t::x * f; to be a declaration, it must be written as typename t::x *f;. If you omit the keyword and the name is taken to be a non-type, but when instantiation finds it denotes a type, the usual error messages are emitted by the compiler. Sometimes, the error consequently is given at definition time:

```cpp
// t::x is taken as non-type, but as an expression the following misses an
// operator between the two names or a semicolon separating them.
t::x f;
```

The syntax allows typename only before qualified names - it is therefor taken as granted that unqualified names are always known to refer to types if they do so.

A similar gotcha exists for names that denote templates, as hinted at by the introductory text.

***The "template" keyword***

Remember the initial quote above and how the Standard requires special handling for templates as well? Let's take the following innocent-looking example:

```cpp
boost::function< int() > f;
```

It might look obvious to a human reader. Not so for the compiler. Imagine the following arbitrary definition of boost::function and f:

```cpp
namespace boost { int function = 0; }
int main() { 
  int f = 0;
  boost::function< int() > f; 
}
```

That's actually a valid expression! It uses the less-than operator to compare boost::function against zero (int()), and then uses the greater-than operator to compare the resulting bool against f. However as you might well know, boost::function in real life is a template, so the compiler knows (14.2/3):

	After name lookup (3.4) finds that a name is a template-name, if this name is followed by a <, 
	the < is always taken as the beginning of a template-argument-list and never as a name 	
	followed by the less-than operator.

Now we are back to the same problem as with typename. What if we can't know yet whether the name is a template when parsing the code? We will need to insert template immediately before the template name, as specified by 14.2/4. This looks like:

```cpp
t::template f<int>(); // call a function template
```

Template names can not only occur after a :: but also after a -> or . in a class member access. You need to insert the keyword there too:

```cpp
this->template f<int>(); // call a function template
```

***Dependencies***

For the people that have thick Standardese books on their shelf and that want to know what exactly I was talking about, I'll talk a bit about how this is specified in the Standard.

In template declarations some constructs have different meanings depending on what template arguments you use to instantiate the template: Expressions may have different types or values, variables may have different types or function calls might end up calling different functions. Such constructs are generally said to depend on template parameters.

The Standard defines precisely the rules by whether a construct is dependent or not. It separates them into logically different groups: One catches types, another catches expressions. Expressions may depend by their value and/or their type. So we have, with typical examples appended:

1. Dependent types (e.g: a type template parameter T)
2. Value-dependent expressions (e.g: a non-type template parameter N)
3. Type-dependent expressions (e.g: a cast to a type template parameter (T)0)

Most of the rules are intuitive and are built up recursively: For example, a type constructed as T[N] is a dependent type if N is a value-dependent expression or T is a dependent type. The details of this can be read in section (14.6.2/1) for dependent types, (14.6.2.2) for type-dependent expressions and (14.6.2.3) for value-dependent expressions.

***Dependent names***

The Standard is a bit unclear about what exactly is a dependent name. On a simple read (you know, the principle of least surprise), all it defines as a dependent name is the special case for function names below. But since clearly T::x also needs to be looked up in the instantiation context, it also needs to be a dependent name (fortunately, as of mid C++14 the committee has started to look into how to fix this confusing definition).

To avoid this problem, I have resorted to a simple interpretation of the Standard text. Of all the constructs that denote dependent types or expressions, a subset of them represent names. Those names are therefore "dependent names". A name can take different forms - the Standard says:

	A name is a use of an identifier (2.11), operator-function-id (13.5), conversion-function-id (12.3.2), 
	or template-id (14.2) that denotes an entity or label (6.6.4, 6.1)

An identifier is just a plain sequence of characters / digits, while the next two are the operator + and operator type form. The last form is template-name <argument list>. All these are names, and by conventional use in the Standard, a name can also include qualifiers that say what namespace or class a name should be looked up in.

A value dependent expression 1 + N is not a name, but N is. The subset of all dependent constructs that are names is called dependent name. Function names, however, may have different meaning in different instantiations of a template, but unfortunately are not caught by this general rule.

***Dependent function names***

Not primarily a concern of this article, but still worth mentioning: Function names are an exception that are handled separately. An identifier function name is dependent not by itself, but by the type dependent argument expressions used in a call. In the example f((T)0), f is a dependent name. In the Standard, this is specified at (14.6.2/1).

***Additional notes and examples***

In enough cases we need both of typename and template. Your code should look like the following

```cpp
template <typename T, typename Tail>
struct UnionNode : public Tail {
    // ...
    template<typename U> struct inUnion {
        typedef typename Tail::template inUnion<U> dummy;
    };
    // ...
};
```

The keyword template doesn't always have to appear in the last part of a name. It can appear in the middle before a class name that's used as a scope, like in the following example

```cpp
typename t::template iterator<int>::value_type v;
```

In some cases, the keywords are forbidden, as detailed below

1. On the name of a dependent base class you are not allowed to write typename. It's assumed that the name given is a class type name. This is true for both names in the base-class list and the constructor initializer list:

```cpp
 template <typename T>
 struct derive_from_Has_type : /* typename */ SomeBase<T>::type 
 { };
```

2. In using-declarations it's not possible to use template after the last ::, and the C++ committee said not to work on a solution.

```cpp
 template <typename T>
 struct derive_from_Has_type : SomeBase<T> {
    using SomeBase<T>::template type; // error
    using typename SomeBase<T>::type; // typename *is* allowed
 };
```

# 11. Check callable signature passed in template parameter

In this example we pass callable object (lambda or functor) in template parameter. This callable object requires a special signature which have to be checked during compilation process. See the example below for understanding how it works:	
	
```cpp
template<typename T, typename GetSomething, typename Equal = std::equal_to<T>, typename Float = float>
class Test
{
    static_assert(std::is_convertible_v<std::invoke_result_t<GetSomething, const T&>, Something<Float>>,
        "GetSomething must be a callable of signature Something<Float>(const T&)");

    static_assert(std::is_convertible_v<std::invoke_result_t<Equal, const T&, const T&>, bool>,
        "Equal must be a callable of signature bool(const T&, const T&)");

    static_assert(std::is_arithmetic_v<Float>);
	
};	
```

# 12. Get the type of a lambda argument?

[ Source: https://stackoverflow.com/questions/6512019/can-we-get-the-type-of-a-lambda-argument ]
	
The simplest of which is simply overloading: a functor type with a templated operator() (a.k.a a polymorphic functor) simply takes all kind of arguments; so what should argument_type be? Another reason is that generic code (usually) attempts to specify the least constraints on the types and objects it operates on in order to more easily be (re)used.

In other words, generic code is not really interested that given Functor f, typename Functor::argument be int. It's much more interesting to know that f(0) is an acceptable expression. For this C++0x gives tools such as decltype and std::declval (conveniently packaging the two inside std::result_of).

The way I see it you have two choices: require that all functors passed to your template use a C++03-style convention of specifying an argument_type and the like; use the technique below; or redesign. I'd recommend the last option but it's your call since I don't know what your codebase looks like or what your requirements are.

For a monomorphic functor type (i.e. no overloading), it is possible to inspect the operator() member. This works for the closure types of lambda expressions.

```cpp
template<typename Ret, typename A, typename... Rest>
A helper(Ret(*) (A, Rest...));

template<typename F, typename Ret, typename A, typename... Rest>
A
helper(Ret (F::*)(A, Rest...));

template<typename F, typename Ret, typename A, typename... Rest>
A
helper(Ret (F::*)(A, Rest...) const);

template<typename F, typename Ret>
void
helper(Ret (F::*)() const);

template<typename F, typename Ret>
void
helper(Ret (F::*)());

template <typename F>
decltype(helper(&F::operator())) helper(F);

template<typename F>
struct first_argument {
    typedef decltype( helper(&F::operator()) ) type;
};


int main()
{
    
    auto l = [](double i){ return 1; };
    auto k = [](){ return 1; };
    
    using Type1 = typename first_argument<decltype(l)>::type;
    using Type2 = typename first_argument<decltype(k)>::type;
    using Type3 = typename first_argument<decltype(&function)>::type;
    
    std::cout << sizeof(Type1) << "\n";
    std::cout << sizeof(Type2) << "\n"; // warning here !!! Test only !!!
    std::cout << sizeof(Type3) << "\n";
    
    return 0;
}
```

***Non-capturing lambdas are convertible to function pointers***


# 13. How does this implementation of std::is_class work?

[ Source: https://stackoverflow.com/questions/35213658/how-does-this-implementation-of-stdis-class-work ]

Possible implementation:

```cpp
template<class T, T v>
    struct integral_constant{
    static constexpr T value = v;
    typedef T value_type;
    typedef integral_constant type;
    constexpr operator value_type() const noexcept {
        return value;
    }
};

namespace detail {
    template <class T> char test(int T::*);   //this line
    struct two{
        char c[2];
    };
    template <class T> two test(...);         //this line
}

//Not concerned about the is_union<T> implementation right now
template <class T>
struct is_class : std::integral_constant<bool, sizeof(detail::test<T>(0))==1 
                                                   && !std::is_union<T>::value> {};
```

It was written using programming technologie called "SFINAE" which stands for "Substitution failure is not an error". The basic idea is this:

```cpp
namespace detail {
  template <class T> char test(int T::*);   //this line
  struct two{
    char c[2];
  };
  template <class T> two test(...);         //this line
}
```

This namespace provides 2 overloads for test(). Both are templates, resolved at compile time. The first one takes a int T::* as argument. It is called a Member-Pointer and is a pointer to an int, but to an int thats a member of the class T. This is only a valid expression, if T is a class. The second one is taking any number of arguments, which is valid in any case.

So how is it used?

```cpp
sizeof(detail::test<T>(0))==1
```

Ok, we pass the function a 0 - this can be a pointer and especially a member-pointer - no information gained which overload to use from this. So if T is a class, then we could use both the T::* and the ... overload here - and since the T::* overload is the more specific one here, it is used. But if T is not a class, then we cant have something like T::* and the overload is ill-formed. But its a failure that happened during template-parameter substitution. And since "substitution failures are not an error" the compiler will silently ignore this overload.

Afterwards is the sizeof() applied. Noticed the different return types? So depending on T the compiler chooses the right overload and therefore the right return type, resulting in a size of either sizeof(char) or sizeof(char[2]).

And finally, since we only use the size of this function and never actually call it, we dont need an implementation.

***The test functions are never actually called. The fact they have no definitions doesn't matter if you don't call them. As you realised, the whole thing happens at compile time, without running any code.***

The expression sizeof(detail::test<T>(0)) uses the sizeof operator on a function call expression. The operand of sizeof is an unevaluated context, which means that the compiler doesn't actually execute that code (i.e. evaluate it to determine the result). It isn't necessary to call that function in order to know the sizeof what the result would be if you called it. To know the size of the result the compiler only needs to see the declarations of the various test functions (to know their return types) and then to perform overload resolution to see which one would be called, and so to find what the sizeof the result would be.

The rest of the puzzle is that the unevaluated function call detail::test<T>(0) determines whether T can be used to form a pointer-to-member type int T::*, which is only possible if T is a class type (because non-classes can't have members, and so can't have pointers to their members). If T is a class then the first test overload can be called, otherwise the second overload gets called. The second overload uses a printf-style ... parameter list, meaning it accepts anything, but is also considered a worse match than any other viable function (otherwise functions using ... would be too "greedy" and get called all the time, even if there's a more specific function t hat matches the arguments exactly). In this code the ... function is a fallback for "if nothing else matches, call this function", so if T isn't a class type the fallback is used.

***It doesn't matter if the class type really has a member variable of type int, it is valid to form the type int T::* anyway for any class (you just couldn't make that pointer-to-member refer to any member if the type doesn't have an int member).***

The int T::* is a pointer to member object. It can be used as follows:

```cpp
struct T { int x; }
int main() {
    int T::* ptr = &T::x;

    T a {123};
    a.*ptr = 0;
}
```

In the other line:
```cpp
template<class T> two test(...);
```
the ellipsis is a C construct to define that a function takes any number of arguments.

Specifically, in this code:
```cpp
namespace detail {
    template <class T> char test(int T::*);
    struct two{
        char c[2];
    };
    template <class T> two test(...);
}
```

you have two overloads:

one that is matched only when a T is a class type (in which case this one is the best match and "wins" over the second one)
on that is matched every time
In the first the sizeof the result yields 1 (the return type of the function is char), the other yields 2 (a struct containing 2 chars).

The boolean value checked is then:

sizeof(detail::test<T>(0)) == 1 && !std::is_union<T>::value
which means: return true only if the integral constant 0 can be interpreted as a pointer to member of type T (in which case it's a class type), but it's not a union (which is also a possible class type).

Notice that:

```cpp
struct X {
void f(int);
int a;
};
struct Y;

int X::* pmi = &X::a;
void (X::* pmf)(int) = &X::f;
double X::* pmd;
char Y::* pmc;
```

declares pmi, pmf, pmd and pmc to be a pointer to a member of X of type int, a pointer to a member of X of type void(int), a pointer to a member ofX of type double and a pointer to a member of Y of type char respectively. ***The declaration of pmd is well-formed even though X has no members of type double. Similarly, the declaration of pmc is well-formed even though Y is an incomplete type.***
