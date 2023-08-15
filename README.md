# Introduction

Viae.ts is a Typescript library for composing and orchestrating a directed-acylic-graph of async functions. 

-  It is declarative in both aspects. It forces the client to be explicit about the inherent complexity in their problem domain
while not having to deal with the incidental complexities of orchestration. 
- It is most useful for graphs where you may care entrypointing into the graph's many subpartitions
- It also memoizes repeated node visits in a given graph traversal (think recursive fibonacci).
- The library does not aim to predict the optimal traversal order but chances are it'll be better than handwritten synchronization logic.
The optimization is for developer time.

## Usage

```javascript
const graph = new Viae();
graph.value('age', 1);
graph.async('birthday', age => Promise.resolve(age+1));
graph.entryPoint((birthday) => {
  console.log(birthday);
});
```

The best way to understand usage is to view the representative test cases in `test.ts`.

## Formalization

All nodes have names. Nodes are either value nodes or function nodes. Value nodes are leaf nodes and have no dependencies.
Function nodes depend on other nodes, which by definition can be value nodes or function nodes. 
The formal parameter of the function should match the name of the node it depends on. If the formal parameter references 
a value node, the value of the value node is resolved. if the formal parameter references a function node, the return value 
of function is resolved. Actually, since functions returns Promise<X>, it is actually X which gets resolved.

Each entrypoint call invokes a traversal of the graph. In a given graph traversal, repeated node visits are memoized. 
This may or may not be desired behavior in your case. It should be trivial to add an option as to whether the function should be memoized, but
there is currently no demand for this. 

## Development

```bash
$ npm i # install
$ npm t # test
$ npm run fmt # format
```

## Background and Motivation

This project was conceived during my time at Waze. The division of labor there involved a QA team that implemented a suite of end-to-end tests.
The system-under-test was a REST API. To makes things easier to reason about, we wanted tests to run in a clean slate. This meant provisoning a set of
REST entities for each test. These entities were hierarchical in nature and their creational dependency formed a DAG. 

The QA team's imperative was to a complete a list of test cases, BUT NOT investigate test failures nor extend tests. The test suite was written
in an imperative manner with a set of utility functions. This  is fine in theory but not in practice. These utility functions were tightly coupled and not re-usable. Editing or adding tests required being familiar with the entire set of utility functions which was becoming unwieldy. Synchronization, memoization, error handling was added ad-hoc, hapharzardly and as an afterthrought. "Callback hell" was one of the symptoms. The incremental merge requests with this short-term mindset was hard to argue against in isolation, but hollistically the it was duct-tape and bubble gum. When a test failed, it was hard to discern the root cause and decide whether there was a regression, the all too common server reliability issues, or a logic bug in the test itself. It would take atleast half a day for an engineer to reproduce because they would soon realize the utility functions were swallowing the stacktrace and producing ambiguous errors.

This API was for an ad system which has inherent complexity in the problem/business domain that is unavoidable. The QA implementation of the test suite multiplexed the problem by adding its own incidental and accidental (even worst than incidental) complexity. 

One of the main goals with the library is standardizing errors and making them understnable. When a given node/function fails, the entire graph shortcircuit. It could be that the graph would have failed for multiple reasons, but the philosophy was to fix one problem at a time. For this node that failed, we want both a clear stacktrace of the failed node aswell as the graph path leading up to that failure. Both are equally important for us, as the former characteristic by itself was ambigious. Having the latter allowed us to partially continue the test from where it failed. 

Another benefit that came out of this library is that we can define one big knowledge graph. It turns out that most test cases were small variations of other tests in
the types of resources they provisioned. After this library was introduced, people added tests can have a localized view of the subgraph they cared about and
not worry about the totality of all utility/functions.

## Alternative design considerations

### Everything is a function

The #value method is just syntatic sugar. It would have been elegant to require every node be a function.
The value nodes are just a special case of functions that have no input:

```
graph.async('age', () => Promise.resolve(1));
```

However, #value makes things much explicit that it is a leave node. Furthermore, it avoids having the boilerplate
of having to write out the function value.


### Asynchronous Everything

In an earlier implementation, there was a distinction between function vs async functions.

```
graph.function('birthday', age => age+1);
```

In practice, this function would end up wrapped into a Promise yielding function. The thing about async is that once one function is async,
everything else is async, too. It's annoying to have to wrap every return result in a Promise, but it's a small price to pay for consistency.
Also, the purpose of this library is specifically for async tasks. If we're just dealing exclusively with synchronous functions, you 
don't need this library and can just use regular functional-style composition.

One more reason is that allowed us to disambiguate the level of promise unwrapping. 

### Runtime cycle detection

Cycle detection is done at runtime. It's trivial, if not easier, to make this at compile time by requiring the user declare the nodes in topological order.
However, declaring nodes out of order provides flexible, especially if you want to use different total graphs at different points in time.


## About the name

The name Viae is derived from the latin word for road. Viae is commonly associated with
the complex network of roads in the Roman empire, which is apt for a library performing graph traversal.

The alternative meaning of viae is "argument". The philosopher Thomas Quinas provided five arguments dubbed Quinque Viae or "Five Ways" which are logical formulations predicating on the idea that things which exist in the universe are either caused by some other prior process, or uncaused. In the graph defined with the Viae.ts, there are two classes of nodes: `value` nodes which just are and `async` nodes which must be invoked to produce a value.

Finally, this project was conceived at Waze which is play on the word "ways" which is rooted from viae.
