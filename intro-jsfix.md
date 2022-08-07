Have you ever given up on upgrading an npm package?
Maybe the API was so vastly different in the new version that upgrading felt intractable.
Or maybe the changelog was incomplete or full of errors making it difficult to make the required changes.
At Coana, we certainly have struggled with dependency upgrades countless times.
Upgrading dependencies can be a cumbersome process, and for good reasons:

- **There is no standard practice for how to write and distribute changelogs**.
Some package developers provide [comprehensive migration guides](https://v6.rxjs.dev/guide/v6/migration) with step-by-step guides to upgrading while [others](https://github.com/graphql/graphql-js/blob/main/resources/gen-changelog.js) auto-generate changelogs from the list of accepted pull requests since the last release.

- **Changelogs are often incomplete or missing details**.
For example, the [Lodash version 4 changelog](https://github.com/lodash/lodash/wiki/Changelog#v400) contains *Removed thisArg params from most methods because they were largely unused, complicated implementations, & can be tackled with _.bind, Function#bind, or arrow functions* that fails to specify exactly which functions are affected (There are 64 in total). 

- **Developers are short on time**.
And let's be honest, even with sufficient time, upgrading dependencies is not really the most interesting task.  

***

### Introducing JSFIX
JSFIX is a tool that automatically migrates your npm application to use a new version of a dependency.
Unlike tools such as [Dependabot](https://github.com/dependabot) or [Snyk](https://snyk.io/), JSFIX doesn't just modify your package.json to depend on some new version of a dependency, *it also adjusts your code to the new API of the dependency*.
For the Lodash example above, JSFIX finds all calls to the 64 affected functions, and then transforms those calls to use the `bind` function.

For example,

```javascript
var l = require('lodash');
function f () {
    l.reduce([1, 2, 3], (t, n) => t + n, 0, this);
}
```
is transformed to

```javascript
var l = require('lodash');
function f () {
    l.reduce([1, 2, 3], ((t, n) => t + n).bind(this), 0);
}
```

Try this example yourself in our [playground](https://jsfix.live/playground).

***

### So how is JSFIX different from other codemod tools?
In essence, JSFIX is mostly semantics-based whereas most classic codemod tools (such as [jscodeshift](https://github.com/facebook/jscodeshift)) are syntax-based.  
In less academic terms:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; **where most other tools primarily consider *how your code looks*, JSFIX also considers *what your code does*.**

So what does this mean in practice? Let's go back to the example above.
We can easily write a jscodeshift patch for making the transformation above.
Informally, such a patch would read "*when calling `reduce` on the object `l` with 4 arguments, insert a call to `bind` on the second argument passing the fourth argument from the `reduce` call to the `bind` call*."
Such a patch works well if you know that `l` and only `l` contains the Lodash module object. 

However, if in another file, you load Lodash into a variable named `lodash` (`var lodash = require('lodash')`), then you have to extend the jscodeshift patch to handle that case as well.
Even worse, consider what would happen if you try to make the patch general enough to handle every application using Lodash out there: There may be hundreds or even thousands of different variables holding the Lodash module object. 

We could try to simplify the patch to look for calls to all methods named `reduce` taking 4 arguments (no constraint on the receiver/base object), but then we have to pray that no application calls another `reduce` method with 4 arguments - clearly, an unrealistic assumption.

You can probably already see that making the patch general enough to handle all cases is practically impossible.
In general: 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; **jscodeshift works well when code follows strict conventions, but falls short otherwise.**

Let's now consider what JSFIX does differently. 
With JSFIX, patches are formalized using small special-purpose programs known as *semantic patches* (a term coined by [Lawall et al](https://dl.acm.org/doi/10.1145/1218063.1217942)).
A semantic patch for patching the Lodash example reads:

```
  call <lodash>.reduce [4,4] -> $callee($1, $2.bind($4), $3)
```
Informally: "*When calling reduce on the Lodash module object with four arguments, insert a call to bind ...*".
Unlike the jscodeshift example, JSFIX does not care what name you give the Lodash module object.
You can write Lodash to whatever variable you want or even write Lodash to another variable, JSFIX will still find the correct calls to `reduce`:

```javascript
var l = require('lodash');
var foo = l;
var bar = foo;
function f () {
    bar.reduce([1, 2, 3], (t, n) => t + n, 0, this);
}

```

Try it yourself with the *Lodash 4.0.0: Removal of thisArg* example on the [playground](https://jsfix.live/playground).
If you think JSFIX is cheating, you can also try to call reduce on some variable that is not Lodash: 

```javascript
var l = require('lodash');
var r = require('some-random-npm-package');
function f () {
    r.reduce([1, 2, 3], (t, n) => t + n, 0, this);
}

```

The playground will then report that *The input program is not affected by the breaking change*.

So how does JSFIX know which variables hold Lodash module object? 
JSFIX uses several static programs analyses to reason about the semantics (or meaning) of the program - that is why we say that JSFIX is semantics-based.
Exactly how these analyses work is out-of-scope for this post, but more information is provided in our [open-access research papers](https://jsfix.live/about-jsfix#research).

***

### How can I get started?

If you head over to [https://jsfix.live/dependency-scanner](https://jsfix.live/dependency-scanner) you can check if JSFIX can help you with upgrades already today.
You can also sign up for notifications about future upgrades relevant for your application.

In case you are wondering how JSFIX works on a real application, you can try the full pipeline on a Lodash client [here](https://jsfix.live/jsfix/https%253A%252F%252Fgithub.com%252Fmtorp%252Fminimongo).

If you have any questions or suggestions, join our [public slack channel](https://join.slack.com/t/jsfix-community/shared_invite/zt-16615g0xc-I4cwSWn9Ghdl4CqfcCz7rQ) or write an email to [contact@coana.tech](mailto:contact@coana.tech).
