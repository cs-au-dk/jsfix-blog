Have you ever given up on upgrading an npm package?
Maybe the API was so vastly different in the new version that upgrading felt ...
Or maybe the changelog was incomplete or full of errors making it difficult to make the required changes.
At Coana, we certainly have struggled with dependency upgrades countless times.
Upgrading dependencies can be a cumbersome process, and for good reasons:

- **There is no standard practice for how to write and distribute changelogs**
Some package developers provide [comprehensive migration guides](https://v6.rxjs.dev/guide/v6/migration) with step-by-step guides to upgrading while [others](https://github.com/graphql/graphql-js/blob/main/resources/gen-changelog.js) auto-generate changelogs from the list of accepted pull requests since the last release.
- **Changelogs are often incomplete or missing details**
<a name="lodash-example"></a>
For example, the [Lodash version 4 changelog](https://github.com/lodash/lodash/wiki/Changelog#v400) containing the *Removed thisArg params from most methods because they were largely unused, complicated implementations, & can be tackled with _.bind, Function#bind, or arrow functions* that fails to specify exactly which functions are affected (There are 64 in total). 
- **Developers are short on time**
And let's be honest, even with sufficient time, upgrading dependencies is not really the most interesting task.  

So why not just fix dependency versions and never worry about upgrades?
Aside from the fact that we might want the features and improvements from the new version, there is also the whole question of security vulnerabilities.
...

***

### Introducing JSFIX
JSFIX is a tool that automatically migrates your npm application to use a new version of an npm dependency.
Unlike tools such as [Dependabot](https://github.com/dependabot) or [Snyk](https://snyk.io/), JSFIX doesn't just modify your package.json to depend on some new version of a dependency, *it also adjusts your code to the new API of the dependency*.
If we consider the [Example from Lodash](#lodash-example) above, JSFIX finds each call in your code base to the 64 affected functions, and then transforms those calls to use the `bind` function.

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

Try this example yourself in our [playground](https://jsfix.live/playground)

***

### Is it just another AST transformer?
The short answer is *no*, JSFIX is mostly semantics-based wheres most classic AST transformers (such as [jscodeshift](https://github.com/facebook/jscodeshift)) are syntax-based.  
In less academic terms:

> where most other tools primarily consider *how your code looks*, JSFIX also considers *what your code does*. 

So what does all that mean in practice? Let's go back to the example above.
We can easily write a jscodeshift patch making the same transformation on the example above.
Informally, such a patch would read *when calling `reduce` on the object `l` with 4 arguments, insert a call to `bind` on the second argument passing the fourth argument from the `reduce` call to the `bind` call*.
Such a patch works well if you know that `l` and only `l` contains the lodash module object. 
If in another file, you load lodash into a variable named `lodash` (`var lodash = require('lodash')`), then you have to extend the jscodeshift patch to handle that case as well.
Even worse, consider what would happen if you try to make the patch general enough to handle every application using lodash out there: There may be hundreds or even thousands of different variables holding the lodash module object. 
We could try to simplify the patch to look for calls to a method called `reduce` taking 4 arguments (no constraint on the receiver/base object), but then we have to pray that no application calls another `reduce` method with 4 arguments - clearly, an unrealistic assumption.
You can probably already see that making the patch general enough to handle all cases is intractable.
In general: 

> jscodeshift works well when code follows strict conventions, but fall short otherwise.

Let's now consider what JSFIX does differently. 
With JSFIX, patches are formalized using small special-purpose programs known as *semantic patches* (a term coined by [Lawall et al](https://dl.acm.org/doi/10.1145/1218063.1217942)).
...

### How can I get started?

### Can I write my own patches?
