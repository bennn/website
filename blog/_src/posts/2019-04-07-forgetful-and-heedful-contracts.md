    Title: Forgetful and Heedful contracts
    Date: 2019-04-07T23:15:11
    Tags: migratory typing, higher-order contracts, by Ben Greenman

_Forgetful_ and _heedful_ are two methods for space-efficient contracts
 developed by Michael Greenberg circa 2014.
These methods were born in the shadow of a third method, _eidetic_,
 with stronger theoretic properties.
Since then, however, the forgetful method has been re-invented at least twice.
Both deserve a second look.
<!-- TODO cite? -->

<!-- more -->
- - -

Contracts are a tool for specifying and dynamically-enforcing the behavior
 of a program.
In a language with contracts, a programmer can annotate an API with
 code that documents the intended use for other readers.
When client code interacts with the API, the annotations ensure that the
 actual behavior matches the expected.
If there is a mismatch, the contract annotations can report an issue
 in terms of: the API code, the client code, and the contract between them.

For example, a Racket module that exports a sorting function can use a contract
 to describe the kind of input it expects.
If a client module sends invalid input, the contract **blames** the client
 module for the error **assuming** that the contract is bug-free:

```
  #lang racket/base

  (module sort racket
    (provide
      (contract-out
        [quicksort
          (-> (vectorof point/c) void?)]))

    (define point/c (vectorof integer?))

    (define (quicksort points)
      ....))

  (module client racket
    (require (submod ".." sort))
    (quicksort '()))

  (require 'client)
```

```
quicksort: contract violation;
 expected a vector
  given: '()
  in: the 1st argument of
      (-> (vectorof (vectorof integer?)) void?)
  contract from: 
      (file.rkt sort)
  blaming: (file.rkt client)
   (assuming the contract is correct)
```

That covers the basics.
For an extended introduction to contracts, visit the Racket guide.
<!-- TODO cite -->

> The quicksort example and the related figures are from the paper
> [_Collapsible Contracts: Fixing a Pathology of Gradual Typing_](TODO)


### Contracts and "Space Efficiency"

The `(vectorof point/c)` contract used above describes a possibly-mutable
 array that contains elements that match the `point/c` contract.
Since the array can be mutated, this contract has implications for two parties:

1. the client module must supply a good array, and
2. the sorting module must not insert a bad element.

To enforce the second condition, the `vectorof` contract wraps incoming
 vectors in a proxy that checks future writes.
Suppose the client sends a vector with four points:

```
(vector (vector 4 4)
        (vector 2 2)
        (vector 1 1)
        (vector 3 3))
```

After applying the contract, the vector is wrapped in a proxy that checks
 incoming writes and outgoing reads.
The following picture illustrates the wrapper with a **solid** blue bar
 for the **write** checks against the sort module and a _striped_ blue bar
 for the _read_ checks against the client.

<img src="/img/vector-chaperone-0.png" alt="A wrapped vector"/>

In a straightforward implementation, these wrappers can stack up if multiple
 contracts are applied to the same value.
For our quicksort in particular, the elements of the vector are mutable
 vectors and may accumulate wrappers as the vector is sorted ---
 because every **write** and _read_ applies a contract to the element.

<img src="/img/vector-chaperone-1.png" alt="Layers of element wrappers"/>

On the bright side, these wrappers enforce the contracts and help the
 programmer understand the source of the error if any contract is violated.

Unfortunately, the wrappers also affect the performance of the program.
There is a price to pay for:
 (1) checking values against the contracts,
 (2) allocating a new wrapper,
 (3) and "indirecting" future writes/reads through a wrapper.
This can be a problem.

> "on a randomly ordered vector of 1,000 points, a call to quicksort can
> wrap the inner vectors an average of 21 times"

To fix the problem, researchers have been exploring _space-efficient_
 implementations of contracts that attach a bounded number of wrappers to any
 value.
