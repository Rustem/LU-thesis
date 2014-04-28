## Introduction

A Log is an append only, totally-ordered by number sequence of records. The ordering of records defines a notion of "time". The log entry number called "timestamp" of the entry. The main problem logs solves in distributed env is recording what happened and when.

In complex systems with many databases and servers involved, logs utilized as an input to queries and graphs to understand behavior across many machines.


### Logs in databases

The purpose of logs in database is a source to restore the state after the crash. Databases ensures the atomicity and durability properties with logs (master-slave env, single-server env).


### Logs in distributed systems (DS)

The two problems a log solves in DS: ordering changes, distributing data. The approach is based on State Machine(SM) Replication Principle:

> if two identical, deterministic processes begin in the same state and get the same inputs in the same order, they will produce the same output and end in the same state.

Here important the term **deterministic** which means that certain action in the presence of particular inputs always produces the same output(s). The processing in deterministic env is time independent and doesn't let other "out of band" input influece its result.
