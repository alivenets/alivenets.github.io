---
layout: post
title:  "Handy macro techniques for development in C"
date:   2023-11-09 12:00:00 +0200
categories: c macros tricks
---

In this article I will describe C macro tricks that I used before when coding extensively in C. Some of the examples here would be of use for embedded C developers.

# Disclaimer

The examples here are only for educational purposes. This code should not be used in the production. MISRA/AUTOSAR C rules exist to block such implementation behavior.

The reasons behind are:
* preprocessor has no clear standard itself
* macros are HARD to maintain and prone to errors at scale
* macros can mess up with the static code analyzers in production

So, please use C++ if possible or minimize macro usage. Use the macro techniques proposed here ONLY if it is REALLY needed and on your own responsibility!


# C macro techniques with examples

## Count elements in in array

Here we can count elements in C array.

```c
#define COUNT_OF(arr_) (sizeof(arr)/sizeof(arr[0]))

void foo()
{
    int a[10];
    for(int i = 0; i < COUNT_OF(a); i++) {
        //...
    }
}
```

## Size of struct member 

Here we can find the size of struct member.

```c
// Safe, since the expression is not evaluated but the type is determined
#define MEMBER_SIZE(type, member) sizeof(((type *)0)->member)

struct bar {
    int a;
    char b[16];
};

void foo()
{
    printf("sizeof b = %d\n", MEMBER_SIZE(struct bar, b));
}
```

## Max value of type

Here we can calculate max value of C type, whether it is signed or unsigned.

```c
#define IS_SIGNED(type)        (((type)(-1)) < ((type)0))

#define UNSIGNED_MAX_OF(type)  (((0x1ULL << ((sizeof(type) * 8ULL) - 1ULL)) - 1ULL) | \
                                    (0xFULL << ((sizeof(type) * 8ULL) - 4ULL)))

#define SIGNED_MAX_OF(type)    (((0x1ULL << ((sizeof(type) * 8ULL) - 1ULL)) - 1ULL) | \
                                    (0x7ULL << ((sizeof(type) * 8ULL) - 4ULL)))

#define TYPE_MAX_VALUE(type)   ((unsigned long long) (IS_SIGNED(type) \
                                   ? SIGNED_MAX_OF(type)              \
                                   : UNSIGNED_MAX_OF(type)))

```

## Stringify

To stringify variable name, any other token or macro, we write `STRINGIFY` macro function. Macro indirection is required to preprocess the input argument in case if it is also a macro.

```c
#define STRINGIFY(value) _STRINGIFY(value)
#define _STRINGIFY(value) #value

#define A() foo

STRINGIFY(foo)
STRINGIFY(A())
```

## Macro concatenation

To concatenate two variables or macro expansion results, we write `CAT` macro. This macro requires indirection in order to handle macro functions as arguments.

```c
#define CAT(a, b) _CAT(a, b)
#define _CAT(a) a ## b

#define A() foo
#define B() bar


int CAT(foo, bar) = 0; // this works
int CAT(A(), B()) = 1; //this also works

```

## Conditional macros

Using C macros, it becomes possible to calculate onditions during preprocessing time.
Here we use Pattern Matching technique: if the resulting token is a macro, it is re-evaluated.

```c
// Deferred IIF declaration
// Required to preprocess the condition argument first before concatenation 
// Will be unwrapped to _IIF_0 or _IIF_1 c
#define IIF(cond) CAT(_IIF_,cond)

// IIF declaration - conditional macro
#define _IIF_0(a,b)    b
#define _IIF_1(a,b)    a
```

## Boolean negation macros

Here we use so called Detection technique, allowing us to detecdt input based on input pattern.
For example, let's write boolean negation macro.

```c
// Choose second argument from the argument list
#define SELECT_2ND(x, n, ...) n
// This detects if PROBE is passed, then 1 is returned. Otherwise, 0 is returned
#define CHECK(...) SELECT_2ND(__VA_ARGS__, 0,)
// PROBE is a detection macro 
#define PROBE(x) x, 1,

// Checking condition using pattern matching.
// If x == 0, preprocessor recognized NOT_0. For the opposite case it is C token NOT_<smth>
#define NOT(x) CHECK(_CAT(NOT_, x))
// Always 1
#define NOT_0 PROBE(~)
```

## X macros

The well-known technique of declaring properties and applying custom operators to them is called "X macro".

Here for example we can list all available cars in the shop.

```c
// Brand, Name, Color, Price
#define ALL_CARS(_X) \
    _X(MERCEDES_BENZ, "Mercedes Benz EQA", 0xFF0000, 50000) \
    _X(BMW, "BMW i3", 0xFF00FF, 40000) \
    _X(VOLKSWAGEN, "Volkswagen ID.4", 0xDD00FF, 46500) 

enum Cars {
#define CAR_ID(prop,...) prop
    ALL_CARS(CAR_ID)
#undef CAR_ID
};
```

