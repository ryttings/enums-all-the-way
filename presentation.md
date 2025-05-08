---
marp: true
---
# Enums all the way down

A C++ Story by Scott Rytting

---

- I learned programming from the bottom up
    - Logic gates -> machine code -> assembly
- I also learned programming through spreadsheets
- I started writing C++ 3 years ago when I started my internship at Keysight
    - Immediately started in a crash course of all the ways C++ is different
    - Initializer lists were spooky
    - So much new syntax, so many ::

---

```cpp
enum class ConfigId{
    Temperature,
    Humidity,
    Schedule
};

class Schedule{
    //...
};

using ThermostatConfig = Configurable{
    {ConfigId::Temperature, double{0}},
    {ConfigId::Humidity, int{0}},
    {ConfigId::Schedule, Schedule{}}
}
```

This meant I saw a *lot* of enums

---

It also meant I saw a lot of ways to convert between enums...

```cpp
enum class InterfaceConfig{
    Temperature,
    Humidity,
    Schedule
};

// How to get from A to B?

enum class DriverConfig{
    Temperature,
    Humidity,
    Schedule
};
```

---

Of course, the first thing I tried:
```cpp
static_cast<DriverConfig>(InterfaceConfig::Temperature);
```

...which worked very well!
(as long as both enums were exactly the same and in exactly the same order)

...It turns out that doesn't happen very often

Much to my dismay...

---

The larger problem:

- How do we map enums to values?

---

A common solution:
```cpp
std::unordered_map<EnumA, EnumB>{
    {EnumA::First, EnumB::First},
    {EnumA::Second, EnumB::Second}
};
```

A few problems:
- It isn't `constexpr`, which means it infects everything it touches to not be `constexpr`
- It's obviously overkill for something so simple

But, it's common because it's expressive and compact.

---

Closer:
```cpp
constexpr int getProperty(Example e){
int result;
switch (e){
    case Example::A:
      result = 51;
      break;
   case Example::B:
      result = 42;
      break;
   case Example::E:
      result = 57;
      break;
   default:
      result = 0;
      break;
   }
   return result;
}
```

---

Which compiles to:
```c
        xor     edx, edx
        cmp     eax, 4
        ja      .L1
        mov     edx, DWORD PTR CSWTCH.1[0+rax*4]
.L1:
        mov     eax, edx
        add     rsp, 24
        ret
CSWTCH.1:
        .long   51
        .long   42
        .long   0
        .long   0
        .long   57
```
With GCC 14.2 -O3

---

<style>
    .container{
        display: flex;
    }
    .col{
        flex: 1;
        }
</style>
<div class="container">

<div class="col">
Even without a default

```cpp
int result;
switch (e){
    case Example::A:
      result = 51;
      break;
   case Example::B:
      result = 42;
      break;
   case Example::C:
      result = 0;
      break;
   case Example::D:
      result = 0;
      break;
   case Example::E:
      result = 57;
      break;
   //default:
   //   result = 0;
   //   break;
   }
   return result;
```

</div>

<div class="col">
Compiles to:

```c
        xor     edx, edx
        cmp     eax, 4
        ja      .L1
        mov     edx, DWORD PTR CSWTCH.1[0+rax*4]
.L1:
        mov     eax, edx
        add     rsp, 24
        ret
CSWTCH.1:
        .long   51
        .long   42
        .long   0
        .long   0
        .long   57
```

(but with a warning)
```
<source>: In function ‘int main()’:
<source>:413:5: warning: ‘result’ may be used uninitialized [-Wmaybe-uninitialized]
  413 | int result;
      |     ^~~~~~
Compiler returned: 0
```
</div>
</div>



---

<div class = "container">

<div class = "col">

A little more ideal:
```cpp
constexpr static auto valueArr =
std::array<int, 5>{
        51,
        42,
        0,
        0,
        57
    };
return valueArr[
    static_cast<unsigned int>(e)];
```

</div>

<div class = "col">

