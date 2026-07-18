---
title:  "AI-Assisted Technical Writing: When To Stop"
layout: post
---

I had a comment in a C++ codebase explaining why a particular cast was
written the way it was.  The comment kept growing... and eventually it was
longer than the code it was documenting.  So I did what seems like the
obvious modern move: I asked Claude.ai to turn it into a blog post I
could link to instead, and pulled the explanation out of the code
entirely.

I'd have thought that would have been the end of it.  *It wasn't.*

I showed the draft to Google Gemini for review.  It found a real problem:
not a style nitpick, an actual technical error.  And Gemini proposed a
fix.

But when I tried the fix, *it didn't compile.*  Which meant my original
comment, the one I'd been confidently maintaining across code reviews
for who knows how long, had been wrong the whole time. Nobody had caught it
because nobody had actually tried to compile the counterexample.

So the article went back to the drawing board.  Then I showed *that*
version to more AIs: Perplexity.ai, ChatGPT, Microsoft CoPilot... and the
article started to spiral.  Each pass surfaced something the previous
one had gotten subtly wrong or hadn't considered.  New names were coined
for things that turned out to need distinguishing.  A whitelist
mechanism got added so a shortcut I'd deliberately chosen for its
runtime characteristics could be marked "checked" rather than
"assumed."  By the end it had gone from a comment, to a blog post, to
something that reads like a lab notebook.

> The article itself is here, if you want the technical specifics
*(it's a kind of insane C++ pointer-cast rabbit hole and not the point of
this post)*:
>
> <https://ae1020.github.io/implicit-cast-vs-waypoint-cast/>

Here's the part I think is actually worth writing down, separate from
any of that: **the disagreements in this process came in two completely
different flavors, and I didn't notice how different they were until I
was deep into the second one.**

The first flavor had a referee.  When an AI told me "hey, I have a
better idea", and it wasn't, a compiler settled the argument in about
ten seconds.  Nobody had to be persuaded of anything... the code either
compiled and did the right thing, or it didn't.  Every technical
dispute in this process eventually resolved the same way: not by which
AI sounded more confident, but by going and checking.

The second flavor didn't have a referee.  Once the technical content
was solid, I asked for opinions on the writeup itself, and got a real
split.  Some models (in particular ChatGPT and CoPilot) thought it was
way too long *(many AI are generally asked to summarize, after all)*.
Claude.ai thought the length was earning its keep by showing the
mistakes happening rather than just asserting the fixed conclusion.
Nobody could compile their way to an answer about the length, because there
isn't a ground truth the way there is for "does this pointer get corrupted."
That's not a factual question.  It's a question about what the piece is *for*.
It turned out nobody (including me) had actually settled that before opinions
started flying.

What broke the deadlock wasn't more argument.  It was going back and
asking what the thing was supposed to accomplish in the first place. I
hadn't set out to write something that would win points for concision on
a general audience -- I'd set out to write something I could link from
a comment, for the **one** person, someday, who opens that file and wonders
why the cast is written that way.  Once that was explicit, most of the
length argument dissolved on its own.  Some of it didn't: a couple of
the "this is too much" critiques turned out to be right on their own
terms, about naming things you don't need to name, not about length at
all.  Those parts I kept.

I think that's the generalizable lesson, if there is one: **when you
can't tell whether a disagreement is technical or editorial, that
confusion is itself worth stopping and naming**, because the two need
completely different tools to resolve.  One needs a compiler, or an
experiment, or a fact.  The other needs you to say out loud what you're
actually trying to do. That's weirdly easy to skip when everyone in the
conversation (human or AI) jumps straight to optimizing before
anyone's confirmed what's being optimized *for*.

There's no compiler for "is this done."  At some point you just decide,
and move on.

So once technical accuracy had reached consensus, the revisions to the
casting article stopped where **I, (the human), decided**.  If you think it's
too long and the outcomes need summarization, just ask an AI.  🤖
