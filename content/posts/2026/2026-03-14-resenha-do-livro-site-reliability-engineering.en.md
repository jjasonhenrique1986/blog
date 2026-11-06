---
title: "Review of Google's Site Reliability Engineering (SRE) book"
slug: "2026-03-14-review-of-the-site-reliability-engineering-book"
description: "A review of Google's Site Reliability Engineering book: SLOs, error budgets, toil, monitoring, on-call and blameless postmortems."
images: ["/images/capa-livro-sre.png"]
author: Jason
date: 2026-03-14
tags: [ "SRE", "DevOps", "Books", "Observability" ]
type: post
---

After the review of the
[Platform Engineering]({{< relref "/posts/2026/2026-01-31-livro-engenharia-de-plataforma" >}}) book,
I decided to go back to a classic: **Site Reliability Engineering: How
Google Runs Production Systems**, known as "Google's SRE
book". In Brazil it was published by Novatec as **Engenharia de
Confiabilidade do Google**, and the English version is available for
free at [sre.google](https://sre.google/books/).

The book came out in 2016, but it's still very current. Many terms we
use day to day today, such as SLO, error budget and toil, became
popular because of it.

Below, I share the main lessons.

---

## What SRE is

The book brings a definition by Ben Treynor Sloss, who created the discipline at
Google, that sums up the idea well: SRE is what happens when you ask
a **software engineer to design an operations team**.

In practice this means treating operations as an engineering
problem: instead of solving everything manually, the team writes software
to automate, measure and improve the reliability of the systems.

A principle that shows up right at the beginning: the SRE team should cap
operational work at **50% of its time at most**. The rest should go
to engineering projects that reduce that operational work in the
future.

---

## Embracing risk: 100% isn't the goal

One of the points that most changes the way you think is the idea that **100%
availability is the wrong target**. Each additional "nine" is very expensive,
and the user usually doesn't even notice the difference, because their network,
phone or internet provider fail more often than that.

That's where the concept of **error budget** comes from: if the goal is 99.9%
availability, the remaining 0.1% is a budget that can be
"spent" on changes, deploys and experiments.

-   Budget left? The team can ship features
    faster.
-   Budget gone? Risky changes stop and the focus shifts to
    stability.

What I find most interesting here is that the error budget settles a
classic conflict between development (which wants to ship fast) and
operations (which wants stability) with an **agreed number**, not with
opinions.

---

## SLI, SLO and SLA

The book makes the difference between these three terms very clear, which many
people use as synonyms:

-   **SLI** (Service Level Indicator): the measured metric. Example: the
    percentage of requests answered successfully in under
    300 ms.
-   **SLO** (Service Level Objective): the target for that indicator.
    Example: 99.9% of requests over a 30-day window.
-   **SLA** (Service Level Agreement): the contract with the customer, with
    consequences (usually financial) if it isn't met.

A practical tip from the book: the internal SLO should be **stricter** than the
SLA, so the team notices and reacts to the problem before breaking the
contract.

---

## Eliminating toil

The book gives a name to something everyone in operations knows well:
**toil**. It's work that is:

-   manual
-   repetitive
-   automatable
-   tactical (reactive)
-   without enduring value
-   growing in proportion to the service

Not all operational work is toil, but too much toil is what
keeps the team from evolving. That's why the 50%-of-time-for-engineering
rule is so important.

---

## Monitoring and the four golden signals

The chapter on monitoring distributed systems presents the
**four golden signals**, which to this day are a great reference to
start monitoring any service:

-   **Latency**: the time it takes to serve a request, separating
    successful ones from failed ones.
-   **Traffic**: the demand on the system, such as requests per second.
-   **Errors**: the rate of requests that fail.
-   **Saturation**: how "full" the service is, such as CPU,
    memory or queue usage.

The book also insists that **alerts must be actionable**: if someone is
woken up in the middle of the night, it should be for something that requires immediate human action.
Alerts nobody can resolve only cause fatigue and make the team
ignore the important ones.

If you want to put these concepts into practice on Kubernetes, you can
start with the article about
[Prometheus and Grafana]({{< relref "/posts/2025/2025-12-06-observabilidade-no-kubernetes-com-prometheus-e-grafana" >}}).

---

## On-call and incident response

The chapters about on-call are very human. The book argues
that on-call has to be **sustainable**: a limited number of
incidents per shift, time to analyze each one calmly and
compensation for the effort.

In incident management, the main point is to have **clear roles**
during the crisis, such as who coordinates the incident, who works on the fix and
who handles communication. That avoids the situation where everyone
touches everything at the same time and nobody knows the real state of the problem.

---

## Blameless postmortems

Perhaps the most important chapter from a cultural point of view is the one on
**blameless postmortems**. After an incident, the
goal is to understand **what** allowed the failure to happen and what
can be improved in the system and the processes, not to find **who**
made the mistake.

If people are afraid of being punished, they hide information, and the
same problem happens again. A good postmortem has a timeline,
impact, root cause and, above all, **improvement actions with an owner and a
deadline**.

---

## Automation and simplicity

The book also has chapters on automation, release engineering and
simplicity. Two ideas stuck with me:

-   **Automation** isn't just about saving time: it brings
    consistency and lets the system scale without growing the team in the
    same proportion.
-   **Simplicity** is a reliability feature. Every line
    of code and every new feature is one more thing that can break.
    Removing code that is no longer used is also engineering work.

---

## Conclusion

**Site Reliability Engineering** isn't a book to read in one go from
start to finish. It's a collection of relatively
independent chapters, and you can start with the topics that make the most sense for
where you are. For beginners, I recommend the chapters on
risk, SLOs, toil, monitoring and postmortems.

Even if your company doesn't have a team called SRE, the concepts
apply to any **DevOps, platform or infrastructure** team.
Together with the Platform Engineering book, it forms a very
good foundation for thinking about how to deliver fast without giving up reliability.

It's also worth checking out Google's other books on the topic,
available on the same site: **The Site Reliability Workbook**, more
hands-on, and **Building Secure and Reliable Systems**.
