---
layout: post
title: Pheidippides (Part 1)
header-img: img/pheidippides.jpg
tags: [Pheidippides, Configuration Management]
author: James Humphrey
github: leafknode
---

<!--excerpt.start-->
Besides all the math and modeling, the Activision Game Science team also does a bunch of big data engineering at Activision. At this year’s [MesosCon](http://events.linuxfoundation.org/events/mesoscon-north-america), we’ll be giving a [talk](https://mesosconna2016.sched.org/event/6jtX/all-marathons-need-a-runner-introducing-pheidippides-james-humphrey-john-dennison-activision-publishing) on an application configuration and deploy tool we built called Pheidippides (named after the apocryphal [Marathon-to-Athens runner](https://en.wikipedia.org/wiki/Pheidippides)). Pheidippides was initially designed to help us more easily deploy our services into [Marathon](https://mesosphere.github.io/marathon/), but with use it has evolved into something more.

Part 1 of this series is an abstract on our issues with configuration management, how we chose to solve these issues, and the resulting Pheidippides specification.  No code yet, but in Part 2 we'll continue with how exactly we implemented the Pheidippides specification and also provide some code that we use to glue it all together.

# Problem Statement

In any given software system, configuration is an important component to
the successful operation of that system. This is especially true in a
service-oriented system that is dominated by micro-services. In these
types of systems, micro-services can proliferate very rapidly and, as
they proliferate, the total amount of configuration across the system
can grow exponentially.
<!--excerpt.end--><!--more-->

This configuration proliferation phenomenon isn’t necessarily the
problem, though. The actual problem is somewhat subtler... How do you
manage it all? In other words:

* How and where do you store the configuration?
* How do you share configuration between different services?
* How do you provide for different sets of configuration for different
    environments (e.g. development/staging/production)?
* How do you add/edit/delete configuration and how does that impact
    existing service runtimes?

It’s these types of questions that characterize the deeper issues
associated with configuration management in a micro-service oriented
architecture and help frame the problem we’re trying to solve.

# Motivation

The primary motivation behind solving this problem is to reduce our
service runtime-deployment risk. By ‘runtime-deployment risk’ I mean the
likelihood of a misconfigured service negatively impacting a runtime
environment on deployment. With the right configuration management
solution in place, there’s less chance for a developer to make a mistake
when configuring a service for deployment and, therefore, less overall
risk of system regression or degradation.

# Approach

After some initial build vs. buy analysis we decided to build our own
solution. At a high level, our custom-build approach was more of an
organic process rather than conventional one. We came up with a set of
requirements and evolved those over time as new features and
improvements were made. This strategy was immensely successful for a
couple of reasons:

1. The changes were so small that there was very little regression on
the existing, legacy deployment mechanism.

2. This type of process provided us time to mature in the space, which
facilitated rapid iteration on our requirements with little-to-no wasted
effort.

Even though there were many factors that were important to us during
this process, the most critical ones we measured were: time-to-deploy
and risk-of-deploy. These two variables helped measure success and
helped refine and redefine our requirements.

# Results

The solution we built is a product we call Pheidippides, although
calling it a product is arguably a mischaracterization. Even though
there is some actual source code, Pheidippides is really more of a
*specification* for managing configuration. The source code that does
exist is primarily glue code that implements different components of the
Pheidippides specification.

At its core, the specification requires that micro service configuration
management be the following:

* Centralized
* Hierarchical
* Immutable

The details of which are explained below.

## Centralized

This requirement specifies that all configurations reside in and are
accessible via one location, primarily to reduce the cost of maintaining
configurations as the system evolves. This might seem like an obvious
requirement but it’s not uncommon for deployment systems to do
quite the opposite. By that I mean store configurations within
individual deployment scripts that are unique to the service it’s
deploying. As more and more services are added to the system, so are
more and more scripts needed for deployment. Eventually, the
configurations are no longer centralized to a few files but
decentralized across many different files. Over time this becomes very
difficult to maintain and, consequently, increases risk of deployment
failure.

For our implementation, we choose ZooKeeper as our centralized store for
our entire set of configuration properties.  We could have easily chosen some other
key-value store solution (e.g. etcd, consul, etc) but ZooKeeper was already a
core component to our stack so it made the most sense.  In Part 2, we'll go into much more detail on how exactly
ZooKeeper is used and why it's the best choice for us.

## Hierarchical

Another requirement was to design the system in such a way so that
services could share configuration from one to another. For example,
groups of services might share the same PostgreSQL database server, all
of which need to know the exact same information such as host and port.
Without any way to share information, it’s common to see many, identical
key-value pairs defined and distributed across many different
configuration sets in many different files. In this scenario, if, for
instance, the PostgreSQL host and port change, then changes need to be
made in more than one place. This type of change-process is very
inefficient and incredibly error prone.

In a hierarchical system, however, being able to share configuration
between services facilitates managing these types of properties and
reduces overall risk of deployment failure.

### Configuration Path

The Pheidippides specification defines this hierarchy by requiring that
each service be uniquely identified by a *configuration path*
(what we sometimes call the *application id*), which is
a URI-like path that conforms to the following [Extended Backus-Norm
Form](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form)
(EBNF) notation:

    letter = "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H" | "I" | "J" | "K" | "L" | "M" | "N" | "O" | "P" | "Q" | "R" | "S" | "T" | "U" | "V" | "W" | "X" | "Y" | "Z" | "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j" | "k" | "l" | "m" | "n" | "o" | "p" | "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" | "y" | "z";

    digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";

    symbol = "_" | "-";

    character = letter | digit | symbol;

    name = character, {character};

    delimeter = "/";

    configuration path = delimeter, name, {configuration path};

Example *configuration paths* for different services might look like the
following:

* /foo/bar/service-1
* /foo/bar/service-2
* /baz/waldo/service-1
* /baz/service-2
* /baz/service-3

If not already obvious, the *configuration path* scheme is nearly
identical in design to how directory structures are defined. Both are
recursive in nature and provide a very natural method for grouping
concepts together. Reading left to right; each successor path is a child
of its predecessor, which provides for inheriting (or sharing)
information from predecessor to successor.

### Configuration Sets

Within each *configuration path* there exists zero-to-one *configuration
set*, and each *configuration set* is composed of zero-to-many key-value
pair properties. Additionally, each successor in the *configuration
path* inherits *configuration sets* from all of its predecessors.

To help illustrate this, consider the following configuration path taken
from the earlier examples:

*/foo/bar/service-1*

As previously defined, all *configuration paths* are naturally
recursive, which means that */foo/bar/service-1* can be broken down into
the following three distinct paths:

* /foo
* /foo/bar
* /foo/bar/service-1

Within each of these paths, there can exist zero to one *configuration
set*, which is composed of zero-to-many key-value pair properties. For
example:

* /foo
  * x=1
* /foo/bar
  * y=2
  * z=3
* /foo/bar/service-1
  * w=4
  * s=${z}
  * x=6

The Pheidippides specification dictates that each successor in the path
inherits properties from its predecessor. Following this rule, the total
*configuration set* of key-value pair properties for the
*/foo/bar/service-1* service would then look as follows:

* /foo/bar/service-1
  * x=6
  * y=2
  * z=3
  * w=4
  * s=3

### Overrides

Property value overriding is another important requirement of the
specification. Pheidippides defines this specifically as: *Any successor
in the configuration path can override the value of one of its
predecessors.*

In the previous example, you might have noticed the fact that ‘x equals
‘6’ not ‘1’. This is because it was overridden in the
*/foo/bar/service-1* configuration set. This is one example of a
successor overriding a value previously defined by a predecessor.

### Variable Substitution

Another important requirement of the Pheidippides specification is to
allow successors to use values already defined by predecessors. One
example of this is illustrated earlier by setting *s=${z}* in the
*/foo/bar/service-1* configuration set.

### Immutable

Once a *configuration path* uniquely identifies a service and that
service is configured for deployment, Pheidippides requires that
any future change to a *configuration set* for that service is only
applied on next service deployment. Ultimately, this means that a
*configuration set* for any given service deployment is immutable until
next deployment.

# Conclusions

In our humble opinion, for what it’s solving, Pheidippides is an
evolutionary idea that fills a critical gap that a lot service-oriented
systems have with configuration management. It not only has
significantly lessened our runtime-deployment risk but also has
dramatically reduced the brain damage of maintaining many different and
similar configuration sets across many distinct services. Ultimately, Pheidippides
increases our confidence in the quality of our deployments, which contributes positively
to the health of our systems.

In 'Part 2' of our Pheidippides series we'll go into much more detail on how exactly we implemented the spec. 
For example, how we use ZooKeeper znodes to represent our configuration paths and also how we configure our Docker 
containers to run within our mesos-marathon cluster.
