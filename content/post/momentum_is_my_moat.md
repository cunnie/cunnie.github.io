---
title: "Momentum is my Moat"
date: 2026-02-22T18:28:53-08:00
draft: true
---

I run a DNS service, [sslip.io](https://sslip.io), a service so simple that at
least 4 [0] different people have created an identical service. One even said,
"I built [similar service] when I couldn't quite remember" [1] sslip.io's name.
That's right, it was easier to build from scratch than spend a few minutes on
Google. And this was before Claude.ai made coding easy!

This begs the question: "If it's so easy to code your service, if it's so easy
to spin up a VPS (virtual private server), then why should anyone use your
service over someone else's? **What's your moat?**"

Simple. My moat is momentum. I've kept my service running, day in, day
out, for over ten years. No outages. It just works.

We celebrate creation, and why not? Creating something new is exciting, at
times breathtaking, especially among developers. Hacker News, that nexus of
developers, has a special section, "[Show
HN](https://news.ycombinator.com/showhn.html)", where we show something that
we've "made that other people can play with". But there's no celebration of something that's been quietly maintained for years and years. We take that for granted[2]

But there's another side to that coin, one decidedly less glamorous: keeping
the service running. Creation is only half the battle; maintenance is the
other.

And boy, has there been maintenance, almost from the get-go. In fact, when I
first built sslip.io, I didn't write code — I shamelessly copied Sam
Stephenson's setup for xip.io. My twist was how it was deployed, using an
opinionated Ansible-style tool named BOSH. It was about the maintenance from
day one.

---

[0]

- xip.io
- nip.io
- backname.io
- traefik.me

[1] Twixes, describing why he built backname.io, a service identical to
sslip.io <https://github.com/Twixes/backname/issues/14>

[2] https://www.explainxkcd.com/wiki/index.php/2347:_Dependency