Michael Greenberg is one of these researchers, and _eidetic_, _forgetful_,
 and _heedful_ are his names for three new implementations.


### Eidetic space-efficiency

The eidetic method introduces a new data structure to represent higher-order
 contracts.
The structure supports a _merge_ operation;
 when two contracts meet, they are merged in a way that avoids duplication.
Eidetic contracts have the same behavior as normal "wrapping" contracts
 and their size is bounded by the number (and height) of source-code
 contracts in the program.

An eidetic contract is a binary tree:

- each node represents a higher-order contract combinator, such as `vectorof`
- the two children of a node represent the obligations of the two parties
  involved in the contract
- each leaf is a list of non-higher-order, or _flat_, contracts

<!-- For example, the `(vectorof point/c)` source-code contract turns into an -->
<!--  eidetic tree with 3 nodes and 4 singleton-list leaves (Sec. 3.1 of _Collapsible Contracts_). -->
<!-- One tree is always "bigger" than the original wrapping contract. -->
<!-- Section 3.1 of the Collapsible Contracts paper has an illustration. -->

A successful merge combines two trees of the same shape
 by re-using half the nodes
 and appending the leaf lists.
Re-using nodes saves some space, and helps reduce the overhead of trees
 relative to simple wrapping contracts.
The main savings comes from filtering the leaf lists --- if an
 implementation comes with a `contract-stronger?` predicate that tests
 whether one flat contract accepts fewer values than a second, then it
 can remove leaf-list contracts that are preceded by stronger ones.
Trees make this filtering possible.

Suffice to say, eidetic comes with practical challenges.
Are trees more expensive than wrappers in the common case?
Where does the `contract-stronger?` predicate come from?

Thankfully, there are at least two "compromise" alternatives.


### Forgetful Space-Efficiency

Mission accomplished, space is constant.
But forgetting the contracts also forgets the blame.

### Heedful Space-Efficiency

Okay we have a weaker bound but a better story for blame.

The eidetic design goes further --- with a weaker bound and a more complicated
 data structure --- and gets blame exactly right.

Read more in Michael's paper, or follow Dan's example.


### Imitation and Flattery

Vitousek, transient, forgetful space efficiency ... something like heedful blame
but undefined behavior for relatively simple example.
Link to my webpage?

Castagna and Lanvin, forgetful contracts, only cared about soundness anyway.

> _Forgetful_ is an interesting middle ground: if contracts exist to make
> partial operations safe (and not abstraction or information hiding),
> forgetfulness may be a good strategy.
<!-- Section 10, bottom of page 23 -->

I don't know any heedful imitators besides myself.


### Well why not eidetic?

Simpler
easier to implement
faster

talk about oopsla contracts, overhead ... huge amount of work, but huge payoff merged in Racket xxx
but many contracts remain


### Strawmen

> Since _eidetic_ and [classic contracts] behave the same, why bother with
> _forgetful_ and _heedful_? First and foremost, the calculi offer insights
> into the semantics of contracts: the soundness of _forgetful_ depends on a
> certain philosophy of contracts; _heedful_ relates to threesomes without blame
> [Siek and Wadler 2010]. Second, we offer them as alternative points in the
> design space. Finally and perhaps cynically, they are strawmen---warp up
> exercises for _eidetic_.
<!-- Section 1, bottom of page 2 -->

Why didn't forgetful and heedful appear in the POPL paper?
In hindsight they shed light on their successors.

The quote above about strawmen might have encouraged the review committee to
ask for their removal.
Who knows.
But surely nobody wants to waste their time with a ``strawman'' design
or acknowledge that their current work is built on a foundation of straw.

So maybe next time, acknowledge the limitations of your design without
a cute allusion?

Michael Greenberg invented two other space-efficient calculi.
These calculi were purged from the final [POPL 2015](TODO) paper
 (except for a stray reference in Appendix B)
 but appear in the [extended version](TODO).
Compared to normal contracts, the