Compiles to:
```
        mov     eax, DWORD PTR valueArr[0+rax*4]
        add     rsp, 24
        ret
valueArr:
        .long   51
        .long   42
        .long   0
        .long   0
        .long   57
```

</div>
</div>

---

A problem:
```cpp
WrongEnum w = WrongEnum::First;
//Only to be used with Example Enum!
constexpr static auto valueArr = std::array<int, 5>{
    51,
    42,
    0,
    0,
    57
};
auto result = valueArr[static_cast<unsigned int>(w)];
```

---

And one that's a little harder to spot:
```cpp
constexpr int arrayProperty(Example e){
    constexpr static auto valueArr = std::array<int, 5>{
        51,
        42,
        0,
        0,
        57
    };
    return valueArr[static_cast<unsigned int>(e)];
}
```

---

And one that's a little harder to spot:
```cpp
constexpr int arrayProperty(Example e){
constexpr static auto valueArr = std::array<int, 5>{
        51, //Example::A
        42, //Example::B
        0,  //Example::C
        0,  //Example::D
        57  //Example::E
    };
    return valueArr[static_cast<unsigned int>(e)];
}
```

---

<div class="container">
<div class="col">

```cpp
constexpr auto valueArr = 
    std::array<int, 5>{
        51, //Example::A
        42, //Example::B
        0,  //Example::C
        0,  //Example::D
        57  //Example::E
    };
```
</div>

<div class="col">

```cpp
//I just like this order more
enum class Example{
    C,
    D,
    A,
    E,
    B,
};
```

</div>
</div>

---
<div class="container">

<div class="col">
Get something that looks like this:

```cpp
std::unordered_map{
    {Example::A, 51},
    {Example::B, 42},
    {Example::E, 57}
};
```
</div>

<div class="col">
And compiles like this:

```cpp
constexpr auto valueArr = 
    std::array<int, 5>{
        51, //Example::A
        42, //Example::B
        0,  //Example::C
        0,  //Example::D
        57  //Example::E
    };
```
</div>
</div>

With all the type safety and quality of life we can muster.

---

First things first.
- This is a special case, so we should make it work only for that special case
- Avoid untrue interfaces
- Make it like it's for everyone to use.

```cpp
template <typename T>
concept Enum = std::is_enum_v<T>;
```

---

```cpp
template <Enum E, typename T, size_t Size>
class EnumIndex
{
public:
   constexpr explicit EnumIndex() = default;

   constexpr T& at(const E identifier)
   {
      return mValueArray[static_cast<size_t>(identifier)];
   }

   constexpr const T& at(const E identifier) const
   {
      return mValueArray[static_cast<size_t>(identifier)];
   }

private:
   std::array<T, Size> mValueArray;
};
```

```cpp
EnumIndex<Example, int, 5>
```

---

```cpp
    EnumIndex<Example, int, 5> index;
    index.at(Example::A) = 51;
    index.at(Example::B) = 42;
    index.at(Example::C) = 0;
    index.at(Example::D) = 0;
    index.at(Example::E) = 57;
```

---

```cpp
consteval auto makeEnumIndex(){
    EnumIndex<Example, int, 5> index;
    index.at(Example::A) = 51;
    index.at(Example::B) = 42;
    index.at(Example::C) = 0;
    index.at(Example::D) = 0;
    index.at(Example::E) = 57;
    return index;
}
//...
constexpr auto value = index.at(Example::A); //OK
//...
auto value = index.at(e);
```

```c
mov     eax, DWORD PTR [rsp+24+rax*4]
```

---

```cpp
consteval auto makeEnumIndex(){
    EnumIndex<Example, int, 5> index;
    index.at(Example::A) = 51;
    index.at(Example::B) = 42;
    index.at(Example::C) = 0;
    index.at(Example::D) = 0;
    index.at(Example::E) = 57;
    return index;
}
```

But this still has some problems:
- Leaky abstractions
    - We really shouldn't have to use the `.at()` operator to build the entire object
    - Requires knowing how many total enums there are
    - Requires caller discipline to initialize everything

