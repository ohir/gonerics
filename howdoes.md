## How does it work underhood.

Example code:

```
1   type MyFloat    float64
2   type MyIntShort int8
3   type MyIntOne   int8
4   type MyIntTwo   int32
5   
6   func Half(x type T) (r type T) {
7     for type T = int64() // contract: Half for int based types
8     r = x / 2
9     return
10  }
11  
12  var ishr MyIntShort = 10
13  var isxx MyIntShort = 20
14  var ione MyIntOne = 30
15  var itwo MyIntTwo = 40
16  var iflo MyFloat = 7
17  
18  shrhalved := Half(ishr)
19  sxxhalved := Half(isxx)
20  onehalved := Half(ione)
21  twohalved := Half(itwo)
22  flohalved := Half(iflo)
```

> Note, that below is given most naïve implementation sketch, meant as
> an aid to show CGG feasibility. I also personify the compiler by conflating
> its phases into short statements of "compiler sees this, compiler does this".

When compiler arrives at goneric func declaration (line 6), it sees the
"type" keyword in the signature and marks the node as "goneric func".
The symbol table entry at "Half" key will now has a pointer to this node.


### 1. a call with a set of types not previously seen

Compiler sees "Half" is called with parameter "ishr" at line 18,

It first looks up the "Half" identificator in the symbol table, and gets
informed that Half is a name of goneric function. So it now mangles the
call site using parameter type names:

`Half(MyIntShort)(MyIntShort) => "HalfΣMyIntShortΞMyIntShort` // sigma theta used

then compiler looks up symbol table for mangled identifier. There is none,
so compiler tries to instantiate a variant of goneric Half:

It takes the declaration subtree (from the "Half" node) and tries to match
`MyIntShort` with the contract. Then compiler reads contract at line 7.
Contract says that input type T must be precisely convertible to an `int64`.

Type `MyIntShort` is, by having base type of `int8`, so the 'Matched'
flag is set on `x` (and then `r`, as r is T too) parameters.

Then compiler mangles call site using base types names it found: it gets to
`HalfϞint8ϟint8` identifier and looks up symbol table for an entry of this
exact code variant. There is none, so compilers tries to make one.

It goes past the contract and instantiates a temporary variant substituting
`int8` for typeholder T:

```go
func HalfϞint8ϟint8(x int8) (r int8) { // greek koppa used
    r = x / 2
		return
}
```

New code variant gets compiled. If it errs, compilation stops with error
mesage composed in terms of goneric declaration of Half. If it does compile
successfully, compiler moves just checked variant subtree to the main ast.

Then symbol table is updated with the variant name "HalfϞint8ϟint8" pointing
to the instantiated code variant and with a call alias using call-site types
mangled in:

`HalfΣMyIntShortΞMyIntShort -> HalfϞint8ϟint8 -> code_variant`

Then the call in place is transformed to the

  `HalfϞint8ϟint8(ishr)`


### 2. subsequent call with same set of types

Compiler sees "Half" is called with parameter "isxx" at line 19,

It first looks up the "Half" identificator in the symbol table, and gets
informed that Half is a name of goneric function. So it now mangles the
call site using parameter type identifiers and arrives at call alias
`HalfΣMyIntShortΞMyIntShort` that already points to the `HalfϞint8ϟint8`
variant.

So call in place is directly transformed to the

  `HalfϞint8ϟint8(isxx)`


### 3. subsequent call with set of types of same base type

Compiler sees "Half" is called with parameter "ione" at line 20,

It first looks up the "Half" identificator in the symbol table, and gets
informed that Half is a name of goneric function. So it now mangles the
call site using parameter type names:

`Half(MyIntOne)(MyIntOne) => HalfΣMyIntOneΞMyIntOne`

then compiler looks up symbol table for just mangled identifier.
There is none, so compiler tries to instantiate a variant of goneric Half:

It takes the declaration subtree (from the "Half" node) and tries to match
MyIntOne with the contract. It reads contract at line 7. Contract says
that input type T must be precisely convertible to an `int64`. Type
MyIntOne is, by having base type of `int8`.

Then compiler mangles call site using just found base types: `HalfϞint8ϟint8`
and looks up symbol table if a code variant is already avaliable.

It is known, so compiler just registers new call alias entry

`"HalfΣMyIntOneΞMyIntOne" -> "HalfϞint8ϟint8"`

then transforms call site to

  `HalfϞint8ϟint8(ione)`


### 4. a call with other set of types

This does not differ from [1,3]. New set of types instaniates a either a new
code alias to an existing variant or makes a new code variant.

Compiler sees "Half" is called with parameter "itwo" at line 21. Then in short:

```go
// ... Half(MyIntTwo)(MyIntTwo) => "HalfΣMyIntTwoΞMyIntTwo"
// ... HalfΣMyIntTwoΞMyIntTwo -> HalfϞint32ϟint32 ->

    func HalfϞint32ϟint32(x int32) (r int32) { 
        r = x / 2
    		return
    }
```

And the call is transformed to the

  `HalfϞint32ϟint32(itwo)`


### 5. parameter type does not match

Compiler sees "Half" is called with parameter "iflo" at line 22,

It first looks up the "Half" identificator in the symbol table, and gets
informed that Half is a name of goneric function. So it now mangles the
call site using parameter type names:

`Half(MyFloat)(MyFloat) => "HalfΣMyFloatΞMyFloat` // sigma theta

Then compiler looks up symbol table for mangled identifier. There is none,
so compiler tries to instantiate a variant of goneric Half:

It takes the declaration subtree (from the "Half" node) and tries to match
MyFloat with the contract. Then compiler reads contract at line 7.
Contract says that input type T must be precisely convertible to an `int64`.
Neither the type `MyFloat` itself nor its base type `float64` is,
so contract does not match. 

At line 8, code is about to use `r`,`x` of `notMatched` type.

This results in compile error in terms of the call site:

"**cannot use iflo (type MyFloat) in argument to goneric Half func.
Contract does not allow MyFloat in place of a type holder T**".

## TODO, write about:

1. `For type switch` compile-time mechanics
1. Contract matching
1. Variant symbol mangling techniques for methodsets
   and type shapes to cheaply get to the significant
 	reduction of object code variants.
1. How CGG generic Sum example will produce only a few
   code variants for plethora of user types involved.
1. Foreseen SSA level optimizations.

If only I can find time for it.

