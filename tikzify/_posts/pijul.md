# Introduction

In the [last post](TODO), I talked about a mathematical framework for a version
control system (VCS) without merge conflicts. In this post I'll explore
[pijul](pijul.com), an implementation of such a system. Note that pijul
is under heavy development; this post is based on a development snapshot
(I almost called it a "git" snapshot by mistake), and might be out of
date by the time you read it.

The main goal of this post is to describe how pijul handles what other VCSes
call conflicts. We'll see an example where pijul's approach works better than
git's, and also some (non-constructive, unfortunately) complaints about pijul's
behavior.

# Some basics

I don't want to write a full pijul tutorial here, but I do need to mention
the basic commands if you're to have any hope of understanding the rest
of the post. Fortunately, pijul commands have pretty close analogues in
other VCSes.

- `pijul init` creates a pijul repository, much like `git init` or `hg init`.
- `pijul add` tells pijul that it should start tracking a file, much like `git
  add` or `hg add`.
- `pijul record` looks for changes in the working directory and records a patch
  with those changes, so it's similar to `git commit` or `hg commit`. Unlike
  those two (and much like `darcs record`), `pijul record` asks a million
  questions before doing anything; you probably want to use the `-a` option to
  stop it.
- `pijul fork` creates a new branch, like `git branch`. Unlike `git branch`,
  which creates a copy of the current branch, `pijul fork` creates a copy of
  the master branch. (This seems like a strange default to me.)
- `pijul apply` adds a patch to the current branch, like `git cherry-pick` or
  `hg merge`.

# Dealing with conflicts

As I explained in the last post, pijul differs from other VCSes by not having
merge conflicts. Instead, it has (what I call) *dagles*, which are different
from files in that their lines are only partially ordered (i.e. form a directed
acyclic graph). The thing about dagles is that you can't really work with them
(for example, by opening them in an editor), so pijul doesn't let you actually
see the dagles: it stores them as dagles internally, but renders them as files
for you to edit. As an example, we'll create a dagle by asking pijul to
perform the following merge:

```tikz
FILE (0, 0) original
to-do
* work

FILE (50, 15) mine
to-do
* shoes
* work

FILE (50, -15) wife's
to-do
* garbage
* work

DAGLE (100, 0) merged
PARENT 0 POS 1/1 to-do
PARENT 1 POS 2/1 * shoes
PARENT 1 POS 2/2 * garbage
PARENT 2/3 POS 3/1 * work

EDGES
a1 b1
a2 b3
a1 c1
a2 c3
b1 d1
b2 d2
b3 d4
c1 d1
c2 d3
c3 d4
````

Here's the pijul commands to do this:

```
$ pijul init

# Create the initial file and record it.
$ cat > todo.txt << EOF
> to-do
> * work
> EOF
$ pijul add todo.txt
$ pijul record -a -m todo

# Switch to a new branch and add the shoes line.
$ pijul fork --branch=master shoes
$ sed -i '2i* shoes' todo.txt
$ pijul record -a -m shoes

# Switch to a third branch and add the garbage line.
$ pijul fork --branch=master garbage
$ sed -i '2i* garbage' todo.txt
$ pijul record -a -m garbage