---

One more issue:
```cpp
constexpr EnumIndex<Example, int, 5> index;
index.at(Example::A) = 51; //Compile error
index.at(Example::B) = 42;
index.at(Example::C) = 0;
index.at(Example::D) = 0;
index.at(Example::E) = 57;
```

- Requires wrapping in another function to make `constexpr`
(not to mention requiring default constructible, cvref, etc.)

---

```cpp
template <EnumConstructorHelper... Helpers>
constexpr auto makeEnumIndex()
{
   constexpr auto helperArr = std::array{Helpers...};
   using EnumType = typename decltype(helperArr)::value_type::EnumType;

   static_assert(enumArrSize<EnumType>() == helperArr.size()
   && "Enum index only valid if defined for every value from min to max");
   static_assert(!containsDuplicates<helperArr>(
     && "Enum index value defined multiple times!");

   auto enumIndex = EnumIndex<
   decltype(helperArr.at(0).mEnum),
   decltype(helperArr.at(0).mData),
   enumArrSize<EnumType>()>();

   for (const auto helper : helperArr)
   {
      enumIndex.at(helper.mEnum) = helper.mData;
   }
   return enumIndex;
}
```

---

<div class="container">
<div class="col">

```cpp
template <Enum E, typename T>
struct EnumConstructorHelper
{
   using EnumType = E;
   using DataType = T;

   EnumType mEnum;
   DataType mData;
};
```

```cpp
template <EnumConstructorHelper... Helpers>
constexpr auto makeEnumIndex()
```
</div>
<div class="col">

Causes the compiler to see this:
```cpp
makeEnumIndex<
{Example::A, 0},
{Example::B, 1},
{Example::C, 2}
>();
```

As this:
```cpp
makeEnumIndex<
EnumConstructorHelper{Example::A, 0},
EnumConstructorHelper{Example::B, 1},
EnumConstructorHelper{Example::C, 2}
>();
```
</div>

---

Next, we make an array, and do some checks on it:
```cpp
//from GCC standard library:
  template<typename _Tp, std::size_t _Nm>
    struct array
    {
      typedef _Tp 	    			      value_type;
      //...
```

```cpp
constexpr auto helperArr = std::array{Helpers...};
using EnumType = typename decltype(helperArr)::value_type::EnumType;

static_assert(enumArrSize<EnumType>() == helperArr.size()
    && "Enum index only valid if defined for every value from min to max");
static_assert(!containsDuplicates<helperArr>(
    && "Enum index value defined multiple times!");
```

---

```cpp
template <auto Array>
constexpr bool containsDuplicates()
{
   for (size_t i = 1; i < Array.size(); i++)
   {
      for (size_t j = 0; j < i; j++)
      {
         if (Array[i] == Array[j])
         {
            return true;
         }
      }
   }
   return false;
}
```

---

Now the tricky part... What does "enum array size" mean?
```cpp
//Minimum size of an array that can index enums directly
template <Enum E>
constexpr size_t enumArrSize()
{
   constexpr auto max = maxEnum<E>();
   return static_cast<size_t>(max) + 1;
}
```

---

```cpp
template <typename E>
constexpr E maxEnum()
{
   int biggest = std::numeric_limits<int>::min();
   constexpr auto allEnums = enumValues<E>();
   for (auto val : allEnums)
   {
      if (static_cast<int>(val) > biggest)
      {
         biggest = static_cast<int>(val);
      }
   }
   return static_cast<E>(biggest);
}
```

---

Well... how many enums could there be?
- Exactly as many as you say there are
```cpp
template <typename E>
constexpr auto enumValues() noexcept
{
   constexpr auto enumMinValue = -10;
   constexpr auto enumMaxValue = 100;
   constexpr auto numPossibleEnums = enumMaxValue - enumMinValue + 1;
   return enumValues<E>(std::make_index_sequence<numPossibleEnums>({}));
}
```