In this example data on cars is wrapped in to `_X` macro function which can be passed as an argument to `ALL_CARS`

This may simplify code generation based on declarative programming, minimize code duplication and adhere to DRY principle.

As a disadvantage, operators may be defined in multiple places and have to be modified whenever the declaration format changes, e.g. new field is added.

Also, the macro length is limited by compiler. With longer property list the limit can be easily reached. [For GCC, it is usually 4096 symbols](https://gcc.gnu.org/onlinedocs/cpp/Implementation-limits.html#:~:text=Number%20of%20characters%20on%20a,lines%20longer%20than%2065%2C535%20characters.). Since the macro preprocess result is one logical line, in case of hundreds of properties, compiler line limits could be overreached.

## Using X macros in advanced way

### Problem statement

Let's design device property handling mechanism. All properties are listed by property ID, which is used by  The property ID can be used by the application tasks in the firmware to read/write property values. Apparently, we need to provide the API to read/modify device properties via property ID

Here is the required API to handle properties by their IDs.

```c
// property_id.h

typedef enum PropertyId {
    ID_DEVICE_NAME,
    ID_TIME_ZONE,
    ID_USE_DHCP,
    ID_IP_ADDRESS,
} PropertyId_t;

// property_handlers.h

int readProperty(PropertyId_t propId, uint8_t *buf, size_t size);
int writeProperty(PropertyId_t propId, const uint8_t *buf, size_t size);
```

### Plain X Macro

Using plain X macro, we list all properties with attributes in one header file and generate code via various definitions of `_X`.

```c
// host_props.h

#include "property_id.h"

#define MAX_STRING 32

#define ALL_PROPERTIES(_X) \
    _X(ID_DEVICE_NAME, device_name, uint8_t, [MAX_STRING]) \
    _X(ID_TIME_ZONE,   time_zone,   uint32_t,            ) \
    _X(ID_USE_DHCP,    use_dhcp,    uint8_t,             ) \
    _X(ID_IP_ADDRESS,  ip_address,  uint32_t,            )
```

Then we can write:

```c
// host_props.c

#include "host_props.h"

// Struct definition to store data in persistent storage
typedef struct DeviceConfig {
#define CONFIG_FIELD(propId, propName, propType, postfix) propType propName postfix;
    ALL_PROPERTIES(CONFIG_FIELD)
#undef CONFIG_FIELD
} DeviceConfig_t;

DeviceConfig_t g_deviceProperties;

// memcpy helper
static int memcpyWithSizeCheck(uint8_t *buf_to, size_t size_to, const uint8_t *buf_from, size_t size_from)
{
    if(size_to != size_from) {
        return -1;
    }
    memcpy(buf_to, buf_from, size_to);
}

int readProperty(PropertyId_t propId, uint8_t *buf, size_t size)
{
    switch(propId) {
#define READ_PROP(propId, propName, propType, postfix) \
        case propId: return memcpyWithSizeCheck(buf, size, &g_deviceProperties_2.propName, sizeof(g_deviceProperties_2.propName));

        ALL_PROPERTIES(READ_PROP)
#undef READ_PROP
        default:
            return -2;
    }
}

int writeProperty(PropertyId_t propId, const uint8_t *buf, size_t size)
{
    switch(propId) {
#define WRITE_PROP(propId, propName, propType, postfix) \
        case propId: return memcpyWithSizeCheck($g_deviceProperties.propName, sizeof(g_deviceProperties.propName, buf, size);

        ALL_PROPERTIES(WRITE_PROP)
#undef #define WRITE_PROP(propId, propName, propType, postfix) \

        default:
            return -2;
    }
}
```

### X operator macro

The previous approach is however hardly scalable:
 * It is hard to maintain properties of different types (stored in memory, hardware properties, deprecated) since they requre separate lists
 * It is hard to address properties separately by its type.

The better way to addres the issues is to decouple property listing from the property declaration. The solution is to list every property separately using X macro and address the property by property ID and operator.

```c
// host_props_declare.h
#include "property_id.h"

#define MAX_STRING 32

// Here we list all properties
// NOTE: the format is important:
// macro name is PROP_<property ID>
#define PROP_ID_DEVICE_NAME(OP)  OP(ID_DEVICE_NAME, device_name,  uint8_t, [MAX_STRING] ) 
#define PROP_ID_TIME_ZONE(OP)    OP(ID_TIME_ZONE,   timezone,     uint32_t,             )
#define PROP_ID_USE_DHCP(OP)     OP(ID_USE_DHCP,    use_dhcp,     uint8_t,              )
#define PROP_ID_IP_ADDRESS(OP)   OP(ID_IP_ADDRESS,  ip_address,   uint32_t,             )

// Define operators to extract property value
#define OP_PROP_TYPE(propId, propFieldName, propType, ...) propType
#define OP_PROP_SIZE(propId, propFieldName, propType, propPostfix) sizeof(CAT(propType, propPostfix))
#define OP_PROP_POSTFIX(propId, propFieldName, propType, propPostfix) propPostfix
#define OP_PROP_NAME(propId, propFieldName,...) propFieldName

// Operator apply macro
#define PROP_APPLY(propId, op) CAT(PROP_,propId)(op)

// Define convenience macros to apply operators
#define PROP_TYPE(propId) PROP_APPLY(propId, OP_PROP_TYPE)
#define PROP_SIZE(propId) PROP_APPLY(propId, OP_PROP_SIZE)
#define PROP_POSTFIX(propId) PROP_APPLY(propId, OP_PROP_POSTFIX)
#define PROP_NAME(propId) PROP_APPLY(propId, OP_PROP_NAME)

#define DECLARE_PROP_VARIABLE(propId, name) PROP_TYPE(propId) name PROP_POSTFIX(propId)

// host_props.h

#define ALL_PROPERTIES(_X) \
    _X(ID_DEVICE_NAME) \
    _X(ID_TIME_ZONE) \
    _X(ID_USE_DHCP) \
    _X(ID_IP_ADDRESS)

// host_props.c

#include "host_props.h"
#include "host_props_declare.h"

typedef struct DeviceConfig {
#define DECLARE_CONFIG_FIELD(propId) PROP_TYPE(propId) PROP_NAME(propId) PROP_POSTFIX(propId);
    ALL_PROPERTIES(DECLARE_CONFIG_FIELD)
#undef DECLARE_CONFIG_FIELD
} DeviceConfig_t;

// other code is unchanged
```

Now, property declaration and listing are decoupled. The base operators can be defined close to the declarations. Properties can be referred separately in any code place.

Now, we can do write the following code:

```c
void onBootup()
{
    // Get type from the property definition, identical to what is in the config
    PROP_TYPE(ID_USE_DHCP) useDhcp;
    readProperty(ID_USE_DHCP, &useDhcp, sizeof(useDhcp));

    if (useDhcp) {
        // connectivity logic...
    }
}
```

However, there is a drawback: the property list requires some maintenance to keep it readable. And if new attrbute is added, all properties should be updated.

### Special properties

There may be some special cases, like empty property attributes. In this part we discover the approach to handle special property values.

Let's add special properties `ID_FW_VERSION` and `ID_DEVICE_TIME` which are not stored in persistent storage. `ID_FW_VERSION` is read-only value containing firmware build version. `ID_DEVICE_TIME` is RTC clock state showing current Unix timestamp, being updatable by a user.

The updated code now looks like following:

```c
// host_props.h 

#define ALL_PROPERTIES(_X) \
    _X(ID_DEVICE_NAME) \
    _X(ID_TIME_ZONE) \
    _X(ID_USE_DHCP) \
    _X(ID_IP_ADDRESS) \
    _X(ID_FW_VERSION) \
    _X(ID_DEVICE_TIME)

// host_props_declare.h

// Using special symbol $ to simplify pattern matching via CHECK macro
// This symbol is mostly unused in the code
#define PROP_ID_FW_VERSION(OP)    OP(ID_FW_VERSION,  $, uint8_t, [MAX_STRING] )
#define PROP_ID_DEVICE_TIME(OP)   OP(ID_DEVICE_TIME, $, uint32_t,             )

// Handle special name using detection technique mentioned earlier
#define IS_SPECIAL_NAME(name) CHECK(CAT(_IS_EMPTY_, name))
#define _IS_EMPTY_$ PROBE(~)

#define IS_SPECIAL_PROPERTY(propId) IS_SPECIAL_NAME(PROP_NAME(propId))

// Define convenience macros for custom read/write property callbacks
#define DEFINE_SET_DEVICE_PROPERTY_CALLBACK(PROP_ID) int CAT(set_, PROP_ID)(PropertyId_t propId, const uint8_t *buf, size_t size)
#define DEFINE_GET_DEVICE_PROPERTY_CALLBACK(PROP_ID) int CAT(get_,PROP_ID)(PropertyId_t propId, uint8_t *buf, size_t size)

#define CALL_GET_DEVICE_PROPERTY_CALLBACK(PROP_ID, buf, size) CAT(get_, PROP_ID)(PROP_ID, buf, size)
#define CALL_SET_DEVICE_PROPERTY_CALLBACK(PROP_ID, buf, size) CAT(set_, PROP_ID)(PROP_ID, buf, size)

// prop_handlers.h

#include "host_props.h"
#include "host_props_declare.h"

#define DEVICE_FIRMWARE_VERSION "1.0.0"

DEFINE_GET_DEVICE_PROPERTY_CALLBACK(ID_FW_VERSION)
{
    snprintf(buf, size, "%s", DEVICE_FIRMWARE_VERSION);
    return 0;
}

DEFINE_SET_DEVICE_PROPERTY_CALLBACK(ID_FW_VERSION)
{
    return -1;
}

DEFINE_GET_DEVICE_PROPERTY_CALLBACK(ID_DEVICE_TIME)
{
    return call_HAL_getTime(buf, size);
}

DEFINE_SET_DEVICE_PROPERTY_CALLBACK(ID_DEVICE_TIME)
{
    return call_HAL_setTime(buf, size);
}
```

Now, we only need to check if the property field is empty to either memcpy from config or call the callback function. That can be also done easily by `IIF` macro.

```c
// host_props.c

typedef struct DeviceConfig {
#define SKIP_SPECIAL_FIELD(propId)
#define DECLARE_CONFIG_FIELD(propId) PROP_TYPE(propId) PROP_NAME(propId) PROP_POSTFIX(propId);
#define CONFIG_FIELD(propId) IIF(IS_SPECIAL_PROPERTY(propId))(SKIP_SPECIAL_FIELD(propId), DECLARE_CONFIG_FIELD(propId))
    ALL_PROPERTIES(CONFIG_FIELD)
#undef CONFIG_FIELD
} DeviceConfig_t;

DeviceConfig_t g_deviceProperties;

int readProperty(PropertyId_t propId, uint8_t *buf, size_t size)
{
    switch(propId) {
#define READ_HOST_PROP(propId) \
    return memcpyWithSizeCheck(buf, size, &g_deviceProperties.PROP_NAME(propId), sizeof(g_deviceProperties.PROP_NAME(propId)));
#define READ_SPECIAL_PROP(propId) \
    return CALL_GET_DEVICE_PROPERTY_CALLBACK(propId, buf, size);
#define READ_PROP(propId, ...) \
        case propId: \
        IIF(IS_SPECIAL_PROPERTY(propId))(READ_SPECIAL_PROP(propId), READ_HOST_PROP(propId)) \
        break;
        ALL_PROPERTIES(READ_PROP)
#undef READ_PROP
        default:
            return -2;
    }
}

int writeProperty(PropertyId_t propId, uint8_t *buf, size_t size)
{
    switch(propId) {
#define WRITE_HOST_PROP(propId) \
return memcpyWithSizeCheck(&g_deviceProperties.PROP_NAME(propId), sizeof(g_deviceProperties.PROP_NAME(propId)), buf, size);
#define WRITE_SPECIAL_PROP(propId) \
    return CALL_GET_DEVICE_PROPERTY_CALLBACK(propId, buf, size);
#define WRITE_PROP(propId) \
        case propId: \
        IIF(IS_SPECIAL_PROPERTY(propId))(WRITE_SPECIAL_PROP(propId), WRITE_HOST_PROP(propId)) \
        break;
        ALL_PROPERTIES(WRITE_PROP)
#undef WRITE_PROP
        default:
            return -2;
    }
}
```

All good now! The property handling code is generated from the declarative property description.

The resulting code can be tested via [this link](https://godbolt.org/z/jKjjfMhqT).

# Another disclaimer

As I was writing before, this approach is cumbersome, fragile and prone to errors. The more robust approach is to use external code generators like m4, jinja2, mustache, etc. The code generation approach provides readable code, decoupling of models (property declaration) from the implementation (template), ease of debugging, ease of maintaining, and many more.

# Useful links

* https://en.wikibooks.org/wiki/C_Programming/Preprocessor_directives_and_macros#X-Macros
* https://embeddedartistry.com/blog/2020/07/27/exploiting-the-preprocessor-for-fun-and-profit/
* https://github.com/pfultz2/Cloak/wiki/C-Preprocessor-tricks,-tips,-and-idioms
* http://jhnet.co.uk/articles/cpp_magic
* https://www.boost.org/doc/libs/1_66_0/libs/preprocessor/doc/index.html
* https://github.com/mcinglis/libpp
* https://github.com/mcinglis/macrofun
* https://arne-mertz.de/2019/03/macro-evil/
* https://pigweed.dev/pw_preprocessor/
* http://saadahmad.ca/cc-preprocessor-metaprogramming-2/
* https://blog.robertelder.org/7-weird-old-things-about-the-c-preprocessor/
* https://github.com/Hirrolot/metalang99
* https://github.com/Hirrolot/awesome-c-preprocessor
* https://gustedt.wordpress.com/category/c99/preprocessor/

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