# Now merge in the "shoes" change to the "garbage" branch.
# Note that we need to know the hash of that patch, which can be determined with
# `pijul changes --branch=shoes`
$ pijul apply <hash-of-shoes-change>
```

The first thing to notice after running those commands is that pijul
doesn't complain about any conflicts. I'm not actually sure that's
a good decision, but we'll discuss that more later. Anyway, if you run
the above commands then the final, merged version of `todo.txt` will
look like this:

```tikz
FILE (0, 0) merged
to-do
* shoes
>>>>>>>>>
* garbage
<<<<<<<<<
* work
```

That's... a little disappointing, maybe, especially since pijul was supposed to
free us from merge conflicts, and this looks a lot like a merge conflict. The
point, though, is that pijul has to somehow produce a file -- one that the
operating system and your editor can understand -- from the dagle that it maintains internally.
The output format just happens to look a bit like what other VCSes output when
they need you to resolve a merge conflict.

As it stands, pijul doesn't have a way to actually see its internal dagles, so
I hacked one in: [this](TODO) fork of pijul has a `pijul graphviz` command for
outputting a dagle in a format to be rendered by graphviz. It's a total hack
(for example, it doesn't even correctly escape quotation marks, so your file
had better not have any), but in this case it's good enough to show that pijul
produced the correct dagle after all:

```tikz
DAGLE (100, 0) merged
PARENT 0 POS 1/1 to-do
PARENT 1 POS 2/1 * shoes
PARENT 1 POS 2/2 * garbage
PARENT 2/3 POS 3/1 * work
```

## What should I do with a conflict?

Since pijul will happily work with dagles internally, you could in principle
ignore a conflict and work on other things. That's probably a bad idea for
several reasons (for starters, there are no good tools for working with dagles,
and their presence will probably break your build). The authors of pijul
explicitly decided to avoid imposing opinions on your workflow, since
nobody has much experience with dagle-based VCSes. So here's an opinion from
me: when you have a conflict, you should resolve it ASAP. I'll give more
reasons later.

In the example above, all we need to do is remove the `>>>` and `<<<` lines
and then record the changes:

```
$ sed -i 3D;5D todo.txt
$ pijul record -a -m resolve
```

# Case study: reverting an old commit

It's (unfortunately) common to discover that an old commit introduced
a show-stopper bug. On the bright side, every VCS worth its salt has some way
of undoing the problematic commit without throwing away everything else you've
written since then. But if the problematic commit predates a merge conflict,
undoing it can be painful.

As an illustration of what pijul brings to the table, we'll look at
a situation where pijul's conflict-avoidance saves the day (at least,
compared to git; darcs also does ok here).
We'll start with the example merge from before, including
our manual dagle resolution:

```tikz
FILE (0, 0) original
to-do
* work

FILE (50, 15) mine
to-do
* shoes
* work

FILE (50, -15) wife's
to-do
* garbage
* work

DAGLE (100, 0) merged
PARENT 0 POS 1/1 to-do
PARENT 1 POS 2/1 * shoes
PARENT 1 POS 2/2 * garbage
PARENT 2/3 POS 3/1 * work

FILE (150, 0) resolved
to-do
* shoes
* garbage
* work

EDGES
a1 b1
a2 b3
a1 c1
a2 c3
b1 d1
b2 d2
b3 d4
c1 d1
c2 d3
c3 d4
d1 e1
d2 e2
d3 e3
d4 e4
````

Then we'll ask pijul to revert the "shoes" patch:

```
$ pijul unrecord --path=<hash-of-shoes-patch>
$ pijul revert
```

The result? We didn't have any conflicts while reverting the old patch,
and the final file is exactly what we expected:

```tikz
FILE (150, 0)
to-do
* garbage
* work
```

Let's try the same thing with git:

```
$ git init

# Create the initial file and record it.
$ cat > todo.txt << EOF
> to-do
> * work
> EOF
$ git add todo.txt
$ git commit -a -m todo

# Switch to a new branch and add the shoes line.
$ git checkout -b shoes
$ sed -i '2i* shoes' todo.txt
$ git commit -a -m shoes

# Switch to a third branch and add the garbage line.
$ pijul checkout -b garbage master
$ sed -i '2i* garbage' todo.txt
$ git commit -a -m garbage

# Now merge in the "shoes" change to the "garbage" branch.
$ git merge shoes
Auto-merging todo.txt
CONFLICT (content): Merge conflict in todo.txt
Automatic merge failed; fix conflicts and then commit the result.
```

That was expected: there's a conflict, so we have to resolve it. So I edited
`todo.txt` and manually resolved the conflict. Then,

```
# Commit the manual resolution.
$ git commit -a -m merge
# Try to revert the shoes patch.
$ git revert <hash-of-shoes-patch>
error: could not revert 4dcf1ae... shoes
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'
```

Since git can't "see through" my manual merge resolution, it can't handle
reverting the patch by itself. I have to manually resolve the conflicting
patches both when applying and reverting.

I won't bore you with long command listings for other VCSes, but you can test
them out yourself! I've tried mercurial (which does about the same as git here)
and darcs (which does about the same as pijul).

# Why you should resolve your conflicts