---

```cpp
template <typename E, std::size_t... I>
constexpr auto enumValues(std::index_sequence<I...>) noexcept
{
   constexpr bool validArr[sizeof...(I)] = {hasValue<E, castVal<E>(I)>()...};
   constexpr auto numValid               = countValues(validArr);

   std::array<E, numValid> enumValues = {};
   for (size_t sourceIndex = 0, targetIndex = 0; targetIndex < numValid; ++sourceIndex)
   {
      if (validArr[sourceIndex])
      {
         enumValues[targetIndex] = castVal<E>(sourceIndex);
         targetIndex++;
      }
   }

   return enumValues;
}
```

---

Here we're doing a couple different things:
1. Templating on a sequence of numbers:
```cpp
template <typename E, std::size_t... I>
constexpr auto enumValues(std::index_sequence<I...>)
```
- Here `std::index_sequence` is really just a wrapper, the thing we want is the list of numbers `I`

---

2. Creating an array of these numbers, after we apply a function to them.
```cpp
constexpr bool validArr[sizeof...(I)] = {hasValue<E, castVal<E>(I)>()...};
```
- `sizeof...(I)` really just means "how many elements are in this parameter pack?"
    - Note that since we're passing in an index sequence from -10 to 100, this array will always be the same size.

- `hasValue<E, castVal<E>(I)>()...` is behaving like a comma separated list of the result of these nested function calls, `hasValue` and `castVal`

```cpp
template <Enum E>
   constexpr auto castVal(const size_t val)
   {
      return static_cast<E>(enumMinValue + val);
   }
```

---

On parameter packs:
- They were very difficult for me
- https://www.scs.stanford.edu/~dm/blog/param-pack.html is a very helpful resource.
- The `...` operator can do many different things, all somewhat related.
- Just play around with them!

---

2a) Actually applying the function (the black magic part)
```cpp
// Note: EnumIndex cannot be used for enums with any
// values defined as >enumMaxValue or <enumMinValue
template <Enum E, E EVal>
consteval bool hasValue()
{
    const auto* const name = std::source_location::current().function_name();
#ifdef __GNUC__
    return std::string_view(name).find("EVal = (") == std::string::npos;
#endif
#ifdef _MSC_VER
    return std::string_view(name).find("(enum") == std::string::npos;
#endif
}
```

---

3. Counting the number of valid enums:
```cpp
template <size_t Size>
constexpr auto countValues(const bool (&validArr)[Size])
{
   size_t count = 0;
   for (size_t i = 0; i < Size; i++)
   {
      if (validArr[i])
      {
         count++;
      }
   }
   return count;
}
```

---

4. Using that count to get the size of the array, and populating that array with the enum values
```cpp
   //...
   std::array<E, numValid> enumValues = {};
   for (size_t sourceIndex = 0, targetIndex = 0; targetIndex < numValid; ++sourceIndex)
   {
      if (validArr[sourceIndex])
      {
         enumValues[targetIndex] = castVal<E>(sourceIndex);
         targetIndex++;
      }
   }

   return enumValues;
}
```

---

Now we've figured out all the valid enums, so we can loop through them to find the biggest one
```cpp
template <typename E>
constexpr E maxEnum()
{
   int biggest             = std::numeric_limits<int>::min();
   constexpr auto allEnums = enumValues<E>();
   for (auto val : allEnums)
   {
      if (static_cast<int>(val) > biggest)
      {
         biggest = static_cast<int>(val);
      }
   }
   return static_cast<E>(biggest);
}
```
Which we can then use as our template parameter for EnumIndex.
(And help users with compilation errors that are not completely horrible)

---

```cpp
static_assert(enumArrSize<EnumType>() == helperArr.size()
&& "Enum index only valid if defined for every value from min to max");
```

