---
title: The Birth of a New Stemcell
date: 2022-11-30T10:53:49-08:00
draft: true
---

_How the BOSH Ecosystem team, when confronted with a last-minute
release-stopping bug, was able to turn lemons into lemonade._

We're on the BOSH Ecosystem Team, and we were excited—we were about to release
our first new stemcell line in over 6 years. The stemcell was good to
go, other teams were testing it, and we were going to launch it the following
week.

_(A stemcell is a heavily customized Linux release based on Canonical's Ubuntu;
it's used to create the virtual machines (VMs) which run applications such as
Tanzu Application Service for VMs. Our earlier stemcell line is based on
Ubuntu Xenial, originally released in 2016. Our newest stemcell line, the
subject of this post, is based on Ubuntu Jammy, released in 2022)_

And then we got a Slack message that brought our plans to a screeching halt:
the MySQL team reported, "... failures in our CI [continuous integration] due
to BOSH timeouts when using the new Jammy stemcell." We tried some quick fixes
("try reducing the number of Director threads", "try increasing the RAM and
CPUs on your Director"), and although they worked, they weren't permanent
fixes. Besides, other teams were seeing problems, too.

We began performance analysis, which may seem like a black art, but it's not:
there's only four things to look at: CPU, RAM, disk, and network. And we
quickly zeroed-in on RAM. The BOSH Director was svelte on the Xenial stemcell,
bloated on the Jammy stemcell. Technically speaking, the Resident Set Size of
the BOSH Director workers on Xenial was a mere 90 MB, but on Jammy a massive
480 MB.

This was a problem.

We couldn't understand how this could be happening: we were running identical
Ruby code on Xenial and on Jammy, and yet the runtime characteristics were
radically different. We needed to dig deeper, move down a layer: it wasn't our
code, maybe it was the Ruby interpreter (the BOSH Director is written in Ruby,
and the Ruby interpreter runs the code).

We were able to write a small snippet of Ruby which exposed the difference: But
why? Why did Ruby behave differently on Xenial than on Jammy? After all, we
were using the same version of Ruby on both stemcells, compiled with the same
flags.

A day into our exploration, Brian Upton, a lead engineer, looked grim. "We're
gonna miss our release date," he said.

It's true. Barring a miracle, we weren't going to make our October release
date.

As a side note: it's easy to talk about code quality when you're making all
your deadlines, but the real test is when you must choose between meeting a
deadline or shipping a good release. The team was unanimous: miss the deadline,
ship a good release. Our managers, Maya Rosecrance and Manual Alba, backed us
up. Each of us had spent time on support calls with customers; we were keenly
aware of the pain caused by software bugs.

Back to the exploration: We resorted to using tools such as `strace` to see
when memory was allocated, but the only thing we learned is that Ruby on Jammy
allocated a lot more memory than Ruby on Xenial. We already knew that.

During our investigation, we discovered something that stunned us: when we
moved the Xenial Ruby onto a Jammy stemcell (and this was not easy; ask us how
we handled shared libraries if you want to hear a horror story), it did not
exhibit the bloat.

In other words, it didn't seem to matter whether the Ruby interpreter was run
on Xenial or Jammy. What mattered was whether the Ruby interpreter was
_compiled_ on Xenial or Jammy.

GCC (Gnu Compiler Collection) was now our prime suspect: it was the C compiler
we used to compile our Ruby interpreter. This is a venerable compiler, first
released in 1987 by Richard Stallman, and in some ways it could be considered
the opening salvo in the war for free software, but that's a story for another
day.

Xenial used GCC version 5.4.0, Jammy, 11.3.0. We went on a hunting expedition to
find out which versions of GCC introduced the bloat, and we narrowed it down to
somewhere in the 9.x release (introduced in 2019).

On a whim we tried an alternate compiler, the Apple-sponsored Clang compiler.
The biggest advantage was that it was flag-compatible with GCC, which meant it
was an easy test, and we were astounded with the results.

We ran 132 BOSH Deploys of 10 VMs per deploy. Our Xenial-based BOSH Director—the
gold standard—averaged 369 seconds/deploy; Our Jammy-GCC-based BOSH Director was
14% slower, averaging 421 seconds/deploy, but the star of the show was the
Jammy-Clang-based Director, clocking in 189 seconds/deploy, almost twice as fast
as the Xenial-based Director!

{{< figure src="https://docs.google.com/spreadsheets/d/e/2PACX-1vS4QxKPJTfDn-z9CQkqsJaRNoo_5TGYxSSAzHzsySMMFx2L6jw5pc187v9WPjU1N2KByhILdbqsCj24/pubchart?oid=1792670048&format=image" alt="Vault logo"  caption="Benchmarks run on vSphere, with a 2 vCPU, 4 GiB Director. Times are shown for 2 simultaneous deploys of 10 VMs apiece. The VMs had no BOSH jobs (the weren't running anything).">}}

This was exciting news—one of the most frequent requests from customers was
to speed up our deploy times, and we had accidentally stumbled on a solution.

These speed-up enhancements made it into Operations Manager (OM) 3.0, and when
you upgrade to OM 3.0 and click “Apply Changes”, you should see faster deploys,
but temper your enthusiasm: your BOSH deploys won’t be twice as fast as the
chart above would indicate. The chart above represents a best-case scenario and
doesn’t consider other factors such as the Diego Cell’s drain phase, which
dominates the deploy time of Tanzu Application Service for VMs. Our
back-of-the-envelope calculations suggest that most users would see deploy time
improvements of 5-10%.

This wasn't the end of the challenges we faced—integrating the Clang compiler
required a significant amount of engineering—but we were out of the woods, and
within weeks we had successfully launched Operations Manager 3.0, which was
based on the Jammy stemcell.
