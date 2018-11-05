# C++ zero cost abstraction to switch from AoS to SoA

Did you ever struggle because of data structures and memory layout? For cache performance you normally want to store data in structure of arrays (SoA) or array of structures (AoS) format. AoS is usually the natural way, and it matches the way the user thinks conceptually, while SoA is usually more efficient for cache reasons (depending on the algorithms and hardware architecture), so often the programmer has to choose a tradeoff between readability and performance.

With this zero-cost abstraction we show how to use either SoA or AoS interchangably, by keeping the "look and feel" of the AoS case. The user can decide which memory layout to use just by flipping a switch.

Also, you might have a code already written in which AoS memory layout is used , and you might think of refactoring it to gain performance. This abstraction allows to keep most of the codebase unchanged, and implement both cases without performance loss. A win-win.

As an example, imagine you have a container of "item"s.
In C++ an item would look like:

```C++
class item
{
public:
    double const&      getMyDouble() const { return mMyDouble; }
    char const&        getMyChar() const { return mMyChar };
    std::string const& getMyString() const { return mMyString };

    double&      myDouble(){ return mMyDouble };
    char&        myChar() { return mMyChar; }
    std::string& myString() { return mMyString; }

private:
    double      mMyDouble;
    char        mMyChar;
    std::string mMyString;
};
```

to be used inside containers, in situations like

```C++
std::vector<item> vec( 150 );
double            b = 0.;
for( auto& i : vec ) {
    i.myChar()   = "a";
    i.myDouble() = ++b;
    i.myString() = std : string( "ccc" );
}
```
The goal is to completely redesign the class "item" in such a way that the latter code snippet remains unchanged.
And we want to do that without paying any const, i.e. building a zero cost abstraction.

This problem is not trivial, and has given headaches to programmers in the last years, forcing them to choose a tradeoff between performance and code readability. Well this time is over, thanks to the magic of modern C++.
Let me explain the details by showing how to build the abstraction from scratch.

We create two helper classes which given an item generate the container type as a vector of tuples or a tuple of vectors, depending on a tag template argument. We call this class a DataLayoutPolicy and we are going to use it e.g. in this way:
```DataLayoutPolicy<std::vector, SoA, char, double, std::string>```
to generate a tuple of vectors of char, int, and double.

```C++
enum class DataLayout { SoA, //structure of arrays
                        AoS //array of structures
};
template <template <typename...> class Container, DataLayout TDataLayout, typename TItem>
struct DataLayoutPolicy {
};
```
This class will only contain static member functions to interact with the container (e.g. extract an element, insert, resize, etc...).
We write two template specializations. The first one (trivial) for the array of structures situation
```C++
template <template <typename...> class Container, template <typename...> class TItem, typename... Types>
struct DataLayoutPolicy<Container, DataLayout::AoS, TItem<Types...>> {
    using type       = Container<TItem<Types...>>;
    using value_type = TItem<Types...>&;

    constexpr static value_type get( type& c_, std::size_t position_ ) { return value_type( *static_cast<TItem<Types...>*>( &c_[ position_ ] ) ); }

    constexpr static void resize( type& c_, std::size_t size_ ) { c_.resize( size_ ); }

    template <typename TValue>
    constexpr static void        push_back( type& c_, TValue&& val_ ) { c_.push_back( val_ ); }
    static constexpr std::size_t size( type& c_ ) { return c_.size(); }
};
```
... just forwarding. We do the same for the structure of arrays case. Note: there are a few things to be explained about the code below.

* It wraps all the types in a ```ref_wrap``` type, which is a "decorated" std::reference_wrapper. This because we want to access the elements as lvalue references,
  to be able to change their value. using a regular reference we would be in trouble if for instance ```Types``` contains any reference.
  One thing worth noticing is that in the AoS case the DataLayoutPolicy::value_type is a reference, while in the SoA case is the value of a ref_wrap type.

