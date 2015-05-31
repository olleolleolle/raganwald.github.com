---
layout: default
title: "Minimalism: On the necessity of `const`, `let`, and `var` in JavaScript"
---

*Disclaimer: JavaScript the language has some complicated edge cases, and as such, the following essay has some hand-wavey bits and some bits that are usually correct but wrong for certain edge cases. If it helps any, pretend that nearly every statement has a footnote reading, "for most cases in practice, however \_\_\_\_\_\_."*

Before there was a `let` or a `const` in JavaScript, there was `var`. Variables scoped with `var` were either *global* if they were evaluated at the top-level, or *function-scoped* if they appeared anywhere inside a function declaration or function expression.

[![Gerrit Rietveld's Roodblauwe stoel](/assets/images/roodblauwe-stoel.jpg)](https://www.flickr.com/photos/matijagrguric/4439360724)

### is "var" necessary?

As a thought experiment, let's ask ourselves: Is `var` really necessary? Can we write JavaScript without it? And let's make it interesting: Can we get rid of `var` without using `let`?

In principle, we can replace declared variables with *parameters*, initializing them to `undefined` if the original code didn't assign them a value. So:

{% highlight javascript %}
function callFirst (fn, larg) {
  return function () {
    var args = Array.prototype.slice.call(arguments, 0);
    
    return fn.apply(this, [larg].concat(args))
  }
}
{% endhighlight %}

Would become:

{% highlight javascript %}
function callFirstWithoutVar (fn, larg) {
  return function (args) {
    args = Array.prototype.slice.call(arguments, 0);
    
    return fn.apply(this, [larg].concat(args))
  }
}
{% endhighlight %}

We can manually hoist any `var` that doesn't appear at the top of the function, so:

{% highlight javascript %}
function repeat (num, fn) {
  var i;
  
  for (i = 1; i <= num; ++i)
    var value = fn(i);
  
  return value;
}
{% endhighlight %}

Would become:

{% highlight javascript %}
function repeat (num, fn, i, value) {
  i = value = undefined;
  
  for (i = 1; i <= num; ++i)
    value = fn(i);
  
  return value;
}
{% endhighlight %}

There are a few flaws with this approach, most significantly that the code we write is misleading to human readers: It clutters the function's signature with its local variables.[^arity] Fortunately, there's a fix: We can wrap function bodies in IIFEs[^iife] and give the IIFEs the extra parameters. Like this:

[^arity]: It also changes the arity of our functions. That can matter for certain meta-programming implementations. 

{% highlight javascript %}
function repeat (num, fn) {
  return ((i, value) => {
    for (i = 1; i <= num; ++i)
      value = fn(i);
  
    return value;
  })();
}
{% endhighlight %}

Now our function has its original signature, and we have the expected behaviour. The flaw with this approach, of course, is that our function is more complicated both in code *and* behaviour: There's this confusing `return ((i, value) => {` and `})();` stuff going on, and even though we all love the techniques espoused in [JavaScript Allongé][ja6], this appears a bit gratuitous.

And at runtime, we are creating an extra closure with every invocation. This has performance implications, memory implications, and it certainly isn't doing our stack traces any favours.

But we get the general idea: If we were willing to live with this code, we could get rid of a lot or even all uses of `var` from our programs. Now, what about `let`?

### is "let" necessary?

`let` has a more complicated behaviour, but if we are careful, we can translate `let` declarations into IIFEs just like `var`. The simplest case is when a `let` is at the top-level of a function. In that case, we can replace it with a `var`. And from there, if we are removing both `let` and `var`, we can excise it completely. So:

{% highlight javascript %}
function arraySum (array) {
  let done,
      sum = 0,
      i = 0;
  
  while ((done = i == array.length, !done)) {
    sum += array[i++];
  }
  return sum
}
{% endhighlight %}

Would become

{% highlight javascript %}
function arraySum (array) {
  var done,
      sum = 0,
      i = 0;
  
  while ((done = i == array.length, !done)) {
    sum += array[i++];
  }
  return sum
}
{% endhighlight %}

And then:

{% highlight javascript %}
function arraySum (array) {
  return ((done, sum, i) => {
    sum = i = 0;
    
    while ((done = i == array.length, !done)) {
      sum += array[i++];
    }
    return sum
  })();
}
{% endhighlight %}

That's fairly obvious. Now what about `let` inside a block? This is, after all, it's claim to fame. The least complicated case is when the body of the block does not contain a `return`. In that case, we use the same IIFE technique, but don't return anything. So this variation:

{% highlight javascript %}
function arraySum (array) {
  let done,
      sum = 0,
      i = 0;
  
  while ((done = i == array.length, !done)) {
    let value = array[i++]
    sum += value;
  }
  return sum
}
{% endhighlight %}

Would become:

{% highlight javascript %}
function arraySum (array) {
  return ((done, sum, i) => {
    sum = i = 0;
    
    while ((done = i == array.length, !done)) {
      ((value) => {
        value = array[i++];
        sum += value;
      })();
    }
    return sum
  })();
}
{% endhighlight %}

By the way, the performance is worse than rubbish, because we're creating and discarding our IIFE on every trip through the roof. In cases, like this, we can avoid a lot of that by cleverly  "hoisting" the IIFE out of the loop:

{% highlight javascript %}
function arraySum (array) {
  return ((done, sum, i, __iife) => {
    sum = i = 0;
    
    __iife = (value) => {
      value = array[i++];
      sum += value;
    };
    
    while ((done = i == array.length, !done)) {
      __iife();
    }
    return sum
  })();
}
{% endhighlight %}

[![Rietveld's Hanging Lamp](/assets/images/hanging-lamp.jpg)](https://www.flickr.com/photos/59633635@N08/6433053889)

### loops and blocks

`let` has special rules for loops. So if we simplify our `arraySum` with a `for...in` loop, we'll need an IIFE around the `for` loop to prevent any `let` within the loop from leaking into the surrounding scope, and one inside the `for` loop to preserve its value within the block. Let's write a completely contrived function:

{% highlight javascript %}
function sumFrom (original, i) {
  let sum = 0,
      array = original.slice(i);
  
  for (let i in array) {
    sum += array[i];
  }
  return `The sum of the numbers ${original.join(', ')} from ${i} is ${sum}`
}
{% endhighlight %}

This can be rewritten as:

{% highlight javascript %}
function sumFrom (original, i) {
  return ((sum, array) => {
    sum = 0;
    array = original.slice(i);
  
    ((i, __iife) => {
      __iife = (i) => {
        sum += array[i];
      };
      
      for (i in array) __iife(i);
    })();

    return `The sum of the numbers ${original.join(', ')} from ${i} is ${sum}`
  })()
}
{% endhighlight %}

Some blocks contain a `return`, and that returns from the nearest enclosing function. But if we replace the block with an IIFE, the `return` will return to the IIFE. When the IIFE surrounds the entire body of the function, we can just return whatever the IIFE returns, as we do above. But when the IIFE represents a block within the body of the function, we can only return the value of the block if it returns something.

So something like this:

{% highlight javascript %}
function maybe (fn) {
  return function (...args) {
    for (let arg of args) {
      if (arg == null) return null;
    }
    return fn.apply(this, args)
  }
}
{% endhighlight %}

Becomes this:

{% highlight javascript %}
function maybe (fn) {
  return function (...args) {
    return ((arg, __iife, __iife_return_value) => {
      __iife = (arg) => {
        if (arg == null) return null;
      };
      
      for (arg of args) {
        __iife_return_value = __iife(arg);
        if (__iife_return_value !== undefined) return __iife_return_value;
      }
      return fn.apply(this, args)
    })();
  }
}
{% endhighlight %}

We'll leave it as "an exercise for the reader" to sort out how to handle a `return` that doesn't return anything:

{% highlight javascript %}
function maybe (fn) {
  return function (...args) {
    for (let arg of args) {
      if (arg == null) return;
    }
    return fn.apply(this, args)
  }
}
{% endhighlight %}

Or a return when we don't know what we are returning:

{% highlight javascript %}
function maybe (fn) {
  return function (...args) {
    for (let arg of args) {
      if (arg == null) return arg;
    }
    return fn.apply(this, args)
  }
}
{% endhighlight %}

[![The Rietveld Schröderhuis](/assets/images/reitveld-schroederhuis.jpg)](https://www.flickr.com/photos/kjbo/4444981118)

### what have we learnt from removing "var" and "let"?

The first thing we've learnt is that for most purposes, `var` and `let` aren't *strictly* necessary in JavaScript. Roughly speaking, scoping constructs with lexical scope can be mechanically transformed into  functional arguments.

This is not news, it's how `let` was originally written in the Scheme flavour of Lisp, and it's how `do` works in CoffeeScript to provide let-like behaviour.

So one argument is, we could strip these out of the language to provide a more minimal set of features.

However, looking at the code we would have to write if we didn't have `var` or bette still, `let` without `var`, it's clear that while a language without `let` would be smaller, the programs we write in it would be larger. 

This is a case where taking something away does not create elegance. If we take `let` away and only use `var`, we have to add IIFEs to get block scope. If we take `var` away too, we get even more IIFEs. Removing `let` makes our programs less elegant.

### wait, what about "const"?

As you know, `const` behaves exactly like `let`, however when a program is first parsed, it is analyzed, and if there are any lines of code that attempt to assign to a `const` variable, an error is generated. This happens *before* the program is executed, it's a syntax error, not a runtime error.

Presuming that it compiles correctly and you haven't attempted to rebind a `const` name, `const` is exactly the same as `let` at runtime. Therefore, removing `const` from a working program is as simple as replacing it with `let`. So the following:

{% highlight javascript %}
function sumFrom (original, i) {
  let sum = 0;
  const array = original.slice(i);
  
  for (let i in array) {
    sum += array[i];
  }
  return `The sum of the numbers ${original.join(', ')} from ${i} is ${sum}`
}
{% endhighlight %}

Can first be translated to:

{% highlight javascript %}
function sumFrom (original, i) {
  let sum = 0,
      array = original.slice(i);
  
  for (let i in array) {
    sum += array[i];
  }
  return `The sum of the numbers ${original.join(', ')} from ${i} is ${sum}`
}
{% endhighlight %}

And thereafter we can leave it be (if we don't like `const` or carry on and get rid of the `let` statements as well, if we like.

### one of these things is not like the others

As we can see, `const` is not like `var` or `let`. Removing `var` by changing it into parameters involves the creation of additional IIFEs, cluttering the code and changing the runtime behaviour. Removing `let` adds much more complexity again. But removing `const` by changing it into `let` is benign. It doesn't add any complexity to the code or the runtime behaviour.

This is not surprising: `const` isn't a scoping construct, it's a *typing* construct. It exists to make assertions about the form of the program, not about its runtime behaviour. That's why languages like C++ implement `const` as an annotation on top of an existing declaration. If JavaScript followed the same philosophy, `const` would be an annotation on top of an existing declaration:

It might look something like this:

{% highlight javascript %}
@const function sumFrom (original, i) {
  let sum = 0;
  let @const array = original.slice(i);
  
  for (let i in array) {
    sum += array[i];
  }
  return `The sum of the numbers ${original.join(', ')} from ${i} is ${sum}`
}
{% endhighlight %}

`const` is not like `var` or `let`. It ay seem like the others, and get lumped alongside them in descriptions of variables and scoping, but it's something else.

### do we need "const"?

If we think of `const` as being `let` plus an annotation that speaks to the compiler, we can understand why some people suggest we not use it at all: It does provide some convenient checking when first writing your program, but thereafter it adds complexity for the reader in the form of a fifth scoping construct (the others being globals, function arguments, `var`, and `let`).

Having more language features is not free, it means more surface area to absorb. And as much as `const` may seem ridiculously simple, if you are coming from another language, is it obvious that you mean that the reference is `const`, or might you wonder if it means that the subject of the reference is `const`?

On the other hand, it serves as a comment that the "compiler" enforces. Does that help a future reader of your program? There's a reasonable argument that if you can't see at a glance whether a name is rebound within a function, refactor that function. So how important is it to tell a future reader, "We both know this variable isn't rebound, but FYI the function will break if you're rewriting something and rebind this value."

It seems obvious that if you change the binding of an existing variable, you might break something. Nobody assumes that if a variable is declared with `let`, that you can rebind it at will without breaking anything.

`const` also seems, well... Incomplete. Once you have the idea that it's a good thing to have the compiler check this for you, doesn't it seem like a good idea to be able to have the compiler check that when you write `foo.bar()`, that `foo` is always known to have a `bar` method?

And also, declaring a property that can't be written looks nothing at all like declaring a variable that can't be rebound. No surprise there, one is a runtime check, the other is a compile-time check. But it's inelegant to have to know that there are two similar but different things, with different syntaxes.

It's not clear that we *need* const the way we need `let`.

### so… should we use "const"?

One can see the argument that `const` shouldn't be in the language *at all*, that it should have been deferred until there was a way to programatically annotate variables with properties that could be statically applied. Or that it should have been deferred until we got gradual typing. Or just left out.

Bt that isn't what happened, so we have the feature. And therefore, we are obliged to understand the feature. Given that we are obliged to understand the feature, shouldn't we use it?

That, my dear reader, is very much a question of taste. If you look around, you can find arguments that we should prefer `var` to `let` in cases where they have equivalent behaviour, thus signalling that something special is going on when you see `let`.

If you follow that argument, you can argue that we should prefer `let` to `const`, because their behaviour is *always* equivalent. Or you can flip it around and argue that you should prefer `const` to `let` in cases where they have equivalent behaviour, leaving `let` only for cases where `var` doesn't block scope and `const` won't compile.

It could be that the community at large will have to flip a coin.

[^iife]: "Immediately Invoked Function Expressions"
  

---

This post was extracted from a draft of the book, [JavaScript Allongé, The "Six" Edition][ja6]. The extracts so far:

[ja6]: https://leanpub.com/javascriptallongesix

* [OOP, Javascript, and so-called Classes](http://raganwald.com/2015/05/11/javascript-classes.html),
* [Left-Variadic Functions in JavaScript](http://raganwald.com/2015/04/03/left-variadic.html),
* [Partial Application in ECMAScript 2015](http://raganwald.com/2015/04/01/partial-application.html),
* [The Symmetry of JavaScript Functions](http://raganwald.com/2015/03/12/symmetry.html),
* [Lazy Iterables in JavaScript](http://raganwald.com/2015/02/17/lazy-iteratables-in-javascript.html),
* [The Quantum Electrodynamics of Functional JavaScript](http://raganwald.com/2015/02/13/functional-quantum-electrodynamics.html),
* [Tail Calls, Default Arguments, and Excessive Recycling in ES-6](http://raganwald.com/2015/02/07/tail-calls-defult-arguments-recycling.html), and:
* [Destructuring and Recursion in ES-6](http://raganwald.com/2015/02/02/destructuring.html).

---