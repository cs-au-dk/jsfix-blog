<!-- 
  JSFIX: A superior codemod tool
  A superior semantic patch language
-->


I a [previous post](https://jsfix.live/blog/intro), we introduced JSFIX and briefly made an informal comparison with other codemod tools. 
We argued that JSFIX is a *semantics*-based codemod tool, which considers what your code does, as opposed to most other *syntax-based* codemod tools, which only consider how your code looks.
In this post, we will examine this point further, by introducing JSFIX's semantic patch language.


### A semantic patch language

We borrow the term *semantic patch* from [Lawall et al](https://dl.acm.org/doi/10.1145/1218063.1217942).
In Lawall's work, a semantic patch is a SmPL (Semantic patch language) program used by the Coccinelle program to migrate a Linux driver written in C from an older version to a newer version of the Linux kernel.

JSFIX has its own semantic patch language, which, analogous to Lawall's, serves to migrate JavaScript/TypeScript programs from an older to a newer version of some npm dependency.  
Apart from the somewhat similar purposes, JSFIX and Coccinelle and their semantic patch languages are otherwise quite different.

Let's try to write a semantic patch for the Lodash breaking change introduced in the previous post.
Recall that we want to transform

```javascript
var l = require('lodash');
function f () {
    l.reduce([1, 2, 3], (t, n) => t + n, 0, this);
}
```
to

```javascript
var l = require('lodash');
function f () {
    l.reduce([1, 2, 3], ((t, n) => t + n).bind(this), 0);
}
```
in other words, we want to take the fourth argument of calls to `reduce` and pass that argument to a `bind` method call we insert on the second argument.  
A JSFIX semantic patch for performing this transformation reads:

\$\texttt{call \<lodash\>.reduce [4,4]} \leadsto \texttt{callee(1, 2.bind(4), 3)}\$.