* we return by value a newly created tuple of ref_wrap of the values. This is surpisingly OK, because the compiler is optimizing all
  copies away, and it is even more OK in C++17 (the returned tuple is a 'prvalue'), because of the guaranteed copy elision added to the standard: the tuple is not copied,
  this code would work even if std::tuple and std::reference_wrapper didn't have a copy/move constructor.

* we use a std::integer sequence to statically unroll a parameter pack: this is ugly but it is "the way" to do it since C++14
  (and in C++11 one had to use template recursion to achieve the same). There is not yet such thing like a "for_each" for parameter packs.

* we use C++17 fold expressions to call a function returning void multiple times. Before C++17 this was achieved concisely with tricky hacks.

```C++
template <typename T>
struct ref_wrap : public std::reference_wrapper<T> {
    operator T&() const noexcept { return this->get(); }
    ref_wrap( T& other_ )
        : std::reference_wrapper<T>( other_ )
    {
    }
    void operator=( T&& other_ ) { this->get() = other_; }
};

template <template <typename...> class Container, template <typename...> class TItem, typename... Types>
struct DataLayoutPolicy<Container, DataLayout::SoA, TItem<Types...>> {
    using type       = std::tuple<Container<Types>...>;
    using value_type = TItem<ref_wrap<Types>...>;

    constexpr static value_type get( type& c_, std::size_t position_ )
    {
        return doGet( c_, position_, std::make_integer_sequence<unsigned, sizeof...( Types )>() ); // unrolling parameter pack
    }

    constexpr static void resize( type& c_, std::size_t size_ )
    {
        doResize( c_, size_, std::make_integer_sequence<unsigned, sizeof...( Types )>() ); // unrolling parameter pack
    }

    template <typename TValue>
    constexpr static void push_back( type& c_, TValue&& val_ )
    {
        doPushBack( c_, std::forward<TValue>( val_ ), std::make_integer_sequence<unsigned, sizeof...( Types )>() ); // unrolling parameter pack
    }

    static constexpr std::size_t size( type& c_ ) { return std::get<0>( c_ ).size(); }

private:
    template <unsigned... Ids>
    constexpr static auto doGet( type& c_, std::size_t position_, std::integer_sequence<unsigned, Ids...> )
    {
        return value_type{ ref_wrap( std::get<Ids>( c_ )[ position_ ] )... }; // guaranteed copy elision
    }

    template <unsigned... Ids>
    constexpr static void doResize( type& c_, unsigned size_, std::integer_sequence<unsigned, Ids...> )
    {
        ( std::get<Ids>( c_ ).resize( size_ ), ... ); //fold expressions
    }

    template <typename TValue, unsigned... Ids>
    constexpr static void doPushBack( type& c_, TValue&& val_, std::integer_sequence<unsigned, Ids...> )
    {
        ( std::get<Ids>( c_ ).push_back( std::get<Ids>( std::forward<TValue>( val_ ) ) ), ... ); // fold expressions
    }
};
```

So now this code shows pretty clearly how this abstraction can be built. We show below a possible strategy to use it.
We define the policy_t type using the DataLayoutPolicy and a generic TItem type

```C++
template <template <typename T> class TContainer, DataLayout TDataLayout, typename TItem>
using policy_t = DataLayoutPolicy<TContainer, TDataLayout, TItem>;
```

The container class forwards most of the calls to the static functions defined by the policy_t type. It might look like the following

