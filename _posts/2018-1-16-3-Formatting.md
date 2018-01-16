---
layout: post
title: Formatting the Code
---

## Task ##

After modernizing our code-base by removing all the `register`s and converting
from K&R to ANSI declarations, the next thing to do before diving into the
code-base is to reformat it.

I'm a big believer in code-formatters. I think that they help protect against
certain kinds of errors, and they make the code-base much easier to work with.
I don't really care _what_ coding style a project has, but I always want it
to be consistent within the project.

Also, it can't be misleading like this snippet (found when compiling):

```
[nick@localhost runit-2.1.2]$ package/compile 
Linking ./src/* into ./compile...
Compiling everything in ./compile...
+ cd compile

...

./compile sv.c
sv.c: In function ‘main’:
sv.c:293:3: warning: this ‘if’ clause does not guard... [-Wmisleading-indentation]
   if (!(action =*argv++)) usage(); --argc;
   ^~
sv.c:293:36: note: ...this statement, but the latter is misleadingly indented as if it were guarded by the ‘if’
   if (!(action =*argv++)) usage(); --argc;
                                    ^~
```

Yeeeuuchh.

## Fix ##

It took an awful lot of screwing around for me to get a combination of formatting tricks
that worked well enough for me to apply them across the entire code-base. I'm still not
100% happy with the results, but it's a start. And it's a good way to keep my changes
consistent within the code-base.

It doesn't take anywhere near this many hacks and tweaks to _keep_ something in a
readable state, but getting it there in the first place can be messy.

The overall steps in my Makefile's formatter recipe are:

1. Turn #includes into comments. This prevents them from being reordered by formatting
   tools I'll be using. Normally I'd love to re-order the includes, but Runit's code
   base has some position-dependent inclusions.

2. Trim all excess whitespace everywhere.

3. Run the result through `clang-format` with a long line-length (because I like `astyle`'s
   line-breaking results better).

4. Run the result through `astyle` with my preferred style options.

5. Run the result through `clang-format` again with a normal line-length for some
   clang-specific effects.

6. Remove any newlines after an `=` character. Clang loves to stick these in for some reason.
   I did this with a few regexes.

7. Remove any whitespace near line-continuations. `Clang-format` loves to add these. Upstream
   clang-format has some nice controls to disable that behavior, but I couldn't get them
   working on my install for whatever reason. Probably user error.

8. Run the result through a second pass with `astyle` to reflow everything.

9. Restore the #includes.

Here's what the final recipe looked like:

```make
format-%: %
	cp -a $* $*.temp.c
	sed -i 's@^ *#include@//DUMMYinclude@g' $*.temp.c
	sed -ri 's/[ \t]+/ /g' $*.temp.c
	clang-format -style="{BasedOnStyle: llvm, ColumnLimit: 1024}" -i $*.temp.c
	astyle --options=astyle.opts $*.temp.c
	clang-format -style="{BasedOnStyle: llvm, ColumnLimit: 80, AlignAfterOpenBracket: true}" -i $*.temp.c
	cat $*.temp.c | perl -pe 's/[=][ \t]*\n/= /g' > $*.temp2.c && mv $*.temp2.c $*.temp.c
	cat $*.temp.c | sed -r 's/[=][ \t]+/= /g' > $*.temp2.c && mv $*.temp2.c $*.temp.c
	cat $*.temp.c | sed -r 's/[=] [#]if/=\n#if/g' > $*.temp2.c && mv $*.temp2.c $*.temp.c
	sed -ri 's/[ \t]+\\$$/ \\/g' $*.temp.c
	astyle --options=astyle.opts $*.temp.c
	sed -i 's@^// *DUMMYinclude@#include@g' $*.temp.c
	if ! diff -q $*.temp.c $* 1>/dev/null; then cp $*.temp.c $*; fi
	rm $*.temp.c
```

## Results ##

The output isn't quite as nice as if it had been hand-tuned, but it's a hell of a lot
easier to read (for me, at least!) than the original source.

Here's an example file from upstream:

`fd_copy.c` (before formatting):
```c
/* Public domain. */

#include <unistd.h>
#include <fcntl.h>
#include "fd.h"

int fd_copy(int to,int from)
{
  if (to == from) return 0;
  if (fcntl(from,F_GETFL,0) == -1) return -1;
  close(to);
  if (fcntl(from,F_DUPFD,to) == -1) return -1;
  return 0;
}
```

And here's the reformatted result:

`fd_copy.c` (after formatting):
```c
/* Public domain. */

#include <unistd.h>
#include <fcntl.h>
#include "fd.h"

int fd_copy(int to, int from)
{
    if (to == from) {
        return 0;
    }

    if (fcntl(from, F_GETFL, 0) == -1) {
        return -1;
    }

    close(to);

    if (fcntl(from, F_DUPFD, to) == -1) {
        return -1;
    }

    return 0;
}
```

Notes:

For whatever reason, my code formatters don't want to put braces around control
statements (if, while, for, etc) that have a single nested control statement.

So while I'd like

```c
if (linelen == linemax) for (;;) {do_thing();}
```

to become:

```c
if (linelen == linemax) {
    for (;;) {
        do_thing();
    }
}
```

instead I get:

```c
if (linelen == linemax)
    for (;;) {
        do_thing();
    }
```

Not sure why. It's a little bit annoying, but still better than the situation before
using the code-formatter.
