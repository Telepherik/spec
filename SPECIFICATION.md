# Telepherik specification

## Introduction

This document describes the Telepherik standard for definition of communication protocols. A telepherik definition file describes types, messages and exchanges which compose a protocol.
A telepherik implementation allows, in some way or another, for developers to use the protocol in the programming language or toolkit of their choice. Telepherik makes a distinction between "server" and "client",
expecting a client to always initiate communication with a server. Telepherik is designed for use primarily when developing both a server and client in different languages, such as is often the case when
developing web applications. It is also designed to be as language-agnostic as possible by making as few assumptions as possible regarding specific capabilities of any given language or platform.

## Conformity

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in
[RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Document language

Telepherik documents SHOULD be written in [KDL version 2](https://github.com/kdl-org/kdl/blob/2.0.0/SPEC.md), implementations MUST support KDL version 2 as a document language, they
MAY support additional document languages. This specification will use examples written in KDL as well as its semantics.

## Version

This document describes Telepherik version `a1`. A Telepherik document MUST define its version in a top-level node named `telepherik_version`.
Implementations MUST verify the `telepherik_version` node of a Telepherik document to ensure that it is `a1`, otherwise this document
does not apply. If an implementation does not recognize the version of a document, it MUST report an error to the user. Implementations MAY support other versions of the standard.
Implementations SHOULD always support the latest version of the standard.

Example:

```kdl
telepherik_version a1
```

## Transport

A Telepherik document MUST define a transport in a top-level node named
`transport`. Implementations MAY support any transport suitable for a binary stream. A transport's name MUST be case-insensitive.

The following transports are RECOMMENDED:

- `tcp` ([Transport Layer Protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol))
- `ws` ([WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API))

Example:

```kdl
transport ws
```

## Types

A Telepherik document MUST define types in a top-level node named `types`. Each child node of this node represents an individual type. The name of each node represents the name of the type. Type names SHALL NOT include any of the following reserved characters: `<>,?!@&:.|`. The first value, known as the supertype, is one of the following:

- `int`, meaning this type represents an integer number.
- `real`, meaning this type represents a real number.
- `enum`, meaning this type represents one of a set of discrete values.
- `string`, meaning this type represents a string of text.
- `binary`, meaning this type does not represent another category of data.
- `struct`, meaning this type represents structured data including other types.

If an unknown supertype is found, implementations MUST report an error to the user.

The supertype defines which additional properties are required and how implementations must interface with user-written code.

If multiple types with the same name are defined, implementations MUST report an error to the user.

If the implementation's language has a type system capable of it, it SHOULD define new data types for the types defined, even when they are "backed" by the same data type.

Implementations SHOULD support all supertypes and all combinations of properties for each supertype. If they are unable to, they MUST report an error to the user if an unsupported type is defined.

### Default properties

In order to reduce repetition, default properties MAY be specified with top level nodes named `default_prop` in the top level of the document. Each node named `default_prop` MUST have 3 values.

The first value MUST be one of the supertypes described in [Types](#types). The second value MUST be the name of one of the properties described in the relevant sub-section of [Types](#types). The third value MUST be a valid value for the relevant property. If a default value for a property is defined this way, specifying it in type definitions is OPTIONAL, if the property is not explicitly defined it is implicitly assumed to take the default value.

For example, the following two definitions are equivalent:

```kdl
types {
    i32 int size=32 endianness=big signed=#true
    u32 int size=32 endianness=big signed=#false
}
```

```kdl
default_prop int endianness big

types {
    i32 int size=32 signed=#true
    u32 int size=32 signed=#false
}
```

If multiple default values are defined for the same property, the implementation MUST report an error to the user.

### `int`

`int` types represent an integer number.

#### Properties

- `size`, an integer value which specifies the size, in bits, that this type takes up. Implementations SHOULD use this property to choose how to represent this type in memory. Implementations MUST report an error to the user if this value is less than or equal to zero, non-integer or not a multiple of 8.
- `endianness`, one of `big` or `small`. If the value is `big`, this type is big-endian. If the value is `small`, this type is small-endian. Implementations MUST report an error to the user if this value is not `big` or `small`.
- `signed`, a boolean value (one of `#true` or `#false`). If the value is true, this type represents a signed number encoded using [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement). If it is false, this type represents an unsigned number.

#### Interface

Implementations MUST make values of this type available as integers. They MAY use different integer types depending on the type's property, for example if they are made for a language which uses different data types for different sizes of integers.

#### Example

```kdl
types {
    i32 int size=32 endianness=big signed=#true
    u32 int size=32 endianness=big signed=#false
}
```

### `real`

`real` types represent a real number. They are based on the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) standard's representation of floating-point numbers.

#### Properties

- `size`, one of `16`, `32`, `64`, `128` or `256`. This property represents the size of the type in bits and corresponds to the data formats Half, Single, Double, Quadruple and Octuple as described in the [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) standard.

#### Interface

Implementations MUST make values of this type available as real numbers and SHOULD use [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) data types to store them. They MAY use different data types depending on the type's property, for example if they are made for a language which uses different data types for different sizes of floating point numbers.

#### Example

```kdl
types {
    float real size=32
    double real size=32
}
```

### `enum`

`enum` types represent one of a set of discrete values. In transfer, these values MUST be represented as an unsigned big endian integer with a size in bytes equal to `2^(max(3, ceil(log2(log2(n)))))` where `n` is the number of possible values (in other words, the least power of two greater than 8 with enough bits to store all possible values), where the first possible value is 0, the next one is 1, and so forth. See [`int`](#int) for more information on integers.

This supertype differs from other supertypes because, instead of properties, it MUST have any strictly positive number of values. Each value represents a separate variant of the enum.

#### Interface

Implementations SHOULD make values of this type available as an enumerated type. Failing that, they MUST make values of this type available as strings.

If an `enum` type has exactly two variants, one named `true` and one named `false`, implementations MUST make values of this type available as a boolean instead.

#### Example

```kdl
// Both types below would be transferred as an unsigned 8-bit integer
types {
    bool enum false true
    color enum red green blue yellow cyan magenta white
}
```

### `string`

`string` types represent a string of text. They may be of a fixed or variable length.

#### Properties

- `size`, either a positive integer or the name of an integer type. This property either represents the length of a string in bytes or the type which encodes the length of a variable-length string in bytes. If this is a variable-length string, the string data MUST be prefixed by data of the length type. Implementations MAY report a warning to the user if the type chosen is signed, since a negative length is nonsensical.
- `encoding`, `utf-8`. Represents what text encoding to use for this string. Implementations MUST support [UTF-8](https://en.wikipedia.org/wiki/UTF-8) and MAY support additional text encodings. Case-insensitive.

#### Interface

Implementations MUST make values of this type available as strings of unicode text.

#### Example

```kdl
types {
    i64 int size=64 endianness=big signed=#false

    uuid string size=36  encoding=utf-8 // Fixed length
    name string size=i64 encoding=utf-8 // Variable length
}
```

### `binary`

`binary` types represent binary data. They may be of a fixed or variable length.

#### Properties

- `size`, either a positive integer or the name of an integer type. This property either represents the length of the binary or the type which encodes the length of the binary. If this binary is of variable length, the binary data MUST be prefixed by data of the length type. Implementations MAY report a warning to the user if the type chosen is signed, since a negative length is nonsensical.

#### Interface

Implementations MAY make this data available in any way that is fit for the specific language or platform.

#### Example

```kdl
types {
    i64 int size=64 endianness=big signed=#false

    signature binary size=2048 // Fixed length
    image_png binary size=i64 // Variable length
}
```

### `struct`

`struct` types represents structured data which itself includes different types.

This supertype is different from other supertypes because, instead of properties, it MUST have at least one child node. The child nodes represent the fields of the structure.

Each child node MUST have a name and exactly one value, that value MUST be the name of a type. In transfer, each child MUST be transferred in the order that it is defined.

`struct` types MUST NOT be recursive, they MUST NOT reference themselves or reference another `struct` type which references them.

#### Interface

Implementations SHOULD define a structure data type if their language supports them for interfacing with user code. If that is impossible, they MUST use a key-value data type instead. While the ordering of fields is important during transfer, that information MAY be present or absent when interfacing with user code.

#### Example

```kdl
types {
    u64 int size=64 endianness=big signed=#false
    double real size=64

    position struct {
        x double
        y double
        z double
    }

    string string size=u64 encoding=utf-8

    person struct {
        name string
        age u64
        position position
    }
}
```

### Type variants

Type variants are predefined in Telepherik. The following variants MUST be supported by implementations:

- `optional<T>` for optional values of type `T`
- `list<T, U>` for lists of values of type `T`. If `U` is a number, the list's length is fixed to `U`. If `U` is a type of supertype `int`, the list's length is encoded with that type.

#### `optional<T>`

`optional<T>` represents an optional value of type `T`. In transfer, values of this type MUST be one of the following:

- A null byte (0x00)
- A 1 byte (0x01) followed by the data to transfer

Implementations SHOULD make this data available as a union type or a nullable type, if possible.

#### `list<T, U>`

`list<T, U>` represents a list of values of type `T`.

If `U` is a number, the list's length is fixed to `U`. The list MUST be transferred as `U` consecutive instances of `T`.

If `U` is a type of supertype `int`, the list's length is variable and encoded with type `U`. The list MUST be prefixed with its length written as `U`.

If `U` is neither, implementations MUST report an error to the user.

## Messages

A Telepherik document MAY define messages in two top-level nodes named `serverbound_messages` and `clientbound_messages`, the first one represents serverbound messages that are sent by the client, the second one represents clientbound messages that are sent by the server. Both of those nodes contain any number of child nodes represening messages. Each of these message nodes MUST have a name and child nodes representing their fields.

Semantically, a message represents data to be sent without waiting for a response.

Each field is represented by a node with a name and exactly one value which represents its type. A message may have any number of fields.

Examples:

```kdl
serverbound_messages {
    move { // 0x0
        x float
        y float
        z float
    }
    rotate { // 0x1
        pitch float
        yaw float
    }
    shoot // 0x2
}

clientbound_messages {
    player_move { // 0x0
        id player_id
        x float
        y float
        z float
    }
    npc_move { // 0x1
        id npc_id
        x float
        y float
        z float
    }
}
```

The fields of a message MUST be transferred in the order they are defined, the data of a message MUST be prefixed by its index (the first message defined is 0, the second is 1, and so on) written as an unsigned big-endian integer with a size in bytes equal to `2^(max(3, ceil(log2(log2(n)))))` where `n` is the number of messages defined.
