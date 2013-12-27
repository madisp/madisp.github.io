---
layout:     post
title:      Stupid - a stupid scripting language for Java/JVM
date:       2013-12-26 15:48:00
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

Obligatory links:

* [source](https://github.com/madisp/stupid)
* [javadoc](http://madisp.com/stupid)
* [binaries](http://dl.bintray.com/madisp/maven/com/madisp/stupid/stupid/) (will be on maven central soon)

Some time ago [I tried to replicate](https://github.com/madisp/bad) AngularJS's variant of two-way databinding in Android. I find Angular's approach to data-binding absolutely lovely. It has a simple expression language that one can use within the templates, so naturally, I needed something similar.

Having never written a compiler nor a interpreter I felt that coming up with something on my own would be an insurmountable task. It would probably have been true if it weren't for ANTLR4. The grammar language is nice and clean, the runtime is easy to use. In a short time I was running a prototype version of stupid in a REPL. A couple of months later I needed a scripting language in another project, hence I decided to pull the language out of its original library and call it stupid. Because, well, the language really is pretty stupid.