---
layout: post
title: Removing legacy C constructs
---

**Task**

Runit must have been written a pretty long time ago originally. Either
that or the author was super old-school. All of the functions were defined
in K&R style instead of ANSI, and the 'register' keyword is sprinkled
liberally through the design.

Here's an exampe of an old-style K&R function declaration:

```c
int alloc_re(x,m,n)
char **x;
unsigned int m;
unsigned int n;
{
  return 1;
}
```

And here's the same function modernized to an ANSI declaration:

```c
int alloc_re(char **x, unsigned int m, unsigned int n)
{
    return 1;
}
```

There's nothing syntactically wrong with K&R function declarations, but
I think of them as a code-smell for code that hasn't been kept modern.
Time to fix them.

**Fix**

Runit is a big enough code-base that it's not practical to go through
it and change every function declaration by hand (especially without
a unit-test suite!).

Through some googling, I came across a few references to a tool called
`cproto`, which knows how to convert K&R function declarations to ANSI.
Cproto worked for me, but I had to jump through some hoops with it to
make it work.

Notably, it seemed to get tripped up by the `#include` headers, which
it insisted on trying to parse.

To get around this, I wound up using `sed` to turn every #include into
a temporary command before I ran cproto on each file (and then again
to fix it afterwards).

Combined with another `sed` operation to delete `register` keywords,
and I had the following Makefile pattern rule:

```make
refactor-%: %
	cp -a $* $*.temp
	sed -i 's@^ *#include@//DUMMYinclude@g' $*.temp
	cproto -a -E 0 $*.temp
	sed -ri 's/\bregister\b//g' $*.temp
	sed -i 's@^//DUMMYinclude@#include@g' $*.temp
	if ! diff -q $*.temp $* 1>/dev/null; then cp $*.temp $*; fi
	rm $*.temp
```

**Notes**

It's interesting what people notice in a code base. A lot of programmers
would look at upstream Runit's K&R declarations and be scared off. Friendly
code attracts new developers!
