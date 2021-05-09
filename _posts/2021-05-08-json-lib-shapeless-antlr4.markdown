---
layout: single
title:  "How to create a small json lib using antlr and shapeless"
date:   2021-05-08 22:17:26 +0300
categories: scala shapeless derivation antlr
---
In this article i will show how antlr4 and shapeless can be used to create a small json library (not for production, of course ^_^)
with ability to decode arbitrary json strings into case classes and encode them back with some scala magic.

## Project setup
Let's begin with a project setup.

Generally speaking, it doesn't really matter which IDE you will use, but i'll use a Intellij Idea. Community edition is more than enough for it. Also, i recommend to instal [antlr4 plugin](https://plugins.jetbrains.com/plugin/7358-antlr-v4) for intellij – it's not necessary, but it really helps to create and debug antlr grammar.

Now we are ready to create a new project. For project building i'll use `sbt`, but it is also possible to use `maven` or `gradle`.
After project creation we need to add antlr4 compiler plugin – [sbt-antlr4](https://github.com/ihji/sbt-antlr4). Also, we will need a `shapeless` for codec derivation, so let's add this library too. After all, `plugins.sbt` and `build.sbt` will look like this:

```scala
// plugins.sbt
addSbtPlugin("com.simplytyped" % "sbt-antlr4" % "0.8.2")
```

```scala
// build.sbt
lazy val root = (project in file("."))
  .enablePlugins(Antlr4Plugin)
  .settings(
    Antlr4 / antlr4Version := "4.7.2",
    Antlr4 / antlr4PackageName := Some("jsonserde.antlr"),
    Antlr4 / antlr4GenListener := false,
    Antlr4 / antlr4GenVisitor := true,
    Antlr4 / antlr4TreatWarningsAsErrors := true,
    libraryDependencies ++= Seq(
      "com.chuusai" %% "shapeless" % "2.3.3"
    ),
    // other settings
  )
```

- `antlr4Version` – antlr4 compiler version
- `antlr4PackageName` – package name of generated code
- `antlr4TreatWarningsAsErrors` – warnings can be dangerous, so let's treat them as errors

`antlr4GenListener` and `antlr4GenVisitor` are the most interesting part. To be able to process parsed data in our code we can use different approaches: listener pattern or visitor pattern. For our small library i think that visitor is more native and easier, but it's also possible to achieve the same results using listener. The most important difference between them is that using visitor you can control node visiting whereas using listener you should react on each call made by ANLTR mechanism.

By default, antlr4 grammars should be placed in `antlr4` directory and as we didn't change it, the final project structure should look like this:
```
root
|------------ src
|             |
|             |---- main
|             |     |
|             |     |--- scala
|             |     |--- antlr4
|             |
|build.sbt
```
## ANTLR4 grammar
To be able to decode json string we need a parser. We can code it from scratch but we have an antlr4, so we will use this tool for parsing.

> This is not the best solution for json parsing, especially for load-heavy systems, and should be considered as an example only.

First of all, we need a grammar. I think, that the best place for exploring grammars and take some ideas is [grammars-v4](https://github.com/antlr/grammars-v4). There are a lot of grammar examples, but we need a special one – [json](https://github.com/antlr/grammars-v4/blob/master/json/JSON.g4).

Our initial grammar gently copied from [grammars-v4](https://github.com/antlr/grammars-v4/blob/master/json/JSON.g4) will look like this:
```antlrv4
grammar JSON;

// Parser rules

json
   : value
   ;

obj
   : '{' pair (',' pair)* '}'
   | '{' '}'
   ;

pair
   : STRING ':' value
   ;

arr
   : '[' value (',' value)* ']'
   | '[' ']'
   ;

value
   : STRING
   | NUMBER
   | obj
   | arr
   | 'true'
   | 'false'
   | 'null'
   ;

// Lexer rules
STRING
   : '"' (ESC | SAFECODEPOINT)* '"'
   ;


fragment ESC
   : '\\' (["\\/bfnrt] | UNICODE)
   ;
fragment UNICODE
   : 'u' HEX HEX HEX HEX
   ;
fragment HEX
   : [0-9a-fA-F]
   ;
fragment SAFECODEPOINT
   : ~ ["\\\u0000-\u001F]
   ;


NUMBER
   : '-'? INT ('.' [0-9] +)? EXP?
   ;


fragment INT
   : '0' | [1-9] [0-9]*
   ;

fragment EXP
   : [Ee] [+\-]? INT
   ;

WS
   : [ \t\n\r] + -> skip
   ;
```

We will change it a little bit soon, but for now let's look on what's going on there.
In general, grammar consists of [parser](https://github.com/antlr/antlr4/blob/master/doc/parser-rules.md) and [lexer](https://github.com/antlr/antlr4/blob/master/doc/lexer-rules.md) rules.

### Lexer
Lexer rules specify token definitions. The syntax for lexer rules resembles the syntax for parser rules, but with some differences -- lexer rules must begin with uppercase letter, for example. Also, it is possible to define a `fragment`. Fragments are not visible for parsers, but they can help in token recognition for lexer.

```antlrv4
NUMBER
   : '-'? INT ('.' [0-9] +)? EXP?
   ;


fragment INT
   : '0' | [1-9] [0-9]*
   ;

fragment EXP
   : [Ee] [+\-]? INT
   ;  
```

`fragment INT` is a rule for an integer: it can be zero OR be non-zero and begin with [1-9] + zero or more other digits:
- `0` – ok
- `123` – ok
- `01` – not ok

`fragment EXP` is a rule for exponent part. It is interesting example which shows us that we can also combine fragments. Correct examples of exponent part are `e1`, `e+321`, `e-3`.

As mentioned before, this rule is not visible for parser rules, but only for lexer rules. The next lexer rule uses this fragment. `NUMBER` is a rule for a number. Numbers can be negative (optional '-'), requires at least one `INT` fragment, can have optional floating point part `('.' [0-9] +)?` and optional exponent part `EXP?`.

Now let's move on to the parser rules.

### Parser
Parser rules are the heart of our grammar. By defining parser rules we define how found (by lexer) tokens combine with each other.

Each combination has it's own complexity, so, don't overcomplicate it with many recursive rules :)

```antlrv4
value
   : STRING
   | NUMBER
   | obj
   | arr
   | 'true'
   | 'false'
   | 'null'
   ;
```

The first rule is a `value`. As we know, json consists of string literals, numbers, arrays of any values, objects with nested values, boolean values and null. In this example `true`, `false` and `null` defined as literals, but it could be done using explicit lexer rule for them. So, in this rule we state that value is a string OR a number OR etc.

```antlrv4
arr
   : '[' value (',' value)* ']'
   | '[' ']'
   ;
```

As mentioned earlier, json can has arrays. Array can be empty `[]` or non empty `[value, value, ...]`. If array is not empty then it should have at least one value and optionally more values.

```antlrv4v
obj
   : '{' pair (',' pair)* '}'
   | '{' '}'
   ;

pair
   : STRING ':' value
   ;
```

Json object is a value which consists of a key-value pairs. Key always a string, but value can be any type. Object as an array can be empty `{}` or non empty `{"f1": "v1"}`.

Finally, there is a rule for the whole json string:
```antlrv4
json
   : value
   ;
```

It is just a single value.

### Visitor implementation
Remember that we wanted to change grammar a little bit? That's the perfect time for this. We'll change rules by adding a labels to each of their alternative:

```antlrv4
obj
   : '{' pair (',' pair)* '}'   #NotEmptyObject
   | '{' '}'                    #EmptyObject
   ;

pair
   : key=STRING ':' value
   ;

arr
   : '[' value (',' value)* ']' #NotEmptyArray
   | '[' ']'                    #EmptyArray
   ;

value
   : STRING     #StringValue
   | NUMBER     #NumberValue
   | obj        #ObjectValue
   | arr        #ArrayValue
   | 'true'     #True
   | 'false'    #False
   | 'null'     #Null
   ;
```

These labels are very useful which we will see soon.

To generate code write a sbt command: `sbt antlr4Generate`. This will generate a `java` code:
```
JSONBaseVisitor
JSONLexer
JSONParser
JSONVisitor
```

You can explore the sources, but for now we need to extend `JSONBaseVisitor` and override some methods.

First of all, let's define our json ADT. I will not overcomplicate it with some number decoding magic – it should be done very carefully, but for now we define that all numbers is a `BigDecimal`.
```scala
sealed trait Json

final case class JsonObj(fields: List[(String, Json)]) extends Json // or stick to Map[String, Json]

object JsonObj {
  final val EMPTY: JsonObj = JsonObj(List.empty)
}

final case class JsonArray(values: Vector[Json]) extends Json

object JsonArray {
  final val EMPTY: JsonArray = JsonArray(Vector.empty)
}

final case class JsonBoolean(value: Boolean) extends Json

final case class JsonString(value: String) extends Json

case object JsonNull extends Json

final case class JsonNumber(value: BigDecimal) extends Json

object JsonNumber {
  def apply(value: BigDecimal): JsonNumber = new JsonNumber(value)
  def apply(value: Byte): JsonNumber = new JsonNumber(BigDecimal(value))
  def apply(value: Short): JsonNumber = new JsonNumber(BigDecimal(value))
  def apply(value: Int): JsonNumber = new JsonNumber(BigDecimal(value))
  def apply(value: Long): JsonNumber = new JsonNumber(BigDecimal(value))
  def apply(value: Float): JsonNumber = new JsonNumber(BigDecimal(value))
  def apply(value: Double): JsonNumber = new JsonNumber(BigDecimal(value))
  def apply(value: BigInt): JsonNumber = new JsonNumber(BigDecimal(value))
}
```

Now we a ready to extend `JSONBaseVisitor`. The implementation will be surprisingly small:
```scala
final class JsonVisitor extends JSONBaseVisitor[Json] {
  override def visitErrorNode(node: ErrorNode): Json = throw new RuntimeException("Parse error")

  override def visitJson(ctx: JSONParser.JsonContext): Json = visit(ctx.value())

  override def visitNotEmptyObject(ctx: JSONParser.NotEmptyObjectContext): Json = JsonObj(
    ctx.pair().asScala.map(pair => (removeQuotes(pair.key.getText), visit(pair.value()))).toList
  )

  override def visitEmptyObject(ctx: JSONParser.EmptyObjectContext): Json = JsonObj.EMPTY

  override def visitNotEmptyArray(ctx: JSONParser.NotEmptyArrayContext): Json = JsonArray(ctx.value().asScala.map(visit(_)).toVector)

  override def visitEmptyArray(ctx: JSONParser.EmptyArrayContext): Json = JsonArray.EMPTY

  override def visitStringValue(ctx: JSONParser.StringValueContext): Json = JsonString(removeQuotes(ctx.STRING().getSymbol.getText))

  override def visitNumberValue(ctx: JSONParser.NumberValueContext): Json = JsonNumber(BigDecimal(ctx.NUMBER().getSymbol.getText))

  override def visitObjectValue(ctx: JSONParser.ObjectValueContext): Json = visit(ctx.obj())

  override def visitArrayValue(ctx: JSONParser.ArrayValueContext): Json = visit(ctx.arr())

  override def visitTrue(ctx: JSONParser.TrueContext): Json = JsonBoolean(true)

  override def visitFalse(ctx: JSONParser.FalseContext): Json = JsonBoolean(false)

  override def visitNull(ctx: JSONParser.NullContext): Json = JsonNull

  private def removeQuotes(str: String): String =
    str.substring(1, str.length - 1)
}
```

Remember we have added a labels to parser rules? As you can see, `JSONBaseVisitor` has methods with names like labels. That's very helpful. Also, there is additional method `removeQuotes` – i didn't come up with a better solution to remove quotes, but this method is only for that. Without this method, string values in scala would be `"value"`.

Usage of this simple decoder:
```scala
  val jsonVisitor: JsonVisitor = new JsonVisitor
  val lexer = new JSONLexer(CharStreams.fromString(str))
  val tokenStream = new CommonTokenStream(lexer)
  val parser = new JSONParser(tokenStream)

  jsonVisitor.visit(parser.json())
```

## Shapeless magic
For now, we can only decode json string into `Json` ADT, but we want to decode it into a case class instance and encode case class instance back into json. Also, we don't want to configure decoders and encoders manually, but automatically. All of this can be achieved using `shapeless`.

First of all, we will define a `JsonReader[A]` and `JsonWriter[A]` for some basic types. There is no any kind of magic yet, just a bunch of instances for some basic types:

```scala
trait JsonReader[A] {
  def read(json: Json): Either[Throwable, A]
}

object JsonReader {
  final implicit val jsonStringRead: JsonReader[String] = {
    case JsonString(value) => Right(value)
    case _                 => Left(new RuntimeException("String"))
  }

  // ...
}

trait JsonWriter[A] {
  def write(value: A): Json
}

object JsonWriter {
  final implicit val jsonStringWrite: JsonWriter[String] = (value: String) => JsonString(value)

  // ...
}
```

Using this type classes we already can decode/encode simple data types:
```scala
import JsonReader.jsonIntRead
import JsonWriter.jsonIntWrite

val i = 10
val json = jsonIntWrite.write(i)

assertResult(i)(jsonIntRead.read(json))
```

So, consider this case class:
```scala
case class A(f1: Int, f2: String, f3: Option[Long])
```
This case class consists of three fields: integer, string and optional long. We can decode/encode each of them separately (except optional types, because we didn't define reader and writer for them yet). Using `JsonReader[A]` and `JsonWriter[A]` we can process single data type but not a product of data types. Let's add another layer of abstraction for it:

```scala
trait Decoder[A] {
  def decode(json: Json): Either[Throwable, A]
}

trait Encoder[A] {
  def encode(value: A): Json
}
```

The traits look the same with the previous ones, but the purpose of them is a little bit different.
As a playground we will use [scastie](https://scastie.scala-lang.org/nryanov/p4ixGZl3SA676gSQgj4j9w/25). As you can see, that's not too many lines of code for auto codec derivation.

Let's start from `Encoder[A]` because it simpler than `Decoder[A]`. In companion object we defined summoner method `apply` and some implicits for derivation:
```scala
implicit val unitWriter: Encoder[Unit] = _ => JsonObj.EMPTY

implicit def fromJsonWriter[A](implicit writer: JsonWriter[A]): Encoder[A] = writer.write

implicit def optionalEncoder[A](implicit encoder: Encoder[A]): Encoder[Option[A]] = {
  case Some(value) => encoder.encode(value)
  case None        => JsonNull
}
```

For `Unit` we will just return an empty object, but you can change it and return JsonNull or whatever you want. Also we want to be able to use already defined `JsonReader` instances for basic types and to do so we define an implicit converter from `JsonReader` to `Encoder`. Finally, we should be able to encode optional data types, so we also define an implicit converter from `Encoder[A]` to `Encoder[Option[A]]`.

Now let's move on into the `EncoderLowPriorityInstances`. First of all, we use this trait because we don't want to create ambiguities for compiler when it try to find an instance for some type. Because of it, we put it into a separate trait (probably, an overkill for this example, but a good practice). The magic is here:

```scala
final implicit def genericEncoder[A, H <: HList](
  implicit gen: LabelledGeneric.Aux[A, H],
  hEncoder: Lazy[Encoder[H]]
): Encoder[A] = (value: A) => hEncoder.value.encode(gen.to(value))

final implicit val hnilEncoder: Encoder[HNil] = (_) => JsonObj.EMPTY

final implicit def hlistEncoder[K <: Symbol, H, T <: HList](
  implicit witness: Witness.Aux[K],
  hEncoder: Lazy[Encoder[H]],
  tEncoder: Lazy[Encoder[T]]
): Encoder[FieldType[K, H] :: T] = { (hlist) =>
  val head = hEncoder.value.encode(hlist.head)
  val tail = tEncoder.value.encode(hlist.tail)
  val fieldName: String = witness.value.name

  JsonObj(List((fieldName, head)) ::: tail.asInstanceOf[JsonObj].fields)
}
```

This is a typical structure for auto derivation using shapeless. `genericEncoder` is used to derive instances for case classes. `LabelledGeneric` is used to allow to get field names.
Type `A` -- type of case class. `H` -- representation of a case class. Example:
```scala
case class A(f1: Int, f2: String, f3: Long)
// H <: HList
Int :: String :: Long :: HNil
```

`hnilEncoder` is used to encode `HNil` instance. It will never be called, but is needed for derivation.
Finally, `hlistEncoder` is used to derive encoder for our representation.
`witness: Witness.Aux[K]` is needed to get a field name. `hEncoder` and `tEncoder` are encoders for head (`H`) of our case class and remaining tail (`T <: HList`). The implementation is basically encode both part then combine them into a single `JsonObject`. Return type of this method is not a `Encoder[H :: T]`, but a `Encoder[FieldType[K, H] :: T]`.
> If we combine Witness and FieldType, we get something very compelling—the ability to extract the field name from a tagged field. We use `FieldType` and `::` in the result type to declare the relationships between the three, and we use a Witness to access the runtime value of the type name.

And that's it! The `Decoder` is the same with only difference in implementation details.

But that's still not super cool implementation. What if we have a json without some field but our case class has a default value for it? Or we want to rename field names in json? All of this also could be done using `shapeless`.

## Shapeless magic – improvements
For field renaming and default values we will use `Decoder` as an example [(scastie playground)](https://scastie.scala-lang.org/nryanov/XZGIgFZ9SUWUBjxQ6zy3xA/10).

For custom field names we will use annotation, so let's define it:
```scala
import scala.annotation.StaticAnnotation

final case class FieldName(value: String) extends StaticAnnotation

// example usage
case class A(@FieldName("customField1") f1: String, f2: Int, f3: Option[Long])
```

Now we need an additional trait which should help us to handle not only current field value, but also custom field name and default value.

```scala
trait DecoderWithMeta[A, B, C] {
  def decode(json: Json, defaults: B, fieldNames: C): Either[Throwable, A]
}
```

The instances in companion object remain the same, but in trait will be changes a little bit:
```scala
final implicit def jsonGenericDecoder[A, H <: HList, HD <: HList, FH <: HList](
  implicit gen: LabelledGeneric.Aux[A, H],
  defaults: Default.AsOptions.Aux[A, HD],
  annotations: Annotations.Aux[FieldName, A, FH],
  hDecoder: Lazy[DecoderWithMeta[H, HD, FH]]
): Decoder[A] = json => hDecoder.value.decode(json, defaults(), annotations()).map(gen.from)

final implicit val hnilDecoder: DecoderWithMeta[HNil, HNil, HNil] = (_, _, _) => Right(HNil)

final implicit def hlistDecoder[K <: Symbol, H, T <: HList, TD <: HList, FH <: Option[FieldName], FT <: HList](
  implicit witness: Witness.Aux[K],
  hDecoder: Lazy[Decoder[H]],
  tDecoder: Lazy[DecoderWithMeta[T, TD, FT]]
): DecoderWithMeta[FieldType[K, H] :: T, Option[H] :: TD, FH :: FT] = { (json, defaults, fieldNames) =>
  val fieldName: String = fieldNames.head.map(_.value).getOrElse(witness.value.name)

  json match {
    case JsonObj(fields) =>
      val jsonField = fields.collectFirst {
        case (str, json) if str == fieldName => json
      }

      jsonField.map(hDecoder.value.decode)
      val head: Either[Throwable, H] = jsonField match {
        case Some(value) => hDecoder.value.decode(value)
        case None =>
          defaults.head match {
            case Some(default) => Right(default)
            case None          => hDecoder.value.decode(null)
          }
      }
      val tail: Either[Throwable, T] = tDecoder.value.decode(json, defaults.tail, fieldNames.tail)

      for {
        h <- head
        t <- tail
      } yield field[K](h) :: t
    case _ => Left(new RuntimeException("Incorrect data: expected JsonObject"))
  }
}
```

Important note is that for our decoder now we need not only `A, H <: HList`, but also `HD <: HList, FH <: HList`. In this case `HD` and `FH` are HList for default values and field names respectively. Also, there are some new instances in the method signature:
```scala
defaults: Default.AsOptions.Aux[A, HD],
annotations: Annotations.Aux[FieldName, A, FH],
```

As name declares, their are needed for processing default values and annotations (`FieldName` in this case). The rest is pretty similar: we still need an instance of decoder for `HNil` and for `HList`:
```scala
final implicit def hlistDecoder[K <: Symbol, H, T <: HList, TD <: HList, FH <: Option[FieldName], FT <: HList](
  implicit witness: Witness.Aux[K],
  hDecoder: Lazy[Decoder[H]],
  tDecoder: Lazy[DecoderWithMeta[T, TD, FT]]
): DecoderWithMeta[FieldType[K, H] :: T, Option[H] :: TD, FH :: FT] = { (json, defaults, fieldNames) =>
  ...
}
```

`witness` and `hDecoder` should be already familiar. `tDecoder` was changed and also there is `TD <: HList, FH <: Option[FieldName], FT <: HList`. So, `tDecoder` now has a type `DecoderWithMeta` -- it is needed because we decode our object step by step (of field by field) and we need to pass remaining json, default values and custom field names to the tail decoder and tail decoder should be able to accept and process them.
`TD <: HList` is a type for default values. `FH <: Option[FieldName], FT <: HList` -- field names current head (`FH`) and tail (`FT`).
The return type is `FieldType[K, H] :: T, Option[H] :: TD, FH :: FT`. You can read it like this:
```
Decoder for a head of case class + its' tail
with optional default value for its' head and tail
with optional custom field name for its' head and tail
```

## Conclusion
At the end, we have a small library for parsing and decoding json. Consider it as a proof of concept – there are still many issues like:
- No check for custom field names duplicates
- Slow deserialization due to processing the whole json object again and again
- etc.

In the end the main purpose was no to create a library, but to show that antlr and shapeless could be combined giving in result an awesome outcome and i believe that it will help you to start exploring `antlr4` and `shapeless`.

The source code of a whole project could be found here: https://github.com/nryanov/json-serde

## Resources
- Antlr4 doc: https://github.com/antlr/antlr4/tree/master/doc
- The Type Astronaut's Guide to Shapeless: https://underscore.io/books/shapeless-guide/
