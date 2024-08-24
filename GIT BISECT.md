## git bisect

### Intro

There is a bug in the "do-math.sh" script. It is a regression.  
We do some investigation and try the code from the last release and see that the bug is not present.
Let's pretend that looking at the code gives us no help in figuring out what the issue is.
Let's use `git bisect` to find out what commit introduced this bug.

### Gather data

Git sha's:

- current main: b0abf02dccd9dd5a286a7e57815363a05f0bcdcb
- previous release: 95a8dd01560439999ccf0e2ceace14a14a677bea

So it's easier to follow, let's give them names:

```sh
export CURRENT_MAIN_GIT_SHA=b0abf02dccd9dd5a286a7e57815363a05f0bcdcb
export PREVIOUS_RELEASE_GIT_SHA=95a8dd01560439999ccf0e2ceace14a14a677bea
```

Let's verify where we do and don't see the bug:

```sh
❯ git checkout $CURRENT_MAIN_GIT_SHA
❯ ./do-math.sh
3 + 4 = 12
10 + 12 = 120

# That looks wrong -- it is showing the bug
```

```sh
❯ git checkout $PREVIOUS_RELEASE_GIT_SHA
❯ ./do-math.sh
3 + 4 = 7
10 + 12 = 22

# That looks correct -- this is before the bug was introduced
```

## Using git bisect

We now have the three things we need in order to use git bisect:

- a bad commit
- a good commit
- a means to test whether the bug is present

```
git bisect start

git bisect good $PREVIOUS_RELEASE_GIT_SHA

git bisect bad $CURRENT_MAIN_GIT_SHA
```

Example output:

```
Bisecting: 81 revisions left to test after this (roughly 6 steps)
[a30b089805216dd2f656b44f1b251c83cfc26569] opaque commit message
```

`git bisect` has moved us to a commit about halfway between the good and the bad commit. We need to test this commit and tell whether this is a good or bad commit.  
We run our test:

```sh
❯ ./do-math.sh
3 + 4 = 12
10 + 12 = 120

# Looking at our output, we see the bug present, so we say that this is a bad commit:
❯ git bisect bad
Bisecting: 40 revisions left to test after this (roughly 5 steps)
[9d25e1b34456dab3b6a8e624609bb91bcd0e345c] opaque commit message

# We test again:
❯ ./do-math.sh
3 + 4 = 12
10 + 12 = 120
# Once again, we look at the output and tell whether this is a good or bad commit.
❯ git bisect bad
Bisecting: 20 revisions left to test after this (roughly 4 steps)
[d58e8c25d199e2df93b252f3165fce69506ff2c6] opaque commit message

# We test again:
❯ ./do-math.sh
3 + 4 = 7
10 + 12 = 22
# Here, we don't see the bug, so we mark this as a good commit:
❯ git bisect good
Bisecting: 10 revisions left to test after this (roughly 3 steps)
[e9e97b8546b040d0e329d8555ac5257fc65e4426] opaque commit message
```

Let's fast forward to the point where we get an answer:

```sh
❯ git bisect good
e7b758915ac53c6fc131b4e2672b66e8f4b53c0b is the first bad commit
commit e7b758915ac53c6fc131b4e2672b66e8f4b53c0b
Author: damonJansenDexcom <59670886+damonJansenDexcom@users.noreply.github.com>
Date:   Sat Aug 24 12:20:32 2024 -0700

    opaque commit message

 do-math.sh | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

Let's see if that makes sense

```sh
❯ git checkout e7b758915ac53c6fc131b4e2672b66e8f4b53c0b

❯ git diff HEAD~1 # looking at the diff with the previous commit
diff --git a/do-math.sh b/do-math.sh
index 08b0a24..c324620 100755
--- a/do-math.sh
+++ b/do-math.sh
@@ -2,7 +2,7 @@
set -euo pipefail

add() {
-  sum=$(($1 + $2))
+  sum=$(($1 * $2))
  echo "$1 + $2 = $sum"
}

# That looks right -- that is where `+` changed to `*`
```

What if I can't tell if a commit is good or bad?

```sh
# Imagine we ended up here:
❯ git checkout 1af3c2a2dc613ed2551dcf8224bd995e1682f9f0
❯ ./do-math.sh
./do-math.sh: line 3: typo: command not found
# We can skip this commit
git bisect skip
```