```c
<source>: In instantiation of ‘constexpr auto makeEnumIndex()
[with EnumConstructorHelper<...auto...> ...Helpers = 
{EnumConstructorHelper<Example, int>{Example::A, 5},
EnumConstructorHelper<Example, int>{Example::B, 6},
EnumConstructorHelper<Example, int>{Example::C, 7},
EnumConstructorHelper<Example, int>{Example::D, 8}}]’:
<source>:281:6:   required from here
  276 |     constexpr auto index = makeEnumIndex<
      |                            ~~~~~~~~~~~~~~
  277 |         {Example::A, 5},
      |         ~~~~~~~~~~~~~~~~
  278 |         {Example::B, 6},
      |         ~~~~~~~~~~~~~~~~
  279 |         {Example::C, 7},
      |         ~~~~~~~~~~~~~~~~
  280 |         {Example::D, 8}
      |         ~~~~~~~~~~~~~~~
  281 |     >();
      |     ~^~
<source>:187:42: error: static assertion failed
  187 |    static_assert(enumArrSize<EnumType>() == helperArr.size()
  && "Enum index only valid if defined for every value from min to max");
      |                  ~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~
<source>:187:42: note: the comparison reduces to ‘(5 == 4)’
Compiler returned: 1

```

---

```cpp
static_assert(!containsDuplicates<helperArr>()
   && "Enum index value defined multiple times!");
```

```c
<source>: In instantiation of ‘constexpr auto makeEnumIndex() [with EnumConstructorHelper<...auto...> ...Helpers = {
    EnumConstructorHelper<Example, int>{Example::A, 5}, 
    EnumConstructorHelper<Example, int>{Example::B, 6}, 
    EnumConstructorHelper<Example, int>{Example::C, 7}, 
    EnumConstructorHelper<Example, int>{Example::D, 8}, 
    EnumConstructorHelper<Example, int>{Example::A, 10}}]’:
<source>:282:6:   required from here
  276 |     constexpr auto index = makeEnumIndex<
      |                            ~~~~~~~~~~~~~~
  277 |         {Example::A, 5},
      |         ~~~~~~~~~~~~~~~~
  278 |         {Example::B, 6},
      |         ~~~~~~~~~~~~~~~~
  279 |         {Example::C, 7},
      |         ~~~~~~~~~~~~~~~~
  280 |         {Example::D, 8},
      |         ~~~~~~~~~~~~~~~~
  281 |         {Example::A, 10}
      |         ~~~~~~~~~~~~~~~~
  282 |     >();
      |     ~^~
<source>:188:48: error: static assertion failed
  188 |    static_assert(!containsDuplicates<helperArr>() && "Enum index value defined multiple times!");
      |                   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^~
<source>:188:48: note: ‘! containsDuplicates<std::array<
EnumConstructorHelper<Example, int>, 5>{
    std::__array_traits<
EnumConstructorHelper<Example, int>,
 5>::_Type{EnumConstructorHelper<Example, int>{Example::A, 5}, EnumConstructorHelper<Example, int>{Example::B, 6},
  EnumConstructorHelper<Example, int>{Example::C, 7},
   EnumConstructorHelper<Example, int>{Example::D, 8},
    EnumConstructorHelper<Example, int>{Example::A, 10}}}>()’ evaluates to false
Compiler returned: 1
```

---

```cpp
template <EnumConstructorHelper... Helpers>
consteval auto makeEnumIndex()
{
   constexpr auto helperArr = std::array{Helpers...};
   using EnumType = typename decltype(helperArr)::value_type::EnumType;

   static_assert(enumArrSize<EnumType>() == helperArr.size()
   && "Enum index only valid if defined for every value from min to max");
   static_assert(!containsDuplicates<helperArr>()
   && "Enum index value defined multiple times!");

   auto enumIndex = EnumIndex<
   decltype(helperArr.at(0).mEnum),
   decltype(helperArr.at(0).mData),
   enumArrSize<EnumType>()>();

   for (const auto helper : helperArr)
   {
      enumIndex.at(helper.mEnum) = helper.mData;
   }
   return enumIndex;
}
```

