---
title: Cache Invalidation is Hard
date: 2021/9/17
description: I thought I had to solve a hard thing.
tag: web development, programming
author: Felipe
---

## My first blog post ever

Ever since I came across [Blitz.js][0], I've been thinking about rewriting
[Contatempo][1] using it. I've also been thinking about creating a blog for a
long time (I guess it was my 2018's new year resolution). Now that I'm learning
Blitz I figured it would be a great motivation to document my learings and
thoughts on a blog. So, hopefully, this will be the first of many posts to
come.

## Context

For this very first post, I'll write about my adventures on facing one the
[two hard things][2] in computer science: cache invalidation. Or so I thought
that was my problem (spoiler alert: it wasn't, at least not the root cause).
I'm sure that nowadays the problem I faced shouldn't be as hard as it was back
then when Phil Karlton coined the phrase, but it took me 3 days (after spending
more than a week not even thinking about it) to find a solution.

So, one of the first things I'm working on the rewrite is the feature that
shows the total time elapsed on the records (a record has a begin date and an
end date), including the time elapsed for the ongoing record. The total elapsed
time is then composed of two parts: the first one being the elapsed time of the
completed records, for which I created a `getTotalTime` query, and the second
one being the elapsed time from the begin date of the ongoing record until now,
for which I used the `getRecords` query that returns the list of records to be
displayed and then filtered the ongoing record. The total is the sum of the two
parts. (And now I realize that maybe `getTotalTime` was not the best name, as
it not actually the **total** total time)

## The problem

For now the creation (or the start) and the finishing (or the end) are set by a
single Start/Stop button. When it is first click (displaying the "Start" label)
a new record is created and when it is clicked again (now displaying the "Stop"
label) that same record is updated with and end date. When any on those actions
are performed, I want to use Blitz's [invalidateQuery][3] on both the
`getRecords` and `getTotalTime` queries, so both the list of records and the
total elapsed time are updated. This worked, but as I went testing I noticed
that sometimes (actually it was every time) the total elapsed time blinked
right after a record was stopped! A `console.log` showed that it was actually
being briefly rendered with an unexpected value. After several hours I found an
explanation for this unexpected value (or should I say values): sometimes the
total elapsed time would include the time of the just ended record twice,
showing a higher value than expected, and sometimes it would not include the
time of the just ended record, showing a lower value than expected. But why?

## Why it occurred

After another several hours, on the next day, I finally realized what was
happening: when the elapsed time showed a higher value, it was because the
cache for the `getTotalTime` query was invalidated first, then triggering a
refetch for that query with the just finished record accounted for, but the
cache for the `getRecords` wasn't invalidated yet, making the same just
finished record time still be included and therefore be counted twice.
Similarly, when the elapsed time showed a lower value it was because the cache
for the `getRecords` query was invalidated first, triggering a refetch
including the just finished record on the list of records, which would make the
second part of the total elapsed time (the part which counted the elapsed time
from the ongoing record) be zero, but the `getTotalTime` query would still not
include this just finished record.

## The cause

The problem then was that I needed a way to invalidate both caches at the same
time so this brief inconsistent state could disappear. I went through a lot the
documentation of both Blitz and react-query (which is what Blitz uses under the
hood) and, with the help of the Blitz community on discord, I came across
[`queryClient.invalidateQueries`][4]. With that I could invalidate both queries
in a single call. I replaced both `invalidateQuery` calls with the single
`queryClient.invalidateQueries` and did a test. And then, just like that, the
problem was **still** happening! Why??

## The real cause

A day later I had a new theory: it doesn't matter if I invalidate both queries
at the same time! Maybe React was already doing it for me even when I was
calling invalidateQuery for each query. The problem really was that, after both
queries were invalidated, both were refetched on the background, independently
of one another, and each refetch would trigger a new render. This new theory
made me realize that the problem was that I had a piece of information (the total elapsed time) that was dependent on two separate, indepent queries. And it
even made me notice a bug I could take a while to notice: the `getRecords`
query was paginated, and it may not always return the ongoing record, which
would result in the total elapsed time being calculated wrong.

## The solution

So, after some consideration I came to the conclusion that, instead of trying
to change the behaviour of the framework and library at play, the best way to
fix this problem was to rethink my queries, and make them return all the
information needed for what they were created. So what I finally did was to
change the `getTotalTime` query to also return the ongoing record, so the total
elapsed time would need only this query to be calculated, a
[simple solution][5] that made much more sense.

Oh, I also renamed `getTotalTime` to [getFinishedRecordsTimeAndOngoingRecord][6]. Yes, naming things really is hard 😅.

[0]: https://blitzjs.com
[1]: https://contatempo.herokuapp.com
[2]: https://www.martinfowler.com/bliki/TwoHardThings.html
[3]: https://blitzjs.com/docs/resolver-client-utilities#invalidate-query
[4]: https://react-query.tanstack.com/guides/query-invalidation#query-matching-with-invalidatequeries
[5]: https://github.com/fmilani/contatempo/commit/cf3c28c060fd676ddde499a441cd9f64d909958d
[6]: https://github.com/fmilani/contatempo/commit/60dab26f856ace3beca16a3b12746e6ea0f52d17
