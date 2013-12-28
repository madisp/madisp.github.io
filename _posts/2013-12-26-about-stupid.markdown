---
layout:     post
title:      A Stupid Scripting Language for Java/JVM
date:       2013-12-27 23:02:45
categories: android, bad, stupid
---

```
> fib = { |x| x < 2 ? 1 : fib.(x-1) + fib.(x-2) }
com.madisp.stupid.Block@783c342b
> fib.(5)
8
> fib.(6)
13
> fib.(7)
21
```

#### Obligatory Links and Foreword

* [source](https://github.com/madisp/stupid)
* [javadoc](http://madisp.com/stupid)
* [binaries](http://dl.bintray.com/madisp/maven/com/madisp/stupid/stupid/) (will be on maven central soon)

Some time ago [I tried to replicate](https://github.com/madisp/bad) AngularJS's variant of two-way databinding in Android. I find Angular's approach to data-binding absolutely lovely. It has a simple expression language that one can use within the templates, so naturally, I needed something similar.

Having never written a compiler nor an interpreter I felt that coming up with something on my own would be an insurmountable task. It would probably have been true if it weren't for ANTLR4. The grammar language is nice and clean, the runtime is easy to use. In a short time I was running a prototype version of stupid in a REPL. A couple of months later I needed a scripting language in another project, hence I decided to pull the language out of its original library and call it stupid. Because, well, the language really is pretty stupid.

I wanted a really concise (think XML-attributes concise) and simple language for evaluating expressions. Hence some ugly-ish decisions.

#### Arithmetic

Integer (and double) arithmetic works as expected.

```
> 2 + 3
5
> 2 + 3 * 4
14
> (2 + 3) * 4
20
> 5/3
1
> 5/3.0
1.6666666666666667
```

#### Accessing Java

If wired up with a ReflectionContext then one can call Java methods and access java fields. As implied by the name, reflection is used.

```
> 'foo'.getClass()
class java.lang.String
> 'foo'.hashCode()
101574
> 'foo'.equals('bar')
false
> 'foo'.equals('foo')
true
```

For a more thorough example, check the [calculator](https://github.com/madisp/stupid-samples/blob/master/calculator/src/main/java/com/madisp/stupid/samples/Calculator.java) sample.

#### Variables

It is possible to evaluate stupid code with a context that allows creating new variables that are only usable within stupid.

```
> a
null
> a = 'foo'
foo
> a + 'bar'
foobar
> a.b
null
> a.b = 'foobar'
foobar
> a.b
foobar
```

#### Type Coercion

In stupid almost anything can evaluate to a boolean (and yes, it has a ternary operator):

```
> 'foo' ? true : false
true
> '' ? true : false
false
> 0 ? true : false
false
> null ? true : false
false
```

Trying to mix types with operators leads to a bit unexpected results, but shouldn't fail:

```
> 3 * null
0
> 3 * 'foo'
0
> 3 * false
0
> 3 > 'foo'
true
```

#### Boolean Logic

Works mostly like in Java, only the operators are `and` and `or` instead of `&&` and `||`.

```
> !false and true
true
> true or false and false
true
> (true or false) and false
false
> false and thisWillNeverBeCalled()
false
> false or thisWillDefinitelyBeCalled()
null
```

One cool feature is that a boolean operator expression returns the value of the last evaluated expression (thats why the last line in the previous example is `null` - the method `thisWillDefinitelyBeCalled` doesn't exist and hence calling it yields null). This enables one to set variables to default values, e.g.:

```
> foo
null
> bar = foo or 'bar'
bar
> bar
bar
```

#### Blocks

Stupid has the notion of anonymous snippets of code - blocks. These are more like contained goto calls and not proper closures - stupid doesn't really have a notion of local variables or captures.

```
> sqr = { |x| x*x }
com.madisp.stupid.Block@68814013
> sqr.(2)
4
> add = { |x,y| x + y }
com.madisp.stupid.Block@d0721b0
> add.(3, 5)
8
```

#### More Samples

See the [stupid-samples](https://github.com/madisp/stupid-samples) repo in github.