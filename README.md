# EEvar - EEPROM Arduino library

Simple and lightweight Arduino library that allows you to save variables in the EEPROM memory. 

No need to keep track of the address, offset or size of the data you want to store in the EEPROM.
After saving the variable to the EEPROM its value gets restored on the power-up or CPU reset.

Works with any POD (`bool`, `int`, `float`, custom structs, etc.) and `String`. 

## Usage

### Example

#### Simple types and `String`

```c++
#include "EEvar.h"

const EEstore<float> eeFloat(25.8);       //allocate EEPROM for storing float 
                                          //and save value 25.8 to it on the first start

const EEstore<int> eeInt(-3685);          //for storing int

const EEstring eeString(20, "initial");   //for storing String (20 chars max)

void setup() {
  Serial.begin(115200);

  //check if it's CPU first start
  Serial.println(
      EEPROMallocator::isFirstStart()
          ? F("First start") 
          : F("Not first start")
  );

  //float
  float v1;
  eeFloat >> v1;      //load from eeprom to v1
  Serial.println(v1);
  //Serial.println(eeFloat.get()); //same as above
  v1 = 99.2;
  eeFloat << v1;      //save v1 to eeprom

  //int
  Serial.println(eeInt.get());
  eeInt << 56;

  //String
  String v4;
  eeString >> v4;
  Serial.println(v4);
  v4 = "long string, not gonna fit";
  eeString << v4;
  
}

void loop() {
}

```

#### Structures

```c++
#include "EEvar.h"

struct Config {				//structure that we want to store
  bool isOn = false;
  char str[10] = "abc";
};

const EEstore<Config> eeStruct((Config())); //store without buffering

EEvar<Config> configVar((Config())); //store with buffering

void setup() {
  Serial.begin(115200);

  //without buffering
  Config conf;
  eeStruct >> conf;			//load from EEPROM to conf
  Serial.print(conf.isOn);
  Serial.print(" ");
  Serial.println(conf.str);
  conf.isOn = true;			//modify conf
  strncpy(conf.str, "hello", sizeof(v3.str));
  eeStruct << conf;			//save conf to EEPROM

  //with buffering
  Serial.println();
  Serial.print(configVar->isOn);
  Serial.print(" ");
  Serial.println(configVar->str);
  configVar->isOn = true;			//modify configVar
  strncpy(configVar->str, "hello", sizeof(Config::str));
  configVar.save();					//save configVar to EEPROM
  
}

void loop() {
}

```

#### Simulate first start

If you want to reset all the saved variables in the EEPROM to their default values
on the next CPU reset (right after you flash the sketch), 
re-define `EE_TEST_VAL` to some different value __before__ including the library.

```c++
#define EE_TEST_VAL 0x315A    // library default is 0x3159
#include "EEvar.h"

```

Try to choose "more random" values, never use something like `0xFFFF`, `0`, `1`, etc. or the first start detection may fail.
`EEPROMallocator::isFirstStart()` will also return `true` after `EE_TEST_VAL` is re-defined.



### Important notes

- All `EEstore<T>`, `EEstring`, `EEvar<T>` must be global or static (or in another way ensure a stable order of instantiations).
- Changing order of created EEPROM variables or adding new ones not at the end will corrupt the saved data.
- Type `T` can only be POD (`bool`, `int`, `float`, custom structs, etc.). `T` cannot be `String` and cannot have `String` as its member. Use `EEstring` for storing a `String`.
- On ESP8266 the library will allocate 512 additional bytes for the flash page buffer. Using `EEvar<T>` will actually store your data __twice__ in the RAM.
- Don't forget about EEPROM/FLASH wear-out! This library does NOT mitigate this problem.

### API

There's three types available in the library: 

- `EEstore<T>` - does not create buffer of your type `T`. Use for:
  - simple types like `bool`, `int`, `float`;
  - large structures, that you don't need to have in the RAM all the time.
- `EEvar<T>` - creates buffer of your type `T`. Use for:
  - values that you read frequently;
  - complex structures.
-  `EEstring` - use for storing `String` type. Does not buffer your string. Preserves string length, but no more than maxLen (first constructor argument).

Comparison table:

|                                     | `EEstore<T>` | `EEvar<T>`    | `EEstring`         |
| ----------------------------------- | ------------ | ------------- | -------------------------- |
| Can store                           | POD          | POD           | String of length <= maxLen |
| Buffered                            | no           | yes           | no                         |
| Easy struct-field access (via `->`) | no           | yes           | -                          |
| Size in RAM, bytes                  | 2            | 2 + sizeof(T) | 2                          |



All available classes and their methods:

```c++
class EEPROMallocator {
  static void* alloc(const uint16_t sz);    //allocate EEPROM bytes (used by the library, don't call it directly)
  static uint16_t busy();                   //counter of busy EEPROM, bytes
  static uint16_t free();                   //counter of free EEPROM, bytes
  static bool isFirstStart();               //check if it's CPU's first start
};

template<typename T>
class EEstore {
  EEstore();
  EEstore(const T& initial);
  const EEstore& operator<<(const T& val) const;  //save val to EEPROM
  const EEstore& operator>>(T& val) const;        //load to val from EEPROM
  const T get() const;                            //read from EEPROM
};

class EEstring {
  EEstring(const uint16_t maxLen, const char* initial = "");
  const EEstring& operator<<(const char* val) const;      //save char string to EEPROM
  const EEstring& operator<<(const String& val) const;    //save String to EEPROM
  const EEstring& operator>>(String& val) const;          //load to String val from EEPROM
  const String get() const;                               //read from EEPROM
};

template<typename T>
class EEvar {
  EEvar(const T& initial);
  T& operator*();               //access struct
  const T& operator*();         //access struct
  T* operator->();              //access struct
  const T* operator->() const;  //access struct
  void save() const;            //save to EEPROM
  void load();                  //load from EEPROM
};
```



## Supported architectures

- AVR-based Arduino: Uno, Nano, Mini Pro, 2560, etc.
- ESP8266

## License

MIT