---

Trying it out:

<div class="container">
<div class="col">

```cpp
__attribute__((noinline)) int doStuff(auto e1, auto e2){
    constexpr static auto index = makeEnumIndex<
        {Example::A, 5},
        {Example::B, 6},
        {Example::C, 7},
        {Example::D, 8},
        {Example::E, 10}
    >();
    constexpr auto value2 = index.at(Example::A);
    const auto value3 = index.at(e1);
    const auto value4 = index.at(e2);
    return value2 + value3 + value4;
}

int main(){
    int userIn;
    std::cin >> userIn;
    auto e = static_cast<Example>(userIn);
    auto e2 = static_cast<Example>(userIn + 1);
    
    return doStuff(e, e2);
}
```
</div>
<div class="col">

```c
doStuff(Example, Example):
        movsx   rdi, edi
        movsx   rsi, esi
        mov     edx, DWORD PTR doStuff(Example, Example)::index[0+rdi*4]
        mov     eax, DWORD PTR doStuff(Example, Example)::index[0+rsi*4]
        lea     eax, [rdx+5+rax]
        ret
main:
        sub     rsp, 24
        mov     edi, OFFSET FLAT:std::cin
        lea     rsi, [rsp+12]
        call    std::basic_istream<char, std::char_traits<char> >::operator>>(int&)
        mov     edi, DWORD PTR [rsp+12]
        add     rsp, 24
        lea     esi, [rdi+1]
        jmp     doStuff(Example, Example)
doStuff(Example, Example)::index:
        .long   5
        .long   6
        .long   7
        .long   8
        .long   10
```

---

What about at runtime?

<div class="container">
<div class="col">

```cpp
__attribute__((noinline)) int doStuff(Example e1, Example e2){
    constinit static auto index = makeEnumIndex<
        {Example::A, 5},
        {Example::B, 6},
        {Example::C, 7},
        {Example::D, 8},
        {Example::E, 10}
    >();
    index.at(Example::A) = 7;
    const auto value2 = index.at(Example::A);
    const auto value3 = index.at(e1);
    const auto value4 = index.at(e2);
    return value2 + value3 + value4;
}

int main(){
    int userIn;
    std::cin >> userIn;
    auto e = static_cast<Example>(userIn);
    auto e2 = static_cast<Example>(userIn + 1);
    
    return doStuff(e, e2);
}
```
</div>
<div class="col">

```c
doStuff(Example, Example):
        mov     DWORD PTR doStuff(Example, Example)::index[rip], 7
        movsx   rdi, edi
        movsx   rsi, esi
        mov     edx, DWORD PTR doStuff(Example, Example)::index[0+rdi*4]
        mov     eax, DWORD PTR doStuff(Example, Example)::index[0+rsi*4]
        lea     eax, [rdx+7+rax]
        ret
main:
        sub     rsp, 24
        mov     edi, OFFSET FLAT:std::cin
        lea     rsi, [rsp+12]
        call    std::basic_istream<char, std::char_traits<char> >::operator>>(int&)
        mov     edi, DWORD PTR [rsp+12]
        lea     esi, [rdi+1]
        call    doStuff(Example, Example)
        add     rsp, 24
        ret
doStuff(Example, Example)::index:
        .long   5
        .long   6
        .long   7
        .long   8
        .long   10
```

---

DefaultValue{}
```cpp
  // Allows creating an enum index without default values for some enum values
  // and removes the static_assert that would otherwise hit.
   template <EnumConstructorHelper... Helpers>
   constexpr auto makeEnumIndex(const default_value auto defaultValue)
   {
      constexpr auto helperArr = std::array{Helpers...};
      using HelperType = typename decltype(helperArr)::value_type;
      using EnumType = typename decltype(helperArr)::value_type::EnumType;

      static_assert(!containsDuplicates<helperArr>()
      && "Enum index value defined multiple times!");

      auto enumIndex =
         EnumIndex<EnumType, typename HelperType::DataType, 
         enumArrSize<EnumType>()>(helperArr, defaultValue.mDefaultValue);

      return enumIndex;
   }
```
---

