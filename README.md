# avrogen

> [!IMPORTANT]
> This repository has moved to GitLab: [gitlab.com/dmlambea/avrogen](https://gitlab.com/dmlambea/avrogen).
> This GitHub copy is a read-only mirror and may be out of date. 

Avro code generator for Go.

**avrogen** is designed to generate easy to read, type-safe Go source code from Avro schemas in JSON format. Since developers spend most of their time reading code, **avrogen** generates clean, beautiful code with usage examples in block comments.

**avrogen** targets the [Avro 1.11.1 specification](https://avro.apache.org/docs/1.11.1/specification/), with full support for logical types, schema evolution, parsing canonical form, fingerprints and single-object encoding.

**avrogen** is inspired by [gogen-avro](https://github.com/actgardner/gogen-avro), a feature-rich code generator for Go. Its repo has been on a (possibly permanent) break since 2022, but it's still a great alternative if you'd rather lean on a more battle-tested project.

> **In a hurry?** Skip ahead to [Working with generated code](#working-with-generated-code) to see what the generated Go looks like.

## Install instructions

**avrogen** can be installed in the usual way:

```sh
go install gitlab.com/dmlambea/avrogen/cmd/avrogen@<tag>
```

This will install the command-line tool from branch/tag labelled `<tag>`, which can be used to generate Go source files from Avro JSON schema files. For example, `go install gitlab.com/dmlambea/avrogen/cmd/avrogen@master` will install the latest changes from master development branch. 


## Usage

### Command-line options

Command line help can be obtained by executing `avrogen` without arguments:

```bash
$ ./avrogen
NAME:
   avrogen - Generates Go source code from Avro schemas in JSON format

USAGE:
   avrogen [global options] [output directory] <file> [file ...]

VERSION:
   (version num.)

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --package NAME, -p NAME       Use NAME as the package name for the generated code. If not set, the name of the package will be deduced from the target directory's name.
   --namespaces STYLE, -n STYLE  Identifiers will be generated using the STYLE naming strategy for namespaced elements.  Accepted values are none, short and full (default: "none")
   --help, -h                    show help
   --version, -v                 print the version
```

### Generating code from Avro Schemas

Given one or more Avro schema files, the corresponding Go source code can be generated:

1. By directly invoking `avrogen`:

   ```shell
   <path-to-avrogen-install-dir-or-GOPATH/bin>/avrogen avro/ my-schema.avsc
   ```

   This will generate the Go source files into directory `avro` and their package name will match the directory name.

2. By using a `go:generate` directive inside a Go source file:

   ```go
   package example
   
   //go:generate mkdir -p avro
   //go:generate $GOPATH/bin/avrogen avro my-schema.avsc
   ```



## Naming

**avrogen** supports the following three namespace naming styles:

| Style | Description                                                  |
| ----- | ------------------------------------------------------------ |
| none  | Namespaces are not being taken into account when generating the Go source code. This makes the code smaller and easier to read, but a naming conflict will arise if the schemas contain two or more types with the same non-qualified names. |
| short | The last component of the namespace will be used as prefix for every generated type name. This allows the code to remain as short as possible while being able to solve basic name conflicts. |
| full  | All names will be prefixed with the namespaces they belong to in order to remove conflicts. This style is really verbose if the namespaces are long. |



## Working with generated code

### Records

Generated code for record types is quite straightforward. Apart from the fields, it also includes four public (exported) methods. For example, being _Xxx_ the public, Go-style name for a generated record, the following methods will be available:

| Method                       | Purpose                                                      |
| ---------------------------- | ------------------------------------------------------------ |
| DeserializeXxx               | Accepts an `io.Reader` and deserializes a record of type Xxx from it. The writer's schema used to deserialize the record is the same as the one used to generate the source code for Xxx. This is the method you will want to use if your schemas are immutable. |
| DeserializeXxxFromJSONSchema | Almost identical to the `DeserializeXxx` method, but also accepting the writer's schema in JSON format. This is probably the method you'll want to use if you are using Avro's schema evolution and/or Confluent's Schema Registry. |
| Serialize                    | Accepts an `io.Writer` and serializes the record's contents into it. |
| JSONSchema                   | Returns the JSON schema this struct was generated from. Handy when you need to publish it to a schema registry, compute its parsing canonical form, or take a fingerprint of it. |

The following example Avro schema:

```json
{
	"type": "record",
	"name": "person",
	"fields": [{
		"name": "name",
		"type": "string"
	}, {
		"name": "age",
		"type": "int"
	}, {
		"name": "vip",
		"type": "boolean"
	}]
}
```

Generates this code (excerpt):

```go
...

// Person is the struct for holding values of
// the Avro type Person.
type Person struct {
	Name string
	Age int32
	Vip bool
}

// DeserializePerson consumes as much data from r as needed to
// deserialize an instance of type Person and returns it, or an
// error.
func DeserializePerson(r io.Reader) (t Person, err error) {
	return DeserializePersonFromJSONSchema(r, t.JSONSchema())
}

// DeserializePersonFromJSONSchema deserializes the data from r, which was
// written by a writer using the given schema in JSON format.
func DeserializePersonFromJSONSchema(r io.Reader, schema string) (t Person, err error) {
	if err = avro.DeserializeFromJSONSchemas(r, schema, t.JSONSchema(), &t); err != nil {
		err = fmt.Errorf("error deserializing Person: %v", err)
	}
	return
}

// Serialize writes this struct's current data to writer w in Avro binary encoding.
func (r Person) Serialize(w io.Writer) error {
	return writePerson(r, w)
}

// JSONSchema returns the JSON schema used to generate this struct.
func (r Person) JSONSchema() string {
	return `{"fields":[{"name":"name","type":"string"},{"name":"age","type":"int"},{"name":"vip","type":"boolean"}],"name":"person","type":"record"}`
}

...
```

### Enums

Enum types are generated as `int32` values. A concrete type and constant values for each symbol are generated for each enum, to promote type-safe code. Additionally, a method `XxxFromString` is generated for each enum (named _Xxx_ in the example), as a convenience method for converting text symbols to type-safe enum constants.

The following example schema:

```json
{
	"type": "enum",
	"name": "status",
	"symbols": ["unknown", "failing", "good"],
	"default": "unknown"
}
```

Generates the following code (excerpt):

```go
// Status is the type defining type-safe numeric values for the
// list of symbols of the enum type Status.
// Enum values can be assigned by using their constant names, e.g.:
//     myEnum := Unknown
// Symbol strings can be casted to enum types by using:
//     myEnum, err := StatusFromString("unknown")
type Status int32

const (
	// Unknown is the constant value for this enum's symbol "unknown"
	Unknown Status = 0

	// Failing is the constant value for this enum's symbol "failing"
	Failing Status = 1

	// Good is the constant value for this enum's symbol "good"
	Good Status = 2
)

// StatusFromString returns the enum constant value for the given
// symbol, or an error if the symbol is not valid.
func StatusFromString(symbol string) (enum Status, err error) {
	switch symbol {
	case "unknown":
		enum = Unknown
	case "failing":
		enum = Failing
	case "good":
		enum = Good
	default:
		err = fmt.Errorf("invalid symbol '%s' for enum Status", symbol)
	}
	return
}

// String returns the symbol string value for this enum constant.
func (e Status) String() string {
	switch e {
	case Unknown:
		return "unknown"
	case Failing:
		return "failing"
	case Good:
		return "good"
	}
	panic(fmt.Sprintf("internal error: unexpected constant value %d for enum Status", e))
}
```

### Fixed, Maps and Arrays

These types are generated as plain Go types:

| Avro type                           | Generated Go type              |
| ----------------------------------- | ------------------------------ |
| Array of string                     | `type ArrayString []string`    |
| Map of int                          | `type MapInt map[string]int32` |
| A fixed type of size 6, named "mac" | `type Mac [6]byte`             |

For maps, two convenience methods are also generated:

- A map constructor, so maps within records can easily be constructed in Go-ish way.
- A convenience `Add` method which returns the same map, so it can be chained for easy map initialization.

Example: the Avro schema `{"type": "map", "values": "string"}` generates the following code (excerpt):

```go
...

// MapString is the type defining an Avro map of string
type MapString map[string]string

// NewMapString is a constructor function to create a map of type map[string]string
// This call can be chained with Add() to easily create & initialize a map.
func NewMapString() MapString {
	return make(MapString)
}

func (m MapString) Add(key string, val string) MapString {
	m[key] = val
	return m
}

...
```

And it can be used in struct initializers as follows:

```
myStruct := sampleRecord{
	accounts: NewMapString().Add("admin", "Admin user"),
}
```

### Unions

Avro union types are generated in three "flavours", depending on their schema types:

- Simple optional unions: those having a null type and another non-null type, like  `[ "null", "int" ]`. This type of unions are called "simple optional" because they can be seen as an optional field. In this case, an optional int:  there is a value or there is none.
  The best idiomatic Go structure for such unions are a pointer to their Go type (`*int32` in this example).
- Complex unions:  those having two or more schema types, none of them being a null type, like `[ "int", "string" ]` or `[ "string", "float", {"type": "array", "items": "int"} ]`. The complexity of this type of unions is that the only Go type able to hold variable-type values is the empty interface (`any`), which is overly tolerant for generating real type-safe Go code.
  Complex unions are generated as special `struct` types with an initializer constructor and type-safe setter and getter methods. You can read more on this topic later.
- Complex optional unions: a mix of the above two, that is, unions having three or more schema types, one of those is a null type.
  Complex optional unions are generated as pointers to complex union structs, as described above.

#### Simple optional unions

Any union having only two schemas when one of them is of `null` type are generated as optional fields:

```json
{
	"type": "record",
	"name": "user",
	"fields": [{
		"name": "name",
		"type": "string"
	}, {
		"name": "age",
		"type": ["null", "int"]
	}]
}
```

The above record schema generates the following Go code (excerpt):

```go
...

// User is the struct for holding values of
// the Avro type User.
type User struct {
	Name string
	Age *int32
}

...
```

Please note that the optional field "age" in the Avro schema is properly generated as optional `int32` field in Go.

Working with this type of unions is quite straightforward: just set the field to nil to make it to serialize as the null type.

#### Complex unions

Complex unions are generated as special structs with no public (exported) fields, but with convenience methods for handling their allowed types. For example, given a union type with a long and a float schemas, the generated union struct is `UnionLongFloat`, with the following relevant methods:

| Method                             | Purpose                                                      |
| ---------------------------------- | ------------------------------------------------------------ |
| NewUnionLongFloat(val any)         | Constructs a new union record with an initial value of `val`, which must be of type `int64` or `float32`. |
| Set(val any)                       | Sets the current value of the union to `val`, which must be of type `int64` or `float32`. The `Set` method returns the union itself to create chainable, expressive code. |
| IsLong / IsFloat                   | Boolean methods to check if the union holds a long or a float value. |
| AsLong / AsFloat                   | Typed methods that return the union's value as the expected type. |

The following example Avro schema of a hypothetical debt record, which can be expressed as a long value (i.e., in cents) or as a percentage:

```json
{
	"type": "record",
	"name": "debtInfo",
	"fields": [{
		"name": "reference",
		"type": "string"
	}, {
		"name": "amount",
		"type": ["long", "float"]
	}]
}
```

Will generate the following code (apart from the record itself, that is omitted for brevity):

```go
...

// UnionLongFloat is a convenience type to hold any of the following supported
// types:
//   - int64
//   - float32
// Values are set by calling the Set() method.
// For checking and returning the current union's value type, the following
// methods have been generated:
//   IsLong() / AsLong()
//   IsFloat() / AsFloat()
type UnionLongFloat struct {
	index int
	value any
}

// NewUnionLongFloat creates a new union of type UnionLongFloat
// holding an initial value of 'value'.
func NewUnionLongFloat(value any) UnionLongFloat {
	u := UnionLongFloat{}
	return u.Set(value)
}

// Set makes this union to have the given value, which must be of one of its
// supported types. This method returns the same receiver it was called on, to
// make it chainable and create expressive code.
func (u *UnionLongFloat) Set(value any) UnionLongFloat {
	switch t := value.(type) {
	case int64:
		u.index = 0
	case float32:
		u.index = 1
	default:
		panic(fmt.Sprintf("invalid union value of type %T for UnionLongFloat", t))
	}
	u.value = value
	return *u
}

// IsLong return true if this union is currently holding a
// value of type int64
func (u *UnionLongFloat) IsLong() bool {
	return (u.index == 0)
}

// AsLong is a convenience function that return this union's
// current value, casted to type int64.
func (u *UnionLongFloat) AsLong() int64 {
	return u.value.(int64)
}

// IsFloat return true if this union is currently holding a
// value of type float32
func (u *UnionLongFloat) IsFloat() bool {
	return (u.index == 1)
}

// AsFloat is a convenience function that return this union's
// current value, casted to type float32.
func (u *UnionLongFloat) AsFloat() float32 {
	return u.value.(float32)
}

...
```

Usage example:

```go
...
// Create a sample record with a $5 debt
myDebt := DebtInfo{
	...
	Amount: NewUnionLongFloat(int64(500)),
	...
}

...

switch {
case myDebt.Amount.IsLong():
	fmt.Printf("Total debt: %d\n", myDebt.Amount.AsLong())
case myDebt.Amount.IsFloat():
	fmt.Printf("Total debt percent: %f\n", myDebt.Amount.AsFloat())
}

...
```

#### Complex optional unions

Unions with three or more schemas, one of which is of type `null`, are generated as a combination of the other two types. The union struct will have methods for all non-null types, but the union field itself will be generated as a pointer. Using the above example, the following record schema:

```json
{
	"type": "record",
	"name": "debtInfo",
	"fields": [{
		"name": "reference",
		"type": "string"
	}, {
		"name": "amount",
		"type": ["null", "long", "float"]
	}]
}
```

Will generate the record:

```go
...

// DebtInfo is the struct for holding values of
// the Avro type DebtInfo.
type DebtInfo struct {
	Reference string
	Amount *UnionNullLongFloat
}

...
```

And the union type just like the previous example:

```go
...

// UnionNullLongFloat is a convenience type to hold any of the following supported
// types:
//   - int64
//   - float32
// Values are set by calling the Set() method.
// For checking and returning the current union's value type, the following
// methods have been generated:
//   IsLong() / AsLong()
//   IsFloat() / AsFloat()
type UnionNullLongFloat struct {
	index int
	value any
}

...
```

Usage example:

```
..

// Create a sample record with no debt (nil Amount field)
myDebt := DebtInfo{
	Amount: nil,  // or simply remove this line
}

// Make a 7.5% debt
myDebt.Amount = NewUnionLongFloat(float32(7.5))

switch {
case myDebt.Amount == nil:
	fmt.Println("Good news: no debt!")
case myDebt.Amount.IsLong():
	fmt.Printf("Total debt: %d\n", myDebt.Amount.AsLong())
case myDebt.Amount.IsFloat():
	fmt.Printf("Debt percent: %f\n", myDebt.Amount.AsFloat())
}
```

### Logical types

Avro logical types are decoded into idiomatic Go types instead of their underlying primitives. So you don't have to write conversion glue every time you read a record:

| Avro logical type        | Generated Go type                                |
| ------------------------ | ------------------------------------------------ |
| `date`                   | `time.Time`                                      |
| `time-millis`            | `time.Duration`                                  |
| `time-micros`            | `time.Duration`                                  |
| `timestamp-millis`       | `time.Time` (UTC)                                |
| `timestamp-micros`       | `time.Time` (UTC)                                |
| `local-timestamp-millis` | `time.Time` (local)                              |
| `local-timestamp-micros` | `time.Time` (local)                              |
| `decimal`                | `*big.Rat` (precision/scale carried by the schema) |
| `duration` (fixed[12])   | `logical.Duration` struct (`Months`, `Days`, `Millis`) |

For example, the schema:

```json
{
    "type": "record",
    "name": "Event",
    "fields": [
        {"name": "Id", "type": "long"},
        {"name": "OccurredAt", "type": {"type": "long", "logicalType": "timestamp-millis"}},
        {"name": "Day", "type": {"type": "int", "logicalType": "date"}}
    ]
}
```

Generates:

```go
type Event struct {
	Id         int64
	OccurredAt time.Time
	Day        time.Time
}
```

Logical types also work inside simple optional unions (e.g. `["null", {"type": "long", "logicalType": "timestamp-millis"}]`), which are generated as `*time.Time` and serialise as null when nil.

If a logical type is unknown to **avrogen**, the field falls back to its underlying primitive (per the Avro spec), so unknown annotations don't break your build.

## Schema registry interop

For Confluent/Apicurio-style registries, **avrogen** ships with the bits you typically need:

- **Parsing canonical form** — `pkg/types/canonical.go` produces a deterministic JSON form so semantically equivalent schemas hash to the same value.
- **Fingerprints** — CRC-64-AVRO, MD5 and SHA-256 over the canonical form (`pkg/types/fingerprint.go`).
- **Single-object encoding** — `pkg/avro/single_object.go` adds and strips the `C3 01` marker plus the 8-byte little-endian fingerprint expected by the spec.

These are plain helpers; combine them with the `JSONSchema()` method on each generated type to publish, look up, or wrap a payload before sending it on the wire.

## License

**avrogen** is released under the [EUPL v1.2](LICENSE.txt). And before you close the tab in panic: the EUPL is **not** a viral licence. You can:

- read, modify and use this code for any purpose, with or without changes,
- ship your fork under EUPL or any of the compatible licences listed in the article 5 clause (GPL, AGPL, LGPL, OSL, EPL, CeCILL, MPL, CC BY-SA, LiLiQ),
- bundle this code alongside proprietary code in your own products.

The [LICENSE.txt](LICENSE.txt) file leads with a plain-language clarification on top of the legal text, so you don't have to read 60 articles to figure out what you're allowed to do. TL;DR: keep the copyright notice and we're good.