One of the reasons that I came out against working with dagles above is that
the way pijul renders dagles as files makes it difficult to do much with them
(to be fair, I don't see how to do it any better). Here are a couple of issues
that I found while playing with pijul.

## Pijul's file rendering is lossy

The first problem is that information is lost in pijul's rendering.
For example, here are two different dagles:

```tikz
DAGLE (0, 0)
PARENT 0 POS 1/1 to-do
PARENT 1 POS 2/1 * shoes
PARENT 2 POS 3/1 * work
PARENT 1 POS 3/2 * garbage
PARENT 4 POS 4/2 * shop
PARENT 3/4 POS 4/1 * home

DAGLE (50, 0)
PARENT 0 POS 1/1 to-do
PARENT 1 POS 2/1 * shoes
PARENT 2 POS 3/1 * work
PARENT 1 POS 2/2 * garbage
PARENT 4 POS 3/2 * shop
PARENT 3 POS 4/1 * home
PARENT 5 POS 4/2 * home
```

But pijul renders both in the same way:

```tikz
FILE (0, 0)
to-do
>>>>>>>>>
* shoes
* work
* home
=========
* garbage
* shop
* home
<<<<<<<<<
```

This is a perfectly good representation of the dagle on the right, but it loses
information from the one on the left (such as the fact that both "home" lines
are the same, and the fact that "shop" and "home" don't have a prescribed
order. The good news here is that as long as your dagle came from merging two
*files*, then pijul's rendering is lossless. That means you can avoid the
problem by flattening your dagles to files after every merge (i.e., by
resolving your merge conflicts immediately).

## Pijul's dagle parsing is funky

The second problem is that pijul's diffing algorithm doesn't interact perfectly
with pijul's rendering algorithm. Say you edit a dagle (rendered by pijul as
a file) to produce another file that pijul is supposed to interpret as a dagle.
If you try to record that change, you're relying on pijul to correctly
interpret your modified file as a dagle. It isn't hard to come up with an
example that doesn't work. Let's start with this dagle:

```tikz
DAGLE (100, 0)
PARENT 0 POS 1/1 to-do
PARENT 1 POS 2/1 * shoes
PARENT 1 POS 2/2 * garbage
PARENT 2/3 POS 3/1 * work
```

Pijul renders this as the file

```tikz
FILE (0, 0)
todo
* shoes
>>>>>>>>>
* garbage
<<<<<<<<<
* work
```

Now let's try swapping the shoes and garbage lines. If we were to swap the
shoes and garbage lines in the dagle, we would end up with the exact same
dagle. But what if we try swapping those lines in the file that pijul rendered?
In that case, pijul interprets the result like this:

```tikz
DAGLE (100, 0)
PARENT 0 POS 1/1 to-do
PARENT 1 POS 2/2 * garbage
PARENT 1/2 POS 3/1 * work
PARENT 2 POS 3/2 >>>>>>>>>
PARENT 4 POS 4/2 * shoes
```

That's right: pijul interpreted the `>>>>>>>>>` part (but not `<<<<<<<<<`) as
a line of actual content!

All in all, it's better to avoid working with dagles yourself; leave that to
pijul's internals. Like [cockroaches](http://www.livescience.com/33995-cockroaches.html),
dagles are important for the ecosystem as a whole, but you should still flatten them
as soon as they appear.

# Conclusion

So hopefully you have some idea now of what pijul can and can't do for you.
It's an actively developed implementation of an exciting (for me, at least) new
way of looking at patches and merges, and it has a simple, fast, and totally
lossless merge algorithm. Will it dethrone git? Not any time soon, at least;
besides git's huge head start, here are a few reasons:

- Pijul is alpha-quality and under heavy development; not only should you be
  worried about your data, it has several UI warts as well.
- Although I can find examples demonstrating the drawbacks of traditional VCSes
  like git, it isn't so clear how often you'd run into them in practice.
- Although pijul's guarantees (like the irrelevance of merge order) *could*
  have workflow benefits, it isn't clear what they are and how important they
  are.
- "Dagle" is a terrible, terrible name. That's nothing to do with pijul as such;
  I take full responsibility.

In the next post, I'll take a look at pijul's innards, focussing particularly on
how it represents your precious data.