```cpp
   // Normal helper for making enum indexes.
   // Requires that every enum be defined, and no enum be defined more than once.
   template <EnumConstructorHelper... Helpers>
   constexpr auto makeEnumIndex()
   {
      constexpr auto helperArr = std::array{Helpers...};
      using HelperType = typename decltype(helperArr)::value_type;
      using EnumType = typename decltype(helperArr)::value_type::EnumType;

      static_assert(enumArrSize<EnumType>() == helperArr.size() &&
                    "Enum index only valid if defined for every value from min to max");
      static_assert(!containsDuplicates<helperArr>() && "Enum index value defined multiple times!");

      auto enumIndex = EnumIndex<EnumType, typename HelperType::DataType,
      enumArrSize<EnumType>()>(helperArr);

      return enumIndex;
   }
```

---

NoDefault{}
```cpp
   // This allows using the containsKey() method of EnumIndex to determine
   // if an enum is valid, and then using the data stored inside it.
   template <EnumConstructorHelper... Helpers>
   constexpr auto makeEnumIndex(const NoDefault /*unused*/)
   {
      constexpr auto helperArr = std::array{Helpers...};

      using HelperType = typename decltype(helperArr)::value_type;
      using EnumType = typename decltype(helperArr)::value_type::EnumType;

      static_assert(!containsDuplicates<helperArr>()
      && "Enum index value defined multiple times!");

      auto enumIndex = EnumIndex<EnumType, typename HelperType::DataType,
      enumArrSize<EnumType>()>(helperArr);

      return enumIndex;
   }
```

---

```cpp
    constexpr explicit EnumIndex(std::array<EnumConstructorHelper<E, T>, Size> helpers)
         : mValues{helpers}
      {
         for (size_t i = 0; i < Size; i++)
         {
            mEnumLookup.at(static_cast<size_t>(mValues.at(i).mEnum)) = static_cast<uint16_t>(i);
         }
      }

      template <size_t NumValues>
      constexpr explicit EnumIndex(std::array<EnumConstructorHelper<E, T>, NumValues> helpers)
         : mValues{createHelperArray<E, T>(NoDefault{})}
      {
         for (size_t i = 0; i < NumValues; i++)
         {
            mEnumLookup.at(static_cast<size_t>(helpers.at(i).mEnum)) = static_cast<uint16_t>(i);
            mValues.at(i) = helpers.at(i);
         }
      }

      template <size_t NumValues>
      constexpr explicit EnumIndex(std::array<EnumConstructorHelper<E, T>, NumValues> helpers, T defaultValue)
         : mEnumLookup{createArray<int, 0, Size>()}
         , mValues{createHelperArray<E, T>(defaultValue)}
      {
         for (size_t i = 0; i < NumValues; i++)
         {
            // Every enum is valid so we'll use the entire array -- organize helpers accordingly
            mValues.at(static_cast<size_t>(helpers.at(i).mEnum)) = helpers.at(i);
         }
      }
```

---

Other nice things:
```cpp
      // Check to see if this index has a value for a specific enum value
      [[nodiscard]] constexpr bool containsKey(const E identifier) const
      {
         return static_cast<size_t>(identifier) < Size && mEnumLookup.at(static_cast<size_t>(identifier)) != -1;
      }

      [[nodiscard]] constexpr auto keys() const
      {
         return mValues |
                std::views::take_while([](const HelperType& helper) { return static_cast<size_t>(helper.mEnum) < Size; }) |
                std::views::transform([](const HelperType& helper) { return helper.mEnum; });
      }

      [[nodiscard]] constexpr auto values() const
      {
         return mValues |
                std::views::take_while([](const HelperType& helper) { return static_cast<size_t>(helper.mEnum) < Size; }) |
                std::views::transform([](const HelperType& helper) { return helper.mData; });
      }
```