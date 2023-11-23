# An Introduction

JAMN (is) Asset Meta Notation

This repository houses the specification of the simple Jamn notation format.  This format is designed for
describing assets (typically for games) and serializing objects, but potentially has other uses.

Read the [spec](spec/jamn-spec.pdf)

See some [examples](examples/)


# An Example
```
# Comments are available
{
	name : "Asset1"
	"model_file" : "Model1.glb" 
} 
{
	"name" : "Asset2"
	model_file : "Model2.glb"
}
```

# The Notation

There are a small set of basic types:

Array
Object/Dict
Number
String 
Boolean
Encoded

These are further specified by optional type designators. 

## Array

An array is an ordered list of values.  An array is introduced by the single square bracket character [ 
Following this are 0 or more values, each value is terminated with a semi-colon character ;
Finally a closing square bracket ] finishes the array.

## Object (Dict)

Objects and dictionaries are interchangable and the distinction is made by the application.  An
object is introduced by a single brace character { followed by 0 or more fields, each field is terminated
with a semi-colon character ;.  Finally a closing } terminates the object.

A field is a string, followed by a colon character : followed by a value.

(Fields may be naked (without the "" characters of a string), with a restricted character set.)

## Number

A number is an (optional) `-` character, followed by a set of digits with optional decimal point and exponential.
They MUST start with a digit (or minus) and no whitespace may occur within a number.

Numbers have a minimum range of 2^64 (int64/uint64), or float64. However implementations MAY support larger numbers.
Larger numbers MUST be prefixed with a type designator (and may be rejected by implementations), it is an error
for numbers larger than the minimum supported range to NOT be prefixed.

Special values %nan and %inf, %negnan, %neginf introduce the (floating point) Not-a-number and infinity.  

Hex (0x...), Octal (0o...), Binary (0b...) numbers are available. The prefix MUST be lower case and
appear immediately after the opening 0. Prefixed numbers may NOT have a -ve prefix. 

Underscores may occur in a number anywhere after the first digit and are ignored.

Numbers MUST be terminated by whitespace or a semicolon, no other character may occur after a number.

## String

A basic string is surrounded by "" quotes.  Within a basic string there are various escape sequences..

A multiline string is surrounded by `` back quotes.  If the opening quote is immediately followed by a newline
then that newline is omitted, all other newlines are included in the string.

Strings have a maximum length of 128 MB.  Strings MAY be larger if they are prefixed by a type designator,
however implementations must reject documents with larger strings if they are unsupported.

### Ident String 

Ident (or Naked) strings are an ASCII subset of Unicode, they must start with an alpha character, underscore or . and may continue with any combination of alpha-numerics, underscores or . / \ .

Ident strings have a maximum length of 256 characters (necessarily 256 BYTES as ASCII subset).

## Boolean

Boolean values are specified by the special values %true and %false.

## Null

The null value is specified by the special value %null.

## Encoded

An encoded object consists of character '=' followed immediately by a string (possibly naked) specifying the encoding, followed by a single space character, followed by the encoded data.

Whitespace *should not* appear after the initial space following encoding string, however if the encoding allows ignoring whitespace then it is allowed (including new-lines).

```
$bigstring ="base64" TWFueSBoYW5kcyBtYWtlIGxpZ2h0IHdvcmsu
```

The exact encoding depends on the encoding method and the encoding string may include length or delimiters etc. as 
colon seperated fields. Due to the nature of encodings an implementation may reject a document as invalid if it does 
not support the encoding, however implementations *may* make a best-effort attempt to look for an end-of-value 
semicolon in order to skip the encoded value.

The type designator is optional.

Since a JAMN document must be UTF-8 it isn't possible to include binary data directly.

# Semi-colons

In the example above no semi-colons are present.  This is due to the automatic insertion rules.  A semi-colon
is inserted into the stream at any newline after any ordinary token (number, string, special value) or ] or }.
Semi-colons can still be used to present values on the same line.  Note that the semicolon is a *terminator*
and comes after all values (arrays and objects included).  A semi-colon is also introduced at EOF/end of stream.

