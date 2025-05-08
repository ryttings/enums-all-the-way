---
marp: true
---
hi
I am scott
This is a presentation!

---

So... why use enums in the first place?
- Let's say we're building embedded software to control a jukebox.
- The jukebox plays songs
- We're writing software that should take a song requested by the user and actually play it

---

**Simple approach: filenames as strings!**
```cpp
userRequest = "SongName";

playSong("SongName.mp3"); //or some other way to identify *where* the song is
```

This has some advantages.
- flexible
- straightforward

But also has some disadvantages
- We can ask for invalid states

---

Let's say there are only 10 songs we can play.
... explain more why here

---

Enums are ways to have a mutually exclusive set of related things, and that's nice!
Encoding more information in the type system is always nice.

---

How can we map from one enum to another?
A simple, efficient way:
```cpp
static_cast<DriverSongId>(Song::SongName);
```

but... there's nothing guaranteeing that these enums will be in order.

```cpp
enum class Song{
    SongA,
    SongB,
    SongC
}

enum class DriverSongId{
    SongA,
    SongC,
    SongB
}
```

---

Nor is there anything guaranteeing that they'll all be valid

```cpp
enum class Song{
    SongA,
    SongB,
    SongC
}

enum class DriverSongId{
    SongA,
    SongB
}
```


---

```cpp
std::map<EnumA, EnumB> translateEnum{
    {EnumA::First, EnumB::First},
    {EnumA::Second, EnumB::Second},
    //...and so on
}
```

```cpp
setConfig(translateEnum.at(EnumA::First));
```

---

That approach might look very inefficient

---

The idea of "Anonymous structs" can be useful, too.

```cpp
struct StateMachineIndex : Entry<uint8_t>{};
struct CounterValue : Entry<uint64_t>{};

constexpr auto getIndexedCounter(const uint64_t countTotal, const MachineMetadata& machine)
{
   const auto countBase = (countTotal >= wordSize) ? machine.symbolsPerWord : 0;
   const uint8_t countIndex = (countTotal % wordSize) / machine.bitsPerSymbol;
   const auto machineIndex = countBase + countIndex;
   const auto counterValue = countTotal / wordSize;

   return makeIndex(StateMachineIndex{machineIndex}, CounterValue{counterValue});
}
```

---

Enums -> Numbers
as
Strong Types -> Types