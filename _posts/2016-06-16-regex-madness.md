---
layout: post
title:  "Regex madness"
date:   2016-06-16 12:57:59
categories: regex
tags: regex
---
{% include JB/setup %}

In order to make my brain melt, I just need to look at this:
```
((?:[^ ]|[^ ] [^ ])+)(?:  +|$)
```

Q: What the hell does it do?
A: It parses a row where columns are separated by two or more spaces!
Q: Why the hell would you do that?
A: Because Huawei

Here's my header line

```
Route name        Route Number  Priority
```

and I want to get the field headers `Route name`, `Route Number` and `Priority` out of the above.  Note that there are two spaces between `Route Number` and `Priority`.

Let's simplify the problem to the point of uselessness and build up from there:

You have a single column containing multiple instances of a

```
aaaaa
```
regex would be
```
(a+)$
```

The column can include a space, but only in the middle

```
aaaaa
aa aa a
```

Here I want to match either an `a` or an `a a` one or more times. Let's do this
* `a` or `a a` = `(a|a a)`
* That multiple times = `(a|a a)+`
* Capture the above by wrapping it with `()`

so

```
((a|a a)+)$
```

But look at that in [regex101.com][r101noncap].  It leaves us with the problem that there are now two capturing groups.  The inside parens are there only to provide the choice, we don't really care abou them.  Enter [Regular Expressions Non-Capturing Groups][noncapture].  Put a `?:` at the beginning of the group:

```
((?:a|a a)+)$
```
Nice!  What next.  Let's have multiple columns separated by two spaces.  What we're actually looking for after our capturing is actually either two spaces **OR** the end of line `$`.  So `(  |$)`.... buuuut that creates another [captured group][r101capspace], therefore `(?:  |$)`.

[Version 3][r101v3] is

```
((?:a|a a)+)(?:  |$)
```

Let's change the separator to 2 or more spaces `  +`

```
((?:a|a a)+)(?:  +|$)
```

And we're nearly there.  The only problem is the character `a` is quite limiting.  What about if we were to change it to the opposite of a space `[^ ]`.  I've saved this step for last because it visually complicates things.  Here is the [final regex][r101final]:

```
((?:[^ ]|[^ ] [^ ])+)(?:  +|$)
```


[r101noncap]: https://regex101.com/r/oS3uZ1/1
[r101capspace]: https://regex101.com/r/oS3uZ1/2
[r101v3]: https://regex101.com/r/oS3uZ1/3
[r101final]: https://regex101.com/r/oS3uZ1/4
[noncapture]: http://www.regular-expressions.info/brackets.html#noncap