```C++
template <template <typename ValueType> class TContainer, DataLayout TDataLayout, typename TItem>
struct BaseContainer {
    using policy_t                    = policy_t<TContainer, TDataLayout, TItem>;
    using iterator                    = Iterator<BaseContainer<TContainer, TDataLayout, TItem>>;
    using difference_type             = std::ptrdiff_t;
    using value_type                  = typename policy_t::value_type;
    using reference                   = typename policy_t::value_type&;
    using size_type                   = std::size_t;
    auto static constexpr data_layout = TDataLayout;

    BaseContainer( size_t size_ )
    {
        resize( size_ );
    }

    template <typename Fwd>
    void push_back( Fwd&& val )
    {
        policy_t::push_back( mValues, std::forward<Fwd>( val ) );
    }

    std::size_t size()
    {
        return policy_t::size( mValues );
    }

    value_type operator[]( std::size_t position_ )
    {
        return policy_t::get( mValues, position_ );
    }

    void resize( size_t size_ )
    {
        policy_t::resize( mValues, size_ );
    }

    iterator begin() { return iterator( this, 0 ); }
    iterator end() { return iterator( this, size() ); }

private:
    typename policy_t::type mValues;
};
```

We call the helper functions defined for the SoA and AoS policies. We know by C++17 guaranteed copy elision that the ```operator[]``` does not instantiate anything -- not yet.

Now this is no standard container, so we have to define an iterator in order to use it whithin STL algorithms. The iterator we build looks like a STL iterator for a container of tuple,
except for the fact that it must hold a reference to the container, because when we call the dereference operator we want to call our storage's ```operator[]```, which statically dispatches the operation
using the container's data layout policy.

```C++
template <typename TContainer>
class Iterator
{

private:
    using container_t = TContainer;

public:
    using policy_t          = typename container_t::policy_t;
    using difference_type   = std::ptrdiff_t;
    using value_type        = typename policy_t::value_type;
    using reference         = value_type&;
    using iterator_category = std::bidirectional_iterator_tag;

    template <typename TTContainer>
    Iterator( TTContainer* container_, std::size_t position_ = 0 )
        : mContainer( container_ )
        , mIterPosition( position_ )
    {
    }

    Iterator& operator=( Iterator const& other_ )
    {
        mIterPosition = other_.mIterPosition;
    }

    friend bool operator!=( Iterator const& lhs, Iterator const& rhs ) { return lhs.mIterPosition != rhs.mIterPosition; }
    friend bool operator==( Iterator const& lhs, Iterator const& rhs ) { return !operator!=( lhs, rhs ); }

    operator bool() const { mIterPosition != std::numeric_limits<std::size_t>::infinity(); }

    Iterator& operator=( std::nullptr_t const& ) { mIterPosition = std::numeric_limits<std::size_t>::infinity(); }

    template <typename T>
    void operator+=( T size_ ) { mIterPosition += size_; }

    template <typename T>
    void operator-=( T size_ ) { mIterPosition -= size_; }

    void operator++() { return operator+=( 1 ); }
    void operator--() { return operator-=( 1 ); }

    value_type operator*()
    {
        return ( *mContainer )[ mIterPosition ];
    }

private:
    container_t* mContainer    = nullptr;
    std::size_t  mIterPosition = std::numeric_limits<std::size_t>::infinity();
};
```

Eventually we define our "item" data structure: we make it a decorator of a std::tuple, with some specific member functions (in this case only getters/setters).
We give readable names to the 0, 1, 2, 3 integer indices, they represent the data members of our item.

```C++
enum Component : int {
    eMyDouble,
    eMyChar,
    eMyString,
    eMyPadding
};

struct Pad {
    char mPad[ 1500 ];
};

template <typename... T>
struct Item : public std::tuple<T...> {
    using std::tuple<T...>::tuple;
    auto& myDouble() { return std::get<eMyDouble>( *this ); }
    auto& myChar() { return std::get<eMyChar>( *this ); }
    auto& myString() { return std::get<eMyString>( *this ); }
};
```

When we call Item's member functions we have to rely on compiler optimization in order for our abstraction to be "zero-cost":
C++17 guaranteed copy elision assures us that no copies of Item has been called so far, but we don't want to call the
constructor either, because we are creating a temporary tuple just to access one of it's member each time and we would thrash it right away. It just happens that compilers do a good job,
and we'll see later that all the accesses get optimized.

so eventually we can write the program:

```C++
template <typename T>
using MyVector = std::vector<T, std::allocator<T>>;

int main( int argc, char** argv )
{
    using container_t = BaseContainer<MyVector, DataLayout::SoA, Item<double, char, std::string, Pad>>;
    container_t container_( 1000 );
    std::cout << "container size " << container_.size() << " \n";
    for( auto&& i : container_ ) {
        i.myDouble() = static_cast<double>( argc );
    }
}
```
and we can write generic and efficient code regardless of the memory layout underneeth.
What's left to do is to check that this is a zero cost abstraction. The easiest way for me to check that is using a debugger:
compile the example with symbols on,
```
> clang++ -std=c++1z -O3 -g main.cpp -o test
```
run it with gdb, set a brakpoint in the for loop, and step through the assembly instructions (the ```layout split``` command shows the
source code and disassembled instructions at the same time)
```
> gdb test
(gdb) break main.cpp : 10 # set breakpoint inside the loop
(gdb) run # execute until the breakpoint
(gdb) layout split # show assembly and source code in 2 separate frames
(gdb) stepi # execute one instruction
```
The instructions being executed inside the loop are in case of AoS data layout
```
0x400b00 <main(int, char**)+192>        movsd  %xmm0,(%rsi)
0x400b04 <main(int, char**)+196>        add    $0x610,%rsi
0x400b0b <main(int, char**)+203>        add    $0xffffffffffffffff,%rcx
0x400b0f <main(int, char**)+207>        jne    0x400b00 <main(int, char**)+192>
```
Notice in particular that in the second line the offset being add to compute the address is 0x160. This changes depending on the
size of the data members in the item object. On the other hand for the SoA data structure we have
```
0x400b60 <main(int, char**)+224>        movups %xmm1,(%rdi,%rsi,8)
0x400b64 <main(int, char**)+228>        movups %xmm1,0x10(%rdi,%rsi,8)
0x400b69 <main(int, char**)+233>        movups %xmm1,0x20(%rdi,%rsi,8)
0x400b6e <main(int, char**)+238>        movups %xmm1,0x30(%rdi,%rsi,8)
0x400b73 <main(int, char**)+243>        movups %xmm1,0x40(%rdi,%rsi,8)
0x400b78 <main(int, char**)+248>        movups %xmm1,0x50(%rdi,%rsi,8)
0x400b7d <main(int, char**)+253>        movups %xmm1,0x60(%rdi,%rsi,8)
0x400b82 <main(int, char**)+258>        movups %xmm1,0x70(%rdi,%rsi,8)
0x400b87 <main(int, char**)+263>        movups %xmm1,0x80(%rdi,%rsi,8)
0x400b8f <main(int, char**)+271>        movups %xmm1,0x90(%rdi,%rsi,8)
0x400b97 <main(int, char**)+279>        movups %xmm1,0xa0(%rdi,%rsi,8)
0x400b9f <main(int, char**)+287>        movups %xmm1,0xb0(%rdi,%rsi,8)
0x400ba7 <main(int, char**)+295>        movups %xmm1,0xc0(%rdi,%rsi,8)
0x400baf <main(int, char**)+303>        movups %xmm1,0xd0(%rdi,%rsi,8)
0x400bb7 <main(int, char**)+311>        movups %xmm1,0xe0(%rdi,%rsi,8)
0x400bbf <main(int, char**)+319>        movups %xmm1,0xf0(%rdi,%rsi,8)
0x400bc7 <main(int, char**)+327>        add    $0x20,%rsi
0x400bcb <main(int, char**)+331>        add    $0x8,%rbx
0x400bcf <main(int, char**)+335>        jne    0x400b60 <main(int, char**)+224>
```
We see the loop is unrolled and vectorized by Clang (version 6.0.0),
and the increment for the address is 0x20, independent of
the number of data members present in the item struct.
