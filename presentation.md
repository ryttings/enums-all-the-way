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

My first attempt:
```cpp
static_cast<DriverConfig>(InterfaceConfig::Temperature);
```

...which worked very well!
(as long as both enums were exactly the same and in exactly the same order)

...It turns out that doesn't happen very often

Much to my dismay...

---

Fell back to the same way you would map any generic value to another:
```cpp
std::unordered_map<EnumA, EnumB>
```

---

Enums are also nice as an unambiguous way to grab related values:

Access::TriggerType -> Driver::DelayGroup -> Group value (abstracted from driver)

First pass enum index

-> proceed to be a sucker for syntax
 - something something all of math is syntax sugar


- Then started using std::source_location to determine valid enums (so the calling code didn't have to know specifically)

- This is about when I broke builds on the CI pipeline for the first time
    - Imagine my surprise that not all C++ compilers work exactly the same