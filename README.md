## "Craftsman's Go Generics" proposal. CGG in short.
### Rationale

I [criticised](https://groups.google.com/forum/#!topic/golang-nuts/_L9G0968HBk) Go team's proposal
so I felt compelled to give a consistent counter-proposal.

### Current state of CGG (TL;DR)

* no new keywords. Only ones known to Go1.
- contract uses already known Go type definitions and casts.
- only one new syntax construct for user, and three for generic code
  1. `x.pkg.Method()` generic method call via package selector
  1. `(x type T)` declarations — as in team's proposal
  1. `for type` contract; tied to the package or func
  1. `for type switch` that can further specialise code on matched type 

### Generic code sample

```go
// Sum method 
func (x type []K) Sum() (r type K) {
  for type ( 
      K range int64(), uint64(), float64(), complex128()
  ) 
  for _, v := range x {
    r += v
  }
  return
}

// Min func returns lesser value T
func Min(a, b T) T {
  for type switch {
  case T func (T) Less(T) T: // specialise for types
      return a.Less(b)       // that have Less method
  }
  if a < b { // all other cases
       return a
  }
  return b
}
```

Current Go1 code that now uses `func Min(a, b int)` can use future generic 
`func Min(a, b T) T` without touching a comma. Call site does not change.

## Generic code
### Terms 

Given generic `func (a type T) Gen(b type U, c uint32) (r type R, e Error)`

- **T** and **U** are **"In"** typeholders; reffered simply as "in holders" or just "holders".
- **R** is a **"return"** (or **"Out"**) typeholder; reffered simply as "return holder"
- **"Substituted type**" means real type the code uses in given place after code variant instantiation.
 Often reffered as either "in type" or "out type". 
- **"in type"** means "substituted type" given to the **"In"** typeholder at call site.
- **"out type"** means "substituted type" of the return parameter given to the **"Out"** typeholder
within generic function. Also: type returned to the call site.
- **"Solid type**" means a non-interface type.
- **"external type"** means a non-generic type defined elsewhere. Parameters 'c' and 'e' are of defined externally types.

### Generic Interfaces

- Generic method MAY NOT be a part of interface specification.
- NO generic method is a member of any methodset.
- Defined on concrete type method MAY USE (wrap) generic code.

### Generic return types

There are general limitations on generic return types:

1. Substituted type of a return typeholder must be defined, unambiguous and solid before use. Naked return is a use, too.
1. **Either** return typeholder MUST BE the same as one of input typeholders
1. **Or** it MUST BE defined in the body of generic function before use.

Eg. `var R int`; `var R struct{ c, d int }`; `var R struct{ twas T; uwas U; from io.Reader }`;

### Generic input types

Given a generic code in some `func` there are properties of the substitution type(s) that are
**required** for the code to work as intended by it's creator. What is
required is written into the contract. What is not required does not matter
hence it has no place in the contract.

Contracts are tied to the package and/or function where are used [Readability! Clarity!].
Reader must not be forced to jump over several files to get a grasp over the code.

### The '**for type**' contract

CGG uses **`for type`** clause as the contract block declaration. The **`for type`**
contract consists of required by the author of the code constraints on **ALL** of **"in"** typeholders.

Contract constraints can be defined at two scope levels:
   - package - right after the imports;
   - function - at the beginning of the func body;

All constraints in `for type` contract must be fulfilled by the substituted type for it to pass.
As if all were conjugated by logical AND. Constraints defined for a given typeholder (its identifier)
at package level MAY NOT be redefined (amended, narrowed, extended) at `func` level. [Clarity! Readability!].

The `for type` either is repeated in every line or uses `for type ( ... )` construct.

```go
package "ac"

import "something"

for type (                 // package level contract
    T constraint
)

//...

func Gen(a, b type T, c type U) (r type R) {
    for type U constraint  // func level contract
                           // T is constrained at package level
    var R bool             // generic return type MUST be defined 
                           // within body 
    // ... body
        return
}
```
.

### Constraints

Constraint may apply to the whole substituted type or to part thereof.
Constraint is given in terms of Go type, type literal or cast. Constrain
may enumerate on possible `one of` types or typecasts using `range` keyword
followed by a comma-separated list.

Full (but not exhaustive) list of constraints is at the end of
this document in Appendix A.

Constraint examples:

`for type (`

Typeholder | Constraint | Description
-----------|------------|------------
`T`|`range st1, st2, st3          `| T **is one of** given types (in set)
`T`|`= TypeX                      `| T **is of type** TypeX
`T`|`= TypeX()                    `| T **is assignable** via a cast to TypeX. 
`T`|`func (*T) Check() bool       `| T **has** method 'Check' of given signature 
`T.height`|`= TypeX()             `| T **has** field 'height' castable to type TypeX

`)`

**Important:** in the context of constraints **assignable to via a cast** means that
a substituted type can be **precisely** casted to the stated type (ie. usually a base one). Implicit conversions are forbidden.

So `T = uint64()` allows for any type that has a base type of byte, uint8..uint64 but not one of base type int.




### "for type" switch

is used for branching specialised code over one or more **in** types.

All cases of this switch are checked and resolved at **compile time**. Logical conditions on each
case consist of valid contractual constrains over types that are further conjugated with explicit
AND (&&), and only with AND. 

**Switch semantics:** case expressions made of contractual constraints are evaluated
left-to-right and top-to-bottom; the first case that checked type matches triggers use of the
statements of the associated case; the other cases are skipped. If no case matches then substituted
type **does not match** and no code from this instance of `for type switch` is used. If substituted
type does not match somewhere within all of possible places (switches) it is a compile error of
'func/method (*identifier*) can not be used with (*given*) type'. There is no `default:` and no `fallthrough`
possible within `for type switch`.

**Out types:** EACH `for type switch case` body MUST define return type(s) for all __out__
typeholders. Clarity and readability trumps repetition and possible inconveniences for the writer.
> First, a human does not need to look for it elsewhere; second, debugger is then allowed to simply hide all but one branches presenting to the user only instantiated code.

Example:

```go
func (type []K) Checkout() (total int) {
  for type switch {
  // any struct that has matching Value & Discount fields
  case ( K.Value = int() && K.Discount = int ):
    for _, v := range x {
      total += (int(v.Value)*1000)/K.Discount
    }

  // any struct that has matching Value field
  case K.Value = int(): // regular price items
    for _, v := range x {
      total += int(v.Value)
    }

  // last: any K of base type castable to int
  case K = int():
    for _, v := range x {
      total += int(v)
    }
  }
  return total
}

```

### Generic Data

> **Note:** this currently is a **rejected** sketch. I think that since one can
> instantiate any anonymous yet concrete type off generic code, it would be better to do generic
> data by convention. Ie. generic packages implementing NewDataIdentifier( x, y, z )
> where x, y, z are of requested types. But I am not sure, so sketch follows:

```go
type TypeName struct {
    for type (
        S constraint
        T constraint
        U constraint
    )
    Sfield S
    Tfield T
    Ufield U
}
```
.

Instantation of generic data **always** uses cast,
even for untyped constants.

`var x = TypeName{ TyForS(a), TyForT(b), int(-100) }`

The main reason for rejection was about user code pollution. The "New" approach
gives clean boundary and indicators.

Second was, that I consciously rejected any binding between generic code and
methodsets. Were it allowed, we would soon read tons and tons of go++++ code.
So such generic data might only be a "pure" one. From the other hand, it is
safe to call a generic method on "pure" generic data. Contracts for those would
be not minimal, but it could give some power. From the yet other leg, though, I do
not feel whether is it really safe; as of readability and clarity wise.

But it is an open question. It might have a place.

## Subtleties

### signed vs unsigned

While Go1 allows a cast from int to uint without doing abs() [this is C fail that lingers]
within `for type` constraint `int()` does NOT allow for unsigned types and vice versa.
In fact "assignable to via a cast" means only precise mapping to the base type for now.

### convertible

I rejected "convertible to" constraint because it can lead, esp novice users, to
hard to debug code. Ie. where example `T = int(<)` convertible to int might make
results fry on floats.


### but we need conversions

It is possible do conversion by convention - in line with current "has Error, has String" convention.
I.e. Huge generic method "ReadAs" would be in a package and convertible types would wrap it in their
real methods (to be seen in methodsets) then... To be pondered of :)


## Open Questions

CGG constraints are typesystem based, so all (for now) can be checked and resolved
eg. by reflect package.

> Can a team's way of specifying constraints and CGG's be combined?

Yep. It is possible.

There can be eg. `for type T { T + T }` constraint written for "addable" type.
Also `for type T { T.field int }` instead of `for type T.field = int`
Or `for type T { T < K }` to check comparability. But see Min example at the
beginning: Substituted types can be checked for comparability by current
compiler exactly in place of their use, where their types are already concrete.
No need to write artificial constraint for it.


### assignable vs castable

There is a slight difference in mental impedance between me and team member due
to strict meaning of 'Assignable to' in language's specification; He is right,
its my fault: I hoped for TypeX() cast-like constraint shape to be
selfdescriptive, but it isnt. Possible resolutions:

1. Replace "assignable to" with "castable to"
2. Be descriptive as in "assignable to via a cast"

Current revision used 2. But if 1. sounds better to you, feedback on it.

---

> Should "cast" constraints use repeated type identifier as `for type T = int(T)`?

**NO**: It would introduce a point of c&p error for almost no gain in readability.

## More code samples

```go
// Sum func
func Sum(x type []K) (total type K) {
  for type K range int64(), uint64(), float64(), complex128()
    for _, v := range x {
      total += v
    }
    return
}

// TODO next
```


## Rebuttals

**Before you raise concern on a list read here. Its possible that someone already
got to it and got an answer.**

---

> That is, the compiler is directed, at compile time, as to which code should be compiled.

That is, the code user is directed, at reading time, as to which code will
compile for given type argument. Easy to see, easy to understand.

> That is metaprogramming.

It is **not**. Every and each statement is written by the human author in plain Go
and no code is produced by the program itself.

---

> code that can not be compiled.

If an author of the `for type case` wrote the case she supposedly wrote a
test case for that branch. So a `for type case` branch will be compiled at least for tests.

---

> I agree that no code is produced by the program, but you're suggesting
> a compile-time decision as to which code should be compiled.    

How does it differ from unused code eliding go compiler and linker do
already? Used `for type case` code will compile and run. Unused will be
elided, just a bit earlier.

---

> where is a formal proof of soudness and completeness for this?

Where is one for Go1?

CGG **swims like Generics, flys like Generics, quacks like Generics.** 
And it is **READABLE**.

--- 

fun ahead: :)
> [ somenick] ohir: It needs compile-time logic. It needs golang compiler to make decissions over types and branch on that

 [ @ohir ] Ek!? Any compiler already does it for almost **every** line of code it sees.

## Appendix A 

### List of possible contractual constraints

Specification by example. Not exhausting one.


`for type (`

typeholder | Constraint | Description
-----------|------------|------------
`T`|`range st1, st2, st3          `| T **is one of** given types (in set)
`T`|`range st1(), st2(), st3()    `| T **is assignable to one of** given types via a cast 
`T`|`range st1, st2, st3()        `|   A mix of above. See "open questions" 4.1
`T`|`= io.Reader                  `| T **implements** interface
`T`|`= context.Context            `|   and other interface
`T`|`= TypeX                      `| T **is of type** TypeX (doesn't make much sense here but it does in switch)
`T`|`= TypeX()                    `| T **is assignable to** TypeX via a cast. 
`T`|`= chan G                     `| T **is a** channel (bidi) for G values
`T`|`= chan<- G                   `| T **is a** channel (send) for G values
`T`|`= <-chan G                   `| T **is a** channel (receive) for G values
`T`|`= struct Solid               `| T **has** AT LEAST ALL fields of Solid type struct (eg. embeds one)
`T.BiPipe`|`= chan G              `| T **has** field 'BiPipe' of bidi channel for G values
`T.TxPipe`|`= chan<- G            `| T **has** field 'TxPipe' of send channel for G values
`T.RxPipe`|`= <-chan G            `| T **has** field 'RxPipe' of receive channel for G values
`T.Weight`|`= TypeX()             `| T **has** field 'Weight' assignable to type TypeX via a cast
`T.Gstats`|`= struct Stats        `| T **has** field 'Gstats' that is of struct type with AT LEAST ALL fields of Stats
`T.LitStr`|`= struct{ a, b int }  `| T **has** field 'LitStr' that is struct type with AT LEAST a, b int fields
`T.vcheck`|`= func(U) bool        `| T **has** field 'vcheck' of func value (signature given)
`T.weight`|`= TypeX               `| T **has** field 'weight' OF type TypeX
`T.output`|`= io.Writer           `| T **has** field 'output' of type implementing io.Writer
`T.output`|`= []                  `|   that CAN BE ranged over, indexed and sliced (elements of any type)
`T.output`|`= []interface{}       `|   kosher version of above
`T.failed`|`= []E                 `| T **has** field 'failed' that IS a slice or array of Es
`T.dummie`|`= [64]TypeX           `| T **has** field 'dummie' of exact array type
`T.Validm`|`= map[K]V             `| T **has** field 'Validm' that is a map of Vs with K type keys
`T`|` func (T) Commit(bool) error` | T **has** method (T) Commit(bool) error; Having pointer one will pass too. 
`T`|` func(*T) Revert() error`     | T **has** method Revert with pointer receiver; (T) will not pass.
`T`|`= func(T) bool`               | T **is a** func of given signature
`T`|`= func Check(T) bool         `|   variant: allow c&p named function. Anonymous function will pass too.
`T.ckin`|` func (ckin) Commit(bool) error`|T **has** field ckin of type that has method Commit(bool) error
`T.ckin`|` func (*ckin) Commit(bool) error`|as above, must have pointer receiver

`)`


## Feedback

---

Give a star if you're positive about a craftsman's approach to Go generics.

Discuss on go-nuts, i read it. If you see a real issue with this approach - open an issue ;)

TC, Ohir

## Footnotes



>> No employer minutes were stolen for this work. Its in public domain now.