Extraneous semi-colons are an error (they don't introduce a null value, only the %null can do that). However the insertion rules allow any amount of newlines and only one semi-colon will be added.

# PseudoTypes (ptypes)

All values may be preceeded with a type designator.  A type designator is introduced with a $ immediately followed by a string (NO whitespace is allowed following the $). The interpretation of type designators is implementation specific, however a set of types are recommended:

f32, f64, i8, i16, i32, i64, ref, any.

These are called PseudoTypes (abbreviated to ptype).  The 'any' type is the default type of objects within Jamn.

## Array Types

An array type is specified using an element type, a trailing underscore and an optional element count.
The element type specifier may include underscores, it is only the final underscore that counts.
This rule allows specifying nested arrays.

For example:

$f32_

specifies an array of f32.

$i8_256

specifies an array of 256 i8 (byte) values.

$any_4

specifies an array of 4 jamn values.

An implementation is allowed to ignore the specifiers and allow the following value to *not* be an array, but
implementations *are* allowed to call such a document invalid.  It is also allowable for the actual elements and length
of an array to not match the ptype, however again this may be an invalid document for a given implementation.

# Alternate Values

For some purposes there is a need for alternate representations (e.g. for floating point) which are
only approximately representable in text using default encoding.  For this purpose the alternate value | may be used.
Any value may be followed by one (or more) alternate values.  An alternate value MUST include a type designator which
is used to interpret the value, however the type of the overall value will be the same as the primary given value.


Jamn documents MUST be encoded as UTF-8, and may include a byte-order prefix.

# Comments

A comment begins with a # and continues to the end of a line.  # characters within strings are NOT considered
as introducing comments. (TODO consider naked strings)

# Top Level Values (Documents)

Any value is allowed at top-level and DOES NOT NEED surrounding braces or brackets.  However this introduces an
ambiguity which is resolved simply - the first value seen determines whether the top-level is an
object or an array.

This means that a simple string:
```
"a string"
```
Is a valid jamn document - (EOF introduces semicolon). 
```
"first value"
"second value"
"third value"
```
Is also valid and denotes an array with three entries.
```
"field1" : "value1"
"field2" : "value2"
"field3" : {
	"inner1" : "value3"
}
```
Is valid and denotes an object with three fields (one of which is an embedded object).

"field1" : "value1"
"value2"

Is NOT valid as it has both a field and value.
```
{
	field1:"value"
}
{
   field1:"value" 
}
```
Is valid and is an array of objects.

# References/Pointers

Jamn supports the notion of reference or pointer.  These are introduced by type designator $ref
on a string value.  The string defines the path from the root of the document to a particular value.
```
{
	"field1" : [
	  { 
	  	"field2" : "value"
	  }
	  ]
}
{ 
  "ref1" : $ref "/0/field1/field2"
}
```
In this example the ref1 field has a reference to "value" in the first object. Note the first
0 in the reference as this document is implicitly an array.

# More Examples

## Example 1
```
doe : "a deer, a female deer"
 ray: "a drop of golden sun"
 pi: 3.14159
 xmas: %true
 "french-hens": 3
 "calling-birds": [
   "huey"
   "dewey"
   "louie"
   "fred" 
]
 "xmas-fifth-day": {
   "calling-birds": "four"
   "french-hens": 3
   "golden-rings": 5
   partridges: {
     count: 1
     location: "a pear tree"
   }  
   "turtle-doves": "two"
}   
```

## Example 2

```
"float1" : 1.2 | $f32 0x44421110 # FIXME: work out the actual value..
"int1" : 34 | $i8 0x34  # FIXME: work out the actual value..
```
These two fields have alternate values, one specifies a f32 as a bit pattern (expressed as Hexadecimal)
The other specifies a byte (i8) as a hex bit pattern.
Note that these aren't particularly convenient for humans but are not needed unless exact bit-level
values are required.


## Example 3
```
$material {
	"name" : "Material1"
	"program" : "pbr_program1"
}
$material {
	"name" : "Material2"
	"program" : "pbr_program1"
}
$program {
	"name" : "pbr_shader1"
	"frag" : "pbr_frag.glsl"
	"vertex" : "pbr_vertex.glsl"
}
```
This example consists of an array of 3 objects.  Each object has a type designator
which can be used by the application to construct implementation dependent objects
for each of the objects in the stream.  Note that the type designators are optional
so if an implementation doesn't support or recognize one they can be ignored.

## Example 4
```
$i8_ [ 1; 2; 3; 4; 5; ]
```
This example represents an array of numbers.  A type designator is given which will (in supported 
applications) construct a byte-sized array.  There is no semantic error checking so the following is
valid:
```
$i8_ [ 1; "two"; 3; 4; 5; ]
```
However clearly in a given situation this may be flagged as an error by the parser or implementation.
```
$i8_4 [0xef;0xbe;0xad;0xde;]
```
This is an array of 4 bytes.

Alternates:
```
$i8_ [ 1 2 3 4 5 ]

$i8_ [ 1 "two" 3 4 5 ]
```

## Example 5
```
$circle { c: [-1 2]; r: 50; }

$path { d: [M 7 7 L 2 3 M 2 6 L 1 5 Z] }
```