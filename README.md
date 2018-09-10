---
title: "Gonerics"
date: 2018-09-10T12:00:46+02:00
anchor: "gonerics"
weight: 10 
---

## Rationale

At last the Go community started to discuss not whether generics are of any use but
actually about __how to implement them__. The [Go2 Generics Proposal](https://go.googlesource.com/proposal/+/master/design/go2draft-generics-overview.md) has been produced by the core-team and published. In my humble opinion, if implemented as proposed, it has potential to destroy Go. Faster than Moose.pm infection destroyed perl.
That's why [I ranted publicly](https://groups.google.com/forum/#!topic/golang-nuts/_L9G0968HBk) on the go-nuts list.

As empty criticism is bad, I felt compelled to give a consistent counter-proposal; one that can make my ["Craftsman wishes"](https://play.golang.org/p/qkslpdyvhaq) code compile and work. Here we are.

## Assumptions

- It must read as natural as Go1
- It must not introduce (much) new syntax to the language: **no instantiations at call site**
- It should not introduce new keywords
- It should not introduce too many new concepts
- It must work (for at least hypothetical 80% usecases)
- It **need not** to be easy to implement, but it must aim for **O(n)** complexity (for n types).

 *Hereafter I will use __CGG__ acronym for the long "Craftsman's Go Generics (proposal)"*.
 *I will use simple language as much as possible. Eg. "contract constarint" instead of "predicate". If my proposal will ring with the community, it will first got community provided corrections then at last proper formal specification. Or the Go devs will tinker with it, maybe.*  

## 0. What stays 

- The syntax introducing generic types of '**type TypePlaceholderIdentifier**' form.

    `func Gen(a, b type T, c type U) (r type R)`. Used only in func **definition**.

- The notion of **contract** as means of stating constraints on allowed types. Contract specifies everything what both compiler and I, either as mere reader or prospect user, need to know to sucessfully invoke func of interest.

## 1. Meet the '**for type**' contract

> *substituted type - real type the code uses in given place after code variant instantiation.*

Given generic `func Gen(a, b type T, c type U) (r type R)` there are properties of the substitution type(s) that are **required** for the function to work as intended by it's creator. What is required is written into the contract. What is not required does not matter hence it has no place in the contract. CGG uses **`for type`** clause as the contract block declaration.

Contract constraints can be declared at three scope levels:

   1. At package level - right after the imports.
   2. At function level - at the beginning of the func body.
   3. At generic branch within the func body.

As with other Go declarations `for type` either is repeated in every line
or uses `for type ( ... )` construct. It reads naturally in both forms:

```go
package "ac"

import "something"

// generic package for types of...
for type (
    T constraint
    U constraint
)
//...

func Gen(a, b type T, c type U) (r type R) {
    // generic function for types of...
    for type U constraint
    for type R constraint
    // T is constrained at package level
    // ... func body
}

```
  ```bnf
  ContractDecl = "for type" ( ContractRule | "(" { ContractRule ";" } ")" ) .
  ContractRule = TypeHolderIdentifier [ "." TypeHolderFieldIdentifier] RealTypePredicate .
  ...(I have no time now for complete BNF, excuse me)...
  ```
. . 


### 1.2. CGG Contractual Constraints

Constraint may apply to the whole substituted type or to part thereof.
Contractual constraint is given as the Go type or type literal.
All constraints in `for type` contract must be fulfilled by the substituted type for it to pass.
As if all were conjugated by logical AND.

Below is "specification by example" of possible constraints. Not exhausting one, yet.

`for type (`

typeholder | Constraint | Description
-----------|------------|------------
`T`|`range st1, st2, st3          `| T **is one of** given types (in set)
`T`|`= io.Reader                  `| T **implements** interface
`T`|`= context.Context            `|   and other interface
`T`|`= int                        `| T **is of type** int (doesn't make much sense here but it does in switch)
`T`|`= int()                      `| T **is assignable** to int
`T`|`= chan G                     `| T **is a** channel (bidi) for G values
`T`|`= chan<- G                   `| T **is a** channel (send) for G values
`T`|`= <-chan G                   `| T **is a** channel (receive) for G values
`T`|`= struct Solid               `| T **has** AT LEAST ALL fields of Solid type struct (eg. embeds one)
`T.BiPipe`|`= chan G              `| T **has** field 'BiPipe' of bidi channel for G values
`T.TxPipe`|`= chan<- G            `| T **has** field 'TxPipe' of send channel for G values
`T.RxPipe`|`= <-chan G            `| T **has** field 'RxPipe' of receive channel for G values
`T.Weight`|`= int()               `| T **has** field 'Weight' assignable to int
`T.Gstats`|`= struct Stats        `| T **has** field 'Gstats' that is of struct type with AT LEAST ALL fields of Stats
`T.LitStr`|`= struct{ a, b int }  `| T **has** field 'LitStr' that is struct type with AT LEAST a, b int fields
`T.vcheck`|`= func(U) bool        `| T **has** field 'vcheck' of func value (signature given)
`T.weight`|`= int                 `| T **has** field 'weight' OF type int
`T.output`|`= io.Writer           `| T **has** field 'output' of type implementing io.Writer
`T.output`|`= []                  `|   that CAN BE indexed and sliced (elements of any type)
`T.output`|`= []interface{}       `|   kosher version of above
`T.failed`|`= []E                 `| T **has** field 'failed' that IS a slice or array of Es
`T.dummie`|`= [64]int             `| T **has** field 'dummie' of exact array type
`T.Validm`|`= map[K]V             `| T **has** field 'Validm' that is a map of Vs with K type keys
`T`|` func (T) Commit(bool) error` | T **has** method (T) Commit(bool) error; Having pointer one will pass too. 
`T`|` func(*T) Revert() error`     | T **has** method Revert with pointer receiver; (T) will not pass.
`T`|`= func(T) bool`               | T **is a** func of given signature
`T`|`= func Check(T) bool         `|   variant: allow c&p named function. Anonymous function will pass too.
`T.ckin`|` func (ckin) Commit(bool) error`|T **has** field ckin of type that has method Commit(bool) error
`T.ckin`|` func (*ckin) Commit(bool) error`|as above, must have pointer receiver

`)`


### 1.3. Third place: `for type switch`

The `for type` contract at package or func level demands from substituted type that it pass all checks.
It might be too much restricting. Hence CGG allows for third level checks: in `for type switch`.
All cases of this switch are checked and resolved at **compile time**. Logical conditions on each
case consist of valid contractual constrains over types that are further conjugated with explicit
&&, and only with AND. Switch semantics: case expressions made of contractual constraints are evaluated
left-to-right and top-to-bottom; the first case that checked type matches triggers compilation of the
statements of the associated case; the other cases are skipped. If no case matches then substituted
type **does not match** and no code from this instance of `for type switch` is used. If substituted
type does not match somewhere it is a compile error of 'func/method (*identifier*) can not be used with
(*given*) type'. There is no `default:` and no `fallthrough`.

Example:

```go
// Method Checkout returns sum of all Ks in given []K slice.
// You can use this method on a slice of any type that is either:
// 1. assignable to int
// 2. has a field named Value that is assignable to int
// +. If K has additional field named Discount that is of type int
//    this discount will be be included in the total.

func (type []K) Checkout() (total type R) {
  for type R range int, complex128 // Eph... Unreal prices ahead
  for type switch {
  case ( K.Value = int() && K.Discount = int ): // items with "Sale" sticker
    var total int // define return type
    for _, v := range x {
      total += (int(v.Value)*1000)/K.Discount
    }
  case K.Value = int(): // regular price items
    var total int // here too
    for _, v := range x {
      total += int(v.Value)
    }
  }
  case K = int(): // services and interest items are int based
    var total int // return is generic so every case needs to define it
    for _, v := range x {
      total += int(v)
    }
  case K = complex128: // unreal prices
    var total type K   // complex128, just for example
    for _, v := range x {
      total += int(v)
    }
  }
  return total
}
```

# [Generic method I wished](https://play.golang.org/p/qkslpdyvhaq) can now be specified :)

```go
package main
import "ac" // cashier helper package

type NItem struct {
  Color FrontColor
  Weight int
  Value int
}
type SItem struct {
  NItem
  Discount int
}

func main() {
var x []int
var regular []NItem
var sale []SItem
var cmplx = []complex128{1 + 6i, -2 + 4i, -4 + 0i, -1 + -9i}
// ...
// cash customer
total := regulars.ac.Checkout() // call generic method from ac package on
total += sale.ac.Checkout()     // many types. This is the single point where
total += x.ac.Checkout()        // language semantics need to change

// And complex prices for unreal things can be sumed too
ctotal := cmplx.ac.Checkout() 
}
```

## 2. Subtleties

### 2.1 In and Out typeholders

There is a subtle thing with "Out" (return) typeholders. I opt for them being specified at func level contract
for readability. Compiler will know whether they are properly declared in `for type switch` cases, but
it might be hard for reader to find all occurences and variants. Thats why in example above is the
`for type R range int, complex128` clause. But it somewhat restricts declarations that will/might use
typeholders in return type declarations. Here I knew it will be complex128. It needs more pondering.

I do not know yet whether `for type switch` case expressions should be allowed to narrow constraints
on already (at func/package contracts) constrained "In" types. It gives power, but can allow for really convoluted and unreadable code.

## 3. More code samples

```go

```

