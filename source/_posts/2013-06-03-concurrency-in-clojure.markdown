---
layout: post
title: "Concurrency in Clojure"
date: 2013-06-03 10:00
comments: true
categories: [clojure slides concurrency]
---

### Overview of Concurrency in Clojure ###

[Slides]( http://slid.es/doppioslash/concurrency-in-clojure )

#### The Old Way ###

Everyone has had to contend with handling a variable that is changed by different parts of a program at the same time.

Problems like Race Conditions and Deadlocks are handled using Locking, an error-prone technique, without digging deeper to reach the real culprit: mutable state.

Clojure doesn't take as radical a stance as Erlang, which has solved the problem by enforcing single assignment, it instead uses Software Transactional Memory.

#### The Clojure Way ####

##### Software Transactional Memory ######

Clojure applies functional programming staples of immutable data structures and avoidance of side effects and for the remaining state it uses Software Transactional Memory, which in practice is similar to how a database controls concurrent access: changes of state are wrapped in transactions.

Transactions ensure:

- Atomicity: either all changes of a transaction are applied or none.
- Consistency: only valid changes are committed.
- Isolation: no transaction sees the effect of other transactions.
- Durability: changes are persistent.

It provides a mechanism for managing references and updates across threads that is easier to use and less error-prone than lock-based concurrency.

##### Identity and state #####

Clojure has a unique philosophy about state, inspired by Alfred North Whitehead: it states that mutable state doesn't actually exist, but it's an illusion, in a similar way to how a sequence of still frames produces the illusion of movement if shown in a sequence fast enough to trick the eye.

So mutable state is actually a causally-linked sequence of immutable values and time is derived from the perception of these successions. We apply pure functions to immutable values to derive new immutable values, assigning identity to those sequences of values.

- Value: an immutable magnitude, quantity, number, or composite of these.
- Identity: a putative entity we associate with a series of causally related values (states) over time
- State: value of an identity at a moment in time
- Time: relative before/after ordering of causal values.

#### Immutability ####

Applying a function that modifies a data structure will return a new data structure rather than modifying the old one.
Values are immutable, they never change. 

{% codeblock Immutability - immutability.clj %}
=> (def a-map {:key "value"})
#'user/a-map    
=> (assoc a-map :another-key "more values")
{:key "value", :another-key "more values"}
=> a-map
{:key "value"}
{% endcodeblock %}

#### Reference Types ####

Clojure has 4 reference types: var, atom, ref and agent which represent identities.

All references contain some value, that can only be changed using the appropriate functions, which are different for every type.
Dereferencing will return the state of a reference at the time deref was invoked, which doesn't guarantee that it wont' be different at a later point.

Dereferencing doesn't block, reads need not be coordinated, it's just grabbing the latest value of an identity.


##### Vars #####

Vars support thread-local state.

Binding works on a per-thread basis by default. The first time a var is bound is called root binding and is seen by default by all threads, though they can only change it locally.

{% codeblock Vars - vars.clj %}
=> (def ^:dynamic a-var 42)
#'user/a-var
=> a-var
42
=> (def ^:dynamic a-var 43)
#'user/a-var
=> a-var
43
{% endcodeblock %}

Let's demonstrate that var is thread-local.

{% codeblock Vars - vars.clj %}
=> (defn print-var 
=>   ([] (println a-var))
=>   ([prefix] (println prefix a-var)))
#'user/print-var
=> (bindind [a-var 44]
=>   (print-var))
44
nil
=> (import java.lang.Thread)
java.lang.Thread
=> (defn with-spawned-thread [fun]
=>   (.start (java.lang.Thread. fun)))
#'user/with-spawned-thread
=> (do
=>   (binding [a-var "rebound a-var"]
=>     (with-spawned-thread
=>       (fn [] (print-var "bg: ")))
=>     (print-var "fg1: "))
=>   (print-var "fg2: "))
bg: 42
fg1:  rebound a-var
fg2: 42
nil
{% endcodeblock %}

The background thread also prints the root value, even though it's inside the context of the binding statement due to the fact that the spawned thread gets its own value of a-var (the root value) and doesn't see the rebound a-var in the spawning thread.

##### Refs #####

Used to change shared state synchronously coordinating changes, multiple refs can be updated in one trasaction.
Any mutation must be wrapped in dosync and more than one can be executed in the same transaction.

{% codeblock Refs - refs.clj %}
=> (def a-ref (ref 42))
#'user/a-ref
=> (ref-set a-ref 43)
IllegalStateException No transaction running  clojure.lang.LockingTransaction.getEx (LockingTransaction.java:208)
=> (dosync (ref-set a-ref 43))
43
=> (deref a-ref)
43
=> @a-ref
43
=> (dosync (alter a-ref inc))
44
{% endcodeblock %}

##### Atoms #####

Used to change indipendent state in an uncoordinated, atomic, synchronous manner. Changes are instantly visible from all threads.
State is mutated using a function, one of the benefits is that the operation can be retried safely.
{% codeblock Atoms - atoms.clj %}
=> (def an-atom (atom 42))
#'user/an-atom
=> @an-atom
42
=> (swap! an-atom inc)
43
=> @an-atom
43
=> (swap! an-atom (fn [old] 44))
44
=> (reset! an-atom 42)
42
=> @an-atom
42
{% endcodeblock %}

Add and update items to a collection inside an atom.

{% codeblock Atoms - atoms.clj %}
=> (def vector-atom (atom []))
#'user/vector-atom
=> (swap! vector-atom conj 42)
[42]
=> (swap! vector-atom conj 43)
[42 43]
=> (swap! vector-atom assoc-in [0] inc)
[43 43]
{% endcodeblock %}

##### Agents #####

They provide a thread-safe mechanism for asynchronous, uncoordinated updates.
Their state can be accessed from any thread.
Send and send-off can be used to mutate agents, the difference being that send-off uses a growing thread pool.

{% codeblock Agents - agents.clj %}
=> (def an-agent (agent 42))
#'user/an-agent
=> @an-agent
42
=> (send an-agent inc)
#<Agent@492f96c3: 43>
{% endcodeblock %}

If the updating function fails the agent will be in an error state and need to be restarted using restart-agent.

#### More ####

We haven't covered Delays, Futures and Promises.