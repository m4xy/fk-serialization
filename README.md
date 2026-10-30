# fk-serialization

Serialization for [fckt](https://github.com/m4xy/fckt), as kotlinx.serialization is Kotlin's: mark a
type `@Serializable`, and a serializer is written for it, which any format writes values with and
reads them back.

```kotlin
// src/shop/User.fk
import uk.m4xy.serialization.*

@Serializable type User(
  @SerialName("full_name") name: String,
  age: Int = 0,
  addresses: List<Address>,
  @Transient visits: Int = 0,
)
```

```kotlin
import uk.m4xy.serialization.json.Json

Json.encode(UserSerializer, user)      // {"full_name":"Ada","age":36,"addresses":[...]}
Json.decode(UserSerializer, text)      // Ok(user), or Error("shop.User needs full_name, which isn't there")
```

## Using it

fckt's compiler knows nothing of serialization. A [kiln](https://github.com/m4xy/kiln) project lists
fk-serialization's processor, which kiln runs before compiling it: it reads what the project
declares, as `fckt --symbols` writes it, and writes `UserSerializer` beside `User`, in
`build/generated`, which the project then compiles as its own code.

```kotlin
// build.fk
import uk.m4xy.kiln.fckt.*

project = Project(name = "shop", dependencies = [library("uk.m4xy:fk-serialization:0.1.0")])

module Serialized : FcktConfiguration {
  layout(): FcktLayout = FcktLayout(processors = [
    Processor("uk.m4xy.serialization.processor.SerializerProcessor", [library("uk.m4xy:fk-serialization-processor:0.1.0")]),
  ])
}

plugins = [Fckt(Serialized)]
```

A record is written as its fields, each by its serial name, its own unless `@SerialName` says
another, but for those marked `@Transient`, which must have a default; a field with a default can be
left out when it's read. A type whose cases carry nothing is an enumeration, written as its case's
name. Another type with cases is written as its case, by name, with that case's fields, every one
of which is read. A field is an `Int`, `Float`, `String`, `Boolean` or `Char`; a `List`, an
`Option`, or a `Map` from `String`, of one; or a type marked `@Serializable`, this project's or
another's whose serializer was written with it. What can't be written, as a field of a function's
type, a type with type parameters, or one only its module can build, the processor says, where it
is, and the build fails.

## Shapes

Every serializer describes the shape of what it writes: `UserSerializer.describe()` is a `Shape`, a
record with each field's name, shape, and whether it can be left out. A list, an option or a map is
the shape of what's in it, and another `@Serializable` type is a `ReferenceShape`, by its full name,
whose shape is worked out when it's asked for, so a type can hold itself. A serializer written by
hand describes nothing unless it says, `UnknownShape`. Shapes are what fk-http's OpenAPI documents are
made from.

## How it works

```
fk-serialization                uk.m4xy.serialization         What serializers are written with, and formats handle
                                ├── Serializer                A spec: serialize(value) with @Encoder, deserialize() with @Decoder
                                ├── Encoder, Decoder          Effects: the pieces of a value, which a format's handler writes or reads
                                ├── Encoding, Decoding        Lists, Options and Maps of what other serializers write; a record's fields
                                ├── IntSerializer, ...        The language's constants
                                ├── Serializable, SerialName, Transient
                                └── json                      Json.encode and Json.decode, handlers of Encoder and Decoder;
                                                              JsonValue, and JsonText, a Format of it

fk-serialization-processor      uk.m4xy.serialization.processor
                                ├── SerializerProcessor       A fckt.symbols Processor: a serializer for each @Serializable type
                                └── SerializerWriting         Writes one, as fckt's code
```

A serializer says what a value is a piece at a time, as `Encoder`'s operations: a record begun,
each field's name and value, then ended. It doesn't know what format it's written in: a format is
a handler of `Encoder`, which writes each piece as it has them, and of `Decoder`, which reads them,
a record's fields by name, in any order. So one serializer is every format's, and a format is
written once for every type: JSON's handlers build and read a `JsonValue`, which `JsonText` writes
and reads as text. `Decoder` does `@Fail`, so what isn't what's asked for says what it is, and
`Json.decode` returns it as an `Error`.

A serializer the processor writes is ordinary code, in `build/generated`, to read; one can also be
written by hand, as a module that follows `Serializer`, for a type the processor can't write one
for.

## Building

`kiln build` here builds and tests the library and the processor, and `kiln publishLocal` publishes
both to `~/.kiln/repository`. `example` is a project that uses them as one would, with tests of
what's written for its types: `kiln -C example build`.
