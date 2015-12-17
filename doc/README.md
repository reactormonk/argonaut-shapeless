# argonaut-shapeless

Automatic [argonaut](https://github.com/argonaut-io/argonaut) codec derivation with [shapeless](https://github.com/milessabin/shapeless)

[![Build Status](https://travis-ci.org/alexarchambault/argonaut-shapeless.svg)](https://travis-ci.org/alexarchambault/argonaut-shapeless)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/alexarchambault/argonaut-shapeless?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

This README documents the current *development* version of argonaut-shapeless,
which should be released after shapeless 2.3.0. For the stable version, see
the [0.3.x branch](https://github.com/alexarchambault/argonaut-shapeless/tree/0.3.x).

*argonaut-shapeless* is available for scala 2.10 and 2.11, and depends on argonaut 6.1 and shapeless 2.3.0-SNAPSHOT.

*argonaut-shapeless* is part of the shapeless ecosystem of
[Typelevel](http://typelevel.org/).

Hitorically, it is one of the very first projects to use of `Lazy` from shapeless 2.1,
that has made type class derivation with implicits much more robust.

## Usage

Add to your `build.sbt`
```scala
resolvers += Resolver.sonatypeRepo("snapshots")

libraryDependencies +=
  "com.github.alexarchambault" %% "argonaut-shapeless_6.1" % "1.0.0-SNAPSHOT"
```

If you are using scala 2.10.x, also add the macro paradise plugin to your build,
```scala
libraryDependencies +=
  compilerPlugin("org.scalamacros" % "paradise" % "2.0.1" cross CrossVersion.full)
```

The examples below assume you imported the content of
`argonaut`, `argonaut.Argonaut`, and `argonaut.Shapeless`, like
```tut:silent
import argonaut._, Argonaut._, Shapeless._
```

### Automatic codecs for case classes

```tut:silent
case class CC(i: Int, s: String)

// encoding
val encode = EncodeJson.of[CC]

val json = encode(CC(2, "a"))
json.nospaces == """{"i":2,"s":"a"}"""

// decoding
val decode = DecodeJson.of[CC]

val result = decode.decodeJson(json)
result == DecodeResult.ok(CC(2, "a"))
```

```tut:invisible
assert(result == DecodeResult.ok(CC(2, "a")))
```

The way argonaut-shapeless encodes case classes can be customized, see below.

### Automatic codecs for sealed traits

```tut:silent
sealed trait Base
case class First(i: Int) extends Base
case class Second(s: String) extends Base

// encoding
val encode = EncodeJson.of[Base]

val json = encode(First(2))
json.nospaces == """{"First":{"i":2}}"""

// decoding
val decode = DecodeJson.of[Base]

val result = decode.decodeJson(json)
result == DecodeResult.ok(First(2))
```

```tut:invisible
assert(result == DecodeResult.ok(First(2)))
```

### Default values

Like [upickle](https://github.com/lihaoyi/upickle-pprint/), *argonaut-shapeless*doesn't put fields equal to their default value in its output,

```tut:silent
case class CC(i: Int = 4, s: String = "foo")

CC().asJson.nospaces == "{}"
CC(i = 3).asJson.nospaces == """{"i":3}"""
CC(i = 4, s = "baz").asJson.nospaces == """{"s":"baz"}"""

"{}".decodeOption[CC] == Some(CC())
"""{"i":2}""".decodeOption[CC] == Some(CC(i = 2))
"""{"s":"a"}""".decodeOption[CC] == Some(CC(s = "a"))
```

```tut:invisible
assert(CC().asJson.nospaces == "{}")
assert(CC(i = 3).asJson.nospaces == """{"i":3}""")
assert(CC(i = 4, s = "baz").asJson.nospaces == """{"s":"baz"}""")

assert("{}".decodeOption[CC] == Some(CC()))
assert("""{"i":2}""".decodeOption[CC] == Some(CC(i = 2)))
assert("""{"s":"a"}""".decodeOption[CC] == Some(CC(s = "a")))
```

### Custom encoding of case classes

When encoding / decoding a case class `C`, argonaut-shapeless looks
for an implicit `JsonProductCodecFor[C]`, which has a field
`codec: JsonProductCodec`. A `JsonProductCodec` provides a general way of
encoding / decoding case classes.

The default `JsonProductCodecFor[T]` for all types provides
`JsonProductCodec.obj`, which encodes / decodes case classes as shown above.

This default can be changed, e.g. to convert field names to `serpent_case`,

```tut:invisible
implicit class readmeStringOps(val s: String) {
  def toSerpentCase: String = {
    // very naive
    // - stackoverflow with long string
    // - does not take into account multiple upper case characters in a row
    // - does not handle properly strings beginning with an upper case char.
    def helper(l: List[Char]): List[Char] =
      l match {
        case Nil => Nil
        case h :: t =>
          if (h.isUpper)
            '_' :: h.toLower :: helper(t)
          else
            h :: helper(t)
      }

    helper(s.toList).mkString
  }
}
```

```tut:silent
import argonaut.derive._

implicit def serpentCaseCodecFor[T]: JsonProductCodecFor[T] =
  JsonProductCodecFor(JsonProductObjCodec(_.toSerpentCase))

case class Identity(firstName: String, lastName: String)

Identity("Jacques", "Chirac").asJson.nospaces == """{"first_name":"Jacques","last_name":"Chirac"}"""
```

```tut:invisible
// assertion just above can be wrong because of the field order not preserved
// by default...
assert(
  PrettyParams.nospace
    .copy(preserveOrder = true)
    .pretty(Identity("Jacques", "Chirac").asJson) ==
  """{"first_name":"Jacques","last_name":"Chirac"}"""
)
```

```tut:invisible
implicit val serpentCaseCodecFor = 2 // mask implicit above
```

This can be changed for all types at once like just above, or only for specific
types, like
```tut:silent
implicit def serpentCaseCodecForIdentity: JsonProductCodecFor[Identity] =
  JsonProductCodecFor(JsonProductObjCodec(_.toSerpentCase))
```

```tut:invisible
// same comment as above
assert(
  PrettyParams.nospace
    .copy(preserveOrder = true)
    .pretty(Identity("Jacques", "Chirac").asJson) ==
  """{"first_name":"Jacques","last_name":"Chirac"}"""
)
```

### Custom encoding for sealed traits

When encoding / decoding a sealed trait `S`, argonaut-shapeless looks
for an implicit `JsonSumCodecFor[S]`, which has a field
`codec: JsonSumCodec`. A `JsonSumCodec` provides a general way of
encoding / decoding sealed traits.

The default `JsonSumCodecFor[S]` for all types `S` provides
`JsonSumCodec.obj` as `JsonSumCodec`, which encodes sealed traits
as illustrated above.

*argonaut-shapeless* provides `JsonSumCodec.typeField` as an alternative,
which discriminates the various cases of a sealed trait by looking
at a field, `type`, like

```tut:silent
implicit def typeFieldJsonSumCodecFor[S]: JsonSumCodecFor[S] =
  JsonSumCodecFor(JsonSumCodec.typeField)

sealed trait Base
case class First(i: Int) extends Base
case class Second(s: String) extends Base

val f: Base = First(2)

f.asJson.nospaces
f.asJson.nospaces == """{"type":"First","i":2}"""
// instead of the default """{"First":{"i":2}}"""
```

```tut:invisible
assert(
  PrettyParams.nospace
    .copy(preserveOrder = true)
    .pretty(f.asJson) ==
  """{"type":"First","i":2}"""
)
```

### Proper handling of custom codecs

Of course, if some of your types already have codecs (defined in their
companion object, or manually imported), these will be given the priority
over the ones derived by argonaut-shapeless, like

```scala
import argonaut._, Argonaut._, Shapeless._

case class Custom(s: String)

object Custom {
  implicit def encode: EncodeJson[Custom] =
    EncodeJson.of[String].contramap[Custom](_.s)
  implicit def decode: DecodeJson[Custom] =
    DecodeJson.of[String].map(Custom(_))
}
```

```tut:invisible
// can't define a case class and its companion from the REPL it seems
object Defn {
  case class Custom(s: String)

  object Custom {
    implicit def encode: EncodeJson[Custom] =
      EncodeJson.of[String].contramap[Custom](_.s)
    implicit def decode: DecodeJson[Custom] =
      DecodeJson.of[String].map(Custom(_))
  }
}

import Defn._
```

```tut:silent
Custom("a").asJson.nospaces == """"a""""
""""b"""".decodeOption[Custom] == Some(Custom("b"))
```

```tut:invisible
assert(Custom("a").asJson.nospaces == """"a"""")
assert(""""b"""".decodeOption[Custom] == Some(Custom("b")))
```

### refined module

*argonaut-shapeless* also has a module to encode/decode types from
[refined](https://github.com/fthomas/refined).

Add it to your dependencies with
```scala
libraryDependencies += "com.github.alexarchambault" %% "argonaut-refined_6.1" % "1.0.0-SNAPSHOT"
```

Use like
```tut:silent
import argonaut._, Argonaut._, Shapeless._, Refined._
import eu.timepit.refined._
import eu.timepit.refined.api.Refined

case class CC(
  i: Int Refined numeric.Greater[W.`5`.T],
  s: String Refined string.StartsWith[W.`""`.T]
)

CC(
  refineMV(6),
  refineMV("Abc")
).asJson.nospaces == """{"i":6,"s":"Abc"}""" // fields are encoded as their underlying type

"""{"i": 7, "s": "Bcd"}""".decodeOption[CC] == Some(CC(refineMV(7), refineMV("Bcd")))
"""{"i": 4, "s": "Bcd"}""".decodeOption[CC] == None // fails as the provided `i` doesn't meet the predicate ``GreaterThan[W.`5`.T]``
```

```tut:invisible
assert("""{"i": 7, "s": "Bcd"}""".decodeOption[CC] == Some(CC(refineMV(7), refineMV("Bcd"))))
assert("""{"i": 4, "s": "Bcd"}""".decodeOption[CC] == None // fails as the provided `i` doesn't meet the predicate ``GreaterThan[W.`5`.T]``)
```

## License

Released under the BSD license. See LICENSE file for more details.

Based on an early (non `Lazy`-based) automatic codec derivation in argonaut
by [Maxwell Swadling](https://github.com/maxpow4h),
[Travis Brown](https://github.com/travisbrown), and
[Mark Hibberd](https://github.com/markhibberd).
