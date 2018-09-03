---
title: Modeled interventions in mrgsolve
author: ~
date: '2018-09-02'
slug: modeled-interventions-in-mrgsolve
tags: ["events"]
---



# Introduction

This post is to introduce modeled interventions in mrgsolve. The main use
case for this functionality is to force mrgsolve to advance the 
system to a specific time so that some aspect of the system 
can change at that time.  This is similar to the `MTIME` functionality
that NONMEM provides.  Additionally, doses (bolus or infusions)
can also be implemented as a modeled event.  These events are "modeled"
because they originate from code in the `$MAIN` block, not the 
input data set, so that the events can be reactive to model parameters
or the state of the model itself. 

# Status
At the time of this post, this functionality is only in a development
branch on the mrgsolve GitHub.  But we've been working on and 
revising this concept for a while now, so hopefully it is 
more or less stable with a reasonable interface.

# Example: time-varying KA

One common use case for modeled events are a time-varying 
model parameter such as an absorption rate constant.  In 
this example, we want KA to start at the time of the oral
dose and then increase at some time after the dose.  And we 
want that to happen for every dose.  We implement that 
by modeling that time after the dose when we want KA to 
change and implement at that change point.

The model might look like this

```{r}
code <- '
$PARAM CL = 1, V = 20, KA1 = 0.1, KA2 = 0.5, KA3 = 5

$PKMODEL cmt = "GUT CENT", depot = TRUE

$MAIN

if(EVID==1) {
  double KA = KA1;
  self.mevent(TIME + 2, 33);
}

if(EVID==33) KA = KA3;

' 
```

Whenever we see a dose (`EVID==1`), we set `KA` to the initial (slower)
`KA` value and register a model event that (in this case) happens 
two hours later.  We use `EVID=33` for that event.  Then, when
we see `EVID==33` come along, we change the `KA` to the faster value.

The simulation looks like this

```{r}
library(mrgsolve)
mod <- mcode("mevent", code)

mod %>% 
  ev(amt = 100) %>% 
  mrgsim(end = 24, delta = 0.1) %>%
  plot(CENT~time)
```

We can refine this even more with an additional intervention 
time for an intermediate value of `KA`

```{r}
code <- '
$PARAM CL = 1, V = 20, KA1 = 0.1, KA2 = 0.5, KA3 = 5

$PKMODEL cmt = "GUT CENT", depot = TRUE

$MAIN

if(EVID==1) {
  double KA = KA1;
  self.mevent(TIME + 1, 33);
  self.mevent(TIME + 2, 34);
}

if(EVID==33) KA = KA2;
if(EVID==34) KA = KA3;

' 
```


The arguments for `self.mtime()` in this case are:

1. time
1. evid

We use a separate `EVID` here to distinguish the first change 
from the second change.


```{r}
mod <- mcode("mevent2", code)

mod %>% 
  ev(amt = 100) %>% 
  mrgsim(end = 12, delta = 0.1) %>%
  plot(CENT~time)
```

Now we have the two change points implemented. Multiple doses would 
look like this:

```{r}
mod %>% 
  ev(amt = 100, ii = 24, addl = 2) %>% 
  mrgsim(end = 120, delta = 0.1) %>%
  plot(CENT~time)
```
  

# Example: modeled dosing events

In the previous example, we used the `mevent` call to 
to force the simulation to stop at the time we wanted the 
parameter to change. This was accomplished by an event object
with `EVID=2`, which forces the simulation (and to the solver)
to stop at the time of the event, reset, and then keep going 
(with the new model parameters).

We can also use this mechanism to implement actual doses like this

```{r}
code <- '
$PARAM CL = 1, V = 20, KA = 2

$PKMODEL cmt = "GUT CENT", depot = TRUE

$MAIN

if(TIME/12 ==floor(TIME/12) && TIME < 120) {
  self.mevent(TIME, 1, 1, 100, 0);
}

' 
```


The arguments for `self.mtime()` in this case are:

1. time
1. evid
1. compartment
1. dose amount
1. rate (for infusions)


```{r}
mod <- mcode("mevent3", code)

mod %>%  mrgsim(end = 240, delta = 0.2) %>% plot(CENT~time)
```



