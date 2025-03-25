# Documentation

*Motoko - Basic:*
[Actor dan Actor Classes (untuk struktur canister)]
Actors
The programming model of the Internet Computer consists of memory-isolated canisters communicating by asynchronous message passing of binary data encoding Candid values. A canister processes its messages one-at-a-time, preventing race conditions. A canister uses call-backs to register what needs to be done with the result of any inter-canister messages it issues.

Motoko provides an actor-based programming model to developers to express services, including those of canister smart contracts on ICP. Each canister is represented as a typed actor. The type of an actor lists the messages it can handle. Each message is abstracted as a typed, asynchronous function. A translation from actor types to Candid types imposes structure on the raw binary data of the underlying Internet Computer. An actor is similar to an object, but is different in that its state is completely isolated, its interactions with the world are entirely through asynchronous messaging, and its messages are processed one-at-a-time, even when issued in parallel by concurrent actors.

Actors
An actor is similar to an object, but is different in that:

Its state is completely isolated.

Its interactions with the world are done entirely through asynchronous messaging.

Its messages are processed one-at-a-time, even when issued in parallel by concurrent actors.

All communication with and between actors involves passing messages asynchronously over the network using the Internet Computer’s messaging protocol. An actor’s messages are processed in sequence, so state modifications never admit race conditions, unless explicitly allowed by punctuating await expressions.

The Internet Computer ensures that each message that is sent receives a response. The response is either success with some value or an error. An error can be the explicit rejection of the message by the receiving canister, a trap due to an illegal instruction such as division by zero, or a system error due to distribution or resource constraints. For example, a system error might be the transient or permanent unavailability of the receiver (either because the receiving actor is oversubscribed or has been deleted).

In Motoko, actors have dedicated syntax and types:

Messaging is handled by so called shared functions returning futures. Shared functions are accessible to remote callers and have additional restrictions: their arguments and return value must be shared types. Shared types are a subset of types that includes immutable data, actor references, and shared function references, but excludes references to local functions and mutable data.

Future, f, is a value of the special type async T for some type T.

Waiting on f to be completed is expressed using await f to obtain a value of type T. To avoid introducing shared state through messaging, for example, by sending an object or mutable array, the data that can be transmitted through shared functions is restricted to immutable, shared types.

All state should be encapsulated within the actor or actor class. The main actor file should begin with imports, followed by the actor or actor class definition.

Defining an actor
Consider the following actor declaration:


persistent actor Counter {

  var count = 0;

  public shared func inc() : async () { count += 1 };

  public shared func read() : async Nat { count };

  public shared func bump() : async Nat {
    count += 1;
    count;
  };
};


The Counter actor declares one field and three public, shared functions:

The field count is mutable, initialized to zero and implicitly private.

Function inc() asynchronously increments the counter and returns a future of type async () for synchronization.

Function read() asynchronously reads the counter value and returns a future of type async Nat containing its value.

Function bump() asynchronously increments and reads the counter.

Shared functions, unlike local functions, are accessible to remote callers and have additional restrictions. Their arguments and return value must be shared type. Shared types are a subset of types that includes immutable data, actor references, and shared function references, but excludes references to local functions and mutable data. Because all interaction with actors is asynchronous, an actor’s functions must return futures, that is, types of the form async T, for some type T.

The only way to read or modify the state (count) of the Counter actor is through its shared functions.

A value of type async T is a future. The producer of the future completes the future when it returns a result, either a value or error.

Unlike objects and modules, actors can only expose functions, and these functions must be shared. For this reason, Motoko allows you to omit the shared modifier on public actor functions, allowing the more concise, but equivalent, actor declaration:


persistent actor Counter {

  stable var count = 0;

  public func inc() : async () { count += 1 };

  public func read() : async Nat { count };

  public func bump() : async Nat {
    count += 1;
    count;
  };
};


For now, the only place shared functions can be declared is in the body of an actor or actor class. Despite this restriction, shared functions are still first-class values in Motoko and can be passed as arguments or results, and stored in data structures.

The type of a shared function is specified using a shared function type. For example, the value inc has type shared () → async Nat and could be supplied as a standalone callback to some other service.

Actor types
Just as objects have object types, actors have actor types. The Counter example above has the following type:

actor {
  inc  : shared () -> async ();
  read : shared () -> async Nat;
  bump : shared () -> async Nat;
}

Unlike objects and modules, actors can only expose functions, and these functions must be shared. For now, the only place shared functions can be declared is in the body of an actor or actor class. Despite this restriction, shared functions are still first-class values in Motoko and can be passed as arguments or results, and stored in data structures.

The shared modifier is required on every member of an actor. Motoko both elides them on display and allows you to omit them when authoring an actor type.

Thus, the previous type can be expressed more succinctly as:

actor {
  inc  : () -> async ();
  read : () -> async Nat;
  bump : () -> async Nat;
}

Like object types, actor types support subtyping: an actor type is a subtype of a more general one that offers fewer functions with more general types.

Asynchronous behavior
Like other modern programming languages, Motoko permits an ergonomic syntax for asynchronous communication among components.

In the case of Motoko, each communicating component is an actor. As an example of using actors, consider this three-line program:

let result1 = service1.computeAnswer(params);
let result2 = service2.computeAnswer(params);
finalStep(await result1, await result2)


This program’s behavior can be summarized as:

The program makes two requests (lines 1 and 2) to two distinct services, each implemented as a Motoko actor or canister smart contract implemented in some other language.

The program waits for each result to be ready (line 3) using the keyword await on each result value.

The program uses both results in the final step (line 3) by calling the finalStep function.

The services interleave their executions rather than wait for one another, since this reduces overall latency. If you try to reduce latency this way without special language support, such interleaving will quickly sacrifice clarity and simplicity.

Even in cases where there are no interleaving executions, for example, if there were only one call above, not two, the programming abstractions still permit clarity and simplicity for the same reason. Namely, they signal to the compiler where to transform the program, freeing the programmer from contorting the program’s logic in order to interleave its execution with the underlying system’s message-passing loop.

In the above example, the program uses await in line 3 to express that interleaving behavior in a simple fashion.

In other programming languages that lack these abstractions, developers would not merely call these two functions directly, but would instead employ very advanced programming patterns, possibly registering developer-provided “callback functions” within system-provided “event handlers”. Each callback would handle an asynchronous event that arises when an answer is ready. This kind of systems-level programming is powerful, but very error-prone, since it decomposes a high-level data flow into low-level system events that communicate through shared state.

Traps and commit points
A trap is a non-recoverable runtime failure caused by errors such as division-by-zero, out-of-bounds array indexing, numeric overflow, cycle exhaustion or assertion failure.

A shared function call that executes without executing an await expression never suspends and executes atomically. A shared function that contains no await expression is syntactically atomic.

Commit points
An atomic shared function whose execution traps has no visible effect on the state of the enclosing actor or its environment - any state change is reverted, and any message that it has sent is revoked. In fact, all state changes and message sends are tentative during execution: they are committed only after a successful commit point is reached.

The points at which tentative state changes and message sends are irrevocably committed are:

Implicit exit from a shared function by producing a result.

Explicit exit via return or throw expressions.

Explicit await expressions.

Traps
A trap will only revoke changes made since the last commit point. In particular, in a non-atomic function that does multiple awaits, a trap will only revoke changes attempted since the last await - all preceding effects will have been committed and cannot be undone.

Consider the following stateful Atomicity actor:

actor Atomicity {

  var s = 0;
  var pinged = false;

  public func ping() : async () {
    pinged := true;
  };

  // an atomic method
  public func atomic() : async () {
    s := 1;
    ignore ping();
    ignore 0/0; // trap!
  };

  // a non-atomic method
  public func nonAtomic() : async () {
    s := 1;
    let f = ping(); // this will not be rolled back!
    s := 2;
    await f;
    s := 3; // this will not be rolled back!
    await f;
    ignore 0/0; // trap!
  };

};


Calling the shared function atomic() will fail with an error, since the last statement causes a trap. However, the trap leaves the mutable variable s with value 0, not 1, and variable pinged with value false, not true. This is because the trap happens before the method atomic has executed an await, or exited with a result. Even though atomic calls ping(), ping() is queued until the next commit point.

Calling the shared function nonAtomic() will also fail with an error due to a trap. In this function, the trap leaves the variable s with value 3, not 0, and variable pinged with value true, not false. This is because each await commits its preceding side-effects, including message sends. Even though f is complete by the second await, this await also forces a commit of the state, suspends execution and allows for interleaved processing of other messages to this actor.

Actor classes
Actor classes enable you to create networks of actors programmatically. Actor classes have to be defined in a separate source file. To illustrate how to define and import actor classes, the following example implements a distributed map of keys of type Nat to values of type Text. It provides simple insert and lookup functions, put(k, v) and get(k), for working with these keys and values.

Defining an actor class
To distribute the data for this example, the set of keys is partitioned into n buckets. For now, we just fix n = 8. The bucket, i, of a key, k, is determined by the remainder of k divided by n, that is, i = k % n. The ith bucket (i in [0..n)) receives a dedicated actor to store text values assigned to keys in that bucket.

The actor responsible for bucket i is obtained as an instance of the actor class Bucket(i), defined in the sample Buckets.mo file, as follows:

Buckets.mo:

import Nat "mo:base/Nat";
import Map "mo:base/OrderedMap";

persistent actor class Bucket(n : Nat, i : Nat) {

  type Key = Nat;
  type Value = Text;

  transient let keyMap = Map.Make<Key>(Nat.compare);

  var map : Map.Map<Key, Value> = keyMap.empty();

  public func get(k : Key) : async ?Value {
    assert((k % n) == i);
    keyMap.get(map, k);
  };

  public func put(k : Key, v : Value) : async () {
    assert((k % n) == i);
    map := keyMap.put(map, k, v);
  };

};


A bucket stores the current mapping of keys to values in a mutable map variable containing an imperative RedBlack tree, map, that is initially empty.

On get(k), the bucket actor simply returns any value stored at k, returning map.get(k).

On put(k, v), the bucket actor updates the current map to map k to ?v by calling map.put(k, v).

Both functions use the class parameters n and i to verify that the key is appropriate for the bucket by asserting ((k % n) == i).

Clients of the map can then communicate with a coordinating Map actor, implemented as follows:

import Array "mo:base/Array";
import Buckets "Buckets";

persistent actor Map {

  let n = 8; // number of buckets

  type Key = Nat;
  type Value = Text;

  type Bucket = Buckets.Bucket;

  let buckets : [var ?Bucket] = Array.init(n, null);

  public func get(k : Key) : async ?Value {
    switch (buckets[k % n]) {
      case null null;
      case (?bucket) await bucket.get(k);
    };
  };

  public func put(k : Key, v : Value) : async () {
    let i = k % n;
    let bucket = switch (buckets[i]) {
      case null {
        let b = await Buckets.Bucket(n, i); // dynamically install a new Bucket
        buckets[i] := ?b;
        b;
      };
      case (?bucket) bucket;
    };
    await bucket.put(k, v);
  };

};


As this example illustrates, the Map code imports the Bucket actor class as module Buckets.

The actor maintains an array of n allocated buckets, with all entries initially null. Entries are populated with Bucket actors on demand.

On get(k, v), the Map actor:

Uses the remainder of key k divided by n to determine the index i of the bucket responsible for that key.

Returns null if the ith bucket does not exist, or

Delegates to that bucket by calling bucket.get(k, v) if it does.

On put(k, v), the Map actor:

Uses the remainder of key k divided by n to determine the index i of the bucket responsible for that key.

Installs bucket i if the bucket does not exist by using an asynchronous call to the constructor, Buckets.Bucket(i), and, after awaiting the result, records it in the array buckets.

Delegates the insertion to that bucket by calling bucket.put(k, v).

While this example sets the number of buckets to 8, you can generalize the example by making the Map actor an actor class, adding a parameter (n : Nat) and omitting the declaration let n = 8;.

For example:

actor class Map(n : Nat) {

  type Key = Nat
  ...
}

Clients of actor class Map are now free to determine the maximum number of buckets in the network by passing an argument on construction.

On ICP, calls to a class constructor must be provisioned with cycles to pay for the creation of a principal. See ExperimentalCycles for instructions on how to add cycles to a call using the imperative ExperimentalCycles.add<system>(cycles) function.

Configuring and managing actor class instances
On ICP, the primary constructor of an imported actor class always creates a new principal and installs a fresh instance of the class as the code for that principal.

To provide further control over actor class installation, Motoko endows each imported actor class with an extra, secondary constructor. This constructor takes an additional first argument that specifies the desired installation mode. The constructor is only available via special syntax that stresses its system functionality.

Using this syntax, it's possible to specify initial canister settings (such as an array of controllers), manually install, upgrade and reinstall canisters, exposing all of the lower-level facilities of the Internet Computer.


[Caller Identification (untuk autentikasi)]

Caller identification
Motoko’s shared functions support a simple form of caller identification that allows you to inspect the ICP principal associated with the caller of a function. Principals are a value that identifies a unique user or canister.

You can use the principal associated with the caller of a function to implement a basic form of access control in your program.

Using caller identification
In Motoko, the shared keyword is used to declare a shared function. The shared function can also declare an optional parameter of type {caller : Principal}.

To illustrate how to access the caller of a shared function, consider the following:


shared(msg) func inc() : async () {
  // ... msg.caller ...
}

In this example, the shared function inc() specifies a msg parameter, a record, and the msg.caller accesses the principal field of msg.

The calls to the inc() function do not change. At each call site, the caller’s principal is provided by the system, not the user. The principal cannot be forged or spoofed by a malicious user.

To access the caller of an actor class constructor, you use the same syntax on the actor class declaration. For example:


shared(msg) persistent actor class Counter(init : Nat) {
  // ... msg.caller ...
}


Adding access control
To extend this example, assume you want to restrict the Counter actor so it can only be modified by the installer of the Counter. To do this, you can record the principal that installed the actor by binding it to an owner variable. You can then check that the caller of each method is equal to owner like this:


shared(msg) persistent actor class Counter(init : Nat) {

  transient let owner = msg.caller;

  var count = init;

  public shared(msg) func inc() : async () {
    assert (owner == msg.caller);
    count += 1;
  };

  public func read() : async Nat {
    count
  };

  public shared(msg) func bump() : async Nat {
    assert (owner == msg.caller);
    count := 1;
    count;
  };
}


In this example, the assert (owner == msg.caller) expression causes the functions inc() and bump() to trap if the call is unauthorized, preventing any modification of the count variable while the read() function permits any caller.

The argument to shared is just a pattern. You can rewrite the above to use pattern matching:


shared({caller = owner}) persistent actor class Counter(init : Nat) {

  var count : Nat = init;

  public shared({caller}) func inc() : async () {
    assert (owner == caller);
    count += 1;
  };

  // ...
}


Simple actor declarations do not let you access their installer. If you need access to the installer of an actor, rewrite the actor declaration as a zero-argument actor class instead.

Recording principals
Principals support equality, ordering, and hashing, so you can efficiently store principals in containers for functions such as maintaining an allow or deny list. More operations on principals are available in the principal base library.

The data type of Principal in Motoko is both sharable and stable, meaning you can compare Principals for equality directly.

Below is an example of how you can record principals in a set.


import Principal "mo:base/Principal";
import OrderedSet "mo:base/OrderedSet";
import Error "mo:base/Error";

persistent actor {

    transient let principalSet = OrderedSet.Make<Principal>(Principal.compare);

    // Create set to record principals
    var principals : OrderedSet.Set<Principal> = principalSet.empty();

    // Check if principal is recorded
    public shared query(msg) func isRecorded() : async Bool {
        let caller = msg.caller;
        principalSet.contains(principals, caller);
    };

    // Record a new principal
    public shared(msg) func recordPrincipal() : async () {
        let caller = msg.caller;
        if (Principal.isAnonymous(caller)) {
            throw Error.reject("Anonymous principal not allowed");
        };

        principals := principalSet.put(principals, caller)
    };
};



import Principal "mo:base/Principal";
import OrderedSet "mo:base/OrderedSet";
import Error "mo:base/Error";

persistent actor {

    // Create set to store principals
    transient var principalSet = Set.Make(Principal.compare);

    var principals : OrderedSet.Set<Principal> = principalSet.empty();

    // Check if principal is recorded
    public shared query(msg) func isRecorded() : async Bool {
        let caller = msg.caller;
        principleSet.contains(principals, caller);
    };

    // Record a new principal
    public shared(msg) func recordPrincipal() : async () {
        let caller = msg.caller;
        if (Principal.isAnonymous(caller)) {
            throw Error.reject("Anonymous principal not allowed");
        };

        principals := principalSet.put(principals, caller)
    };
};


[Mutable State (untuk penyimpanan data)]
Mutable state
Each actor in Motoko may use, but may never directly share, internal mutable state.

Immutable data can be shared among actors, and also handled through each other's external entry points which serve as shareable functions. Unlike shareable data, a key Motoko design invariant is that mutable data is kept private to the actor that allocates it and is never shared remotely.

Immutable vs mutable variables
The var syntax declares mutable variables in a declaration block:


let text  : Text = "abc";
let num  : Nat = 30;

var pair : (Text, Nat) = (text, num);
var text2 : Text = text;

The declaration list above declares four variables. The first two variables (text and num) are lexically-scoped, immutable variables. The final two variables (pair and text2) are lexically-scoped, mutable variables.

Assignment to mutable memory
Mutable variables permit assignment and immutable variables do not.

If you try to assign new values to either Text or num above, you will get static type errors because these variables are immutable.

You may freely update the value of mutable variables pair and text2 using the syntax for assignment, written as :=, as follows:

text2 := text2 # "xyz";
pair := (text2, pair.1);
pair

In the example above, each variable is updated based on applying a simple update rule to their current values. Likewise, an actor processes some calls by performing updates on its private mutable variables, using the same assignment syntax as above.

Special assignment operations
The assignment operation := is general and works for all types.

Motoko includes special assignment operations that combine assignment with a binary operation. The assigned value uses the binary operation on a given operand and the current contents of the assigned variable.

For example, numbers permit a combination of assignment and addition:


var num2 = 2;
num2 += 40;
num2

After the second line, the variable num2 holds 42, as one would expect.

Motoko includes other combinations as well. For example, we can rewrite the line above that updates text2 more concisely as:

text2 #= "xyz";
text2

As with +=, this combined form avoids repeating the assigned variable’s name on the right hand side of the special assignment operator #=.

The full table of assignment operators lists numerical, logical, and textual operations over appropriate types number, boolean and text values, respectively.

Reading from mutable memory
Once you have updated each variable, you must read from the mutable contents. This does not require a special syntax.

Each use of a mutable variable looks like the use of an immutable variable, but does not act like one. In fact, its meaning is more complex. As in many other language, the syntax of each use hides the memory effect that accesses the memory cell identified by that variable and gets its current value. Other languages from functional traditions generally expose these effects syntactically.

var vs let bound variables
Consider the following two variable declarations, which look similar:


let x : Nat = 0


var x : Nat = 0

The only difference in their syntax is the use of keyword let versus var to define the variable x, which in each case the program initializes to 0.

However, these programs carry different meanings, and in the context of larger programs, the difference in meanings will impact the meaning of each occurrence of x.

For the first program, which uses let, each such occurrence means 0. Replacing each occurrence with 0 will not change the meaning of the program.

For the second program, which uses var, each occurrence means “read and produce the current value of the mutable memory cell named x.” In this case, each occurrence’s value is determined by the dynamic state of the contents of the mutable memory cell named x.

As one can see from the definitions above, there is a fundamental contrast between the meanings of let-bound and var-bound variables.

In large programs, both kinds of variables can be useful, and neither kind serves as a good replacement for the other. However, let-bound variables are more fundamental.

For instance, instead of declaring x as a mutable variable initially holding 0, you could instead use y, an immutable variable that denotes a mutable array with one entry holding 0:


var x : Nat       = 0 ;
let y : [var Nat] = [var 0] ;

The read and write syntax required for this encoding reuses that of mutable arrays, which is not as readable as that of var-bound variables. As such, the reads and writes of variable x will be easier to read than those of variable y.

For this practical reason and others, var-bound variables are a core aspect of the language's design.

Immutable arrays
Before discussing mutable arrays, we introduce immutable arrays, which share the same projection syntax but do not permit mutable updates after allocation.

Allocate an immutable array of constants

let a : [Nat] = [1, 2, 3] ;

The array a above holds three natural numbers, and has type [Nat]. In general, the type of an immutable array is [_], using square brackets around the type of the array’s elements, which must share a single common type.

Read from an array index
You can read from an array using the usual bracket syntax of [ and ] around the index you want to access:

let x : Nat = a[2] + a[0] ;

Every array access in Motoko is safe. Accesses that are out of bounds will not access memory unsafely, but instead will cause the program to trap as with an assertion failure.

The Array module
The Motoko standard library provides basic operations for immutable and mutable arrays. It can be imported as follows:

import Array "mo:base/Array";

For more information about using arrays, see the array library descriptions.

Allocate an immutable array with varying content
Each new array allocated by a program will contain a varying number of elements. Without mutation, you need a way to specify this family of elements all at once in the argument to allocation.

To accommodate this need, the Motoko language provides the higher-order array allocation function Array.tabulate, which allocates a new array by consulting a user-provided generation function, gen,for each element.

func tabulate<T>(size : Nat,  gen : Nat -> T) : [T]


Function gen specifies the array as a function value of arrow type Nat → T, where T is the final array element type.

The function gen actually functions as the array during its initialization. It receives the index of the array element and produces the element of type T that should reside at that index in the array. The allocated output array populates itself based on this specification.

For instance, you can first allocate array1 consisting of some initial constants, then functionally update some of the indices by changing them in a pure, functional way, to produce array2, a second array that does not destroy the first.

let array1 : [Nat] = [1, 2, 3, 4, 6, 7, 8] ;

let array2 : [Nat] = Array.tabulate<Nat>(7, func(i:Nat) : Nat {
    if ( i == 2 or i == 5 ) { array1[i] * i } // change 3rd and 6th entries
    else { array1[i] } // no change to other entries
  }) ;


Even though we changed array1 into array2 in a functional sense, notice that both arrays and both variables are immutable.

Mutable arrays
Each mutable array in Motoko introduces private mutable actor state.

Because Motoko’s type system enforces that remote actors do not share their mutable state, the Motoko type system introduces a firm distinction between mutable and immutable arrays that impacts typing, subtyping, and the language abstractions for asynchronous communication.

Locally, the mutable arrays can not be used in places that expect immutable ones, since Motoko’s definition of subtyping for arrays correctly distinguishes those cases for the purposes of type soundness. Additionally, in terms of actor communication, immutable arrays are safe to send and share, while mutable arrays can not be shared or otherwise sent in messages. Unlike immutable arrays, mutable arrays have non-shareable types.

Allocate a mutable array of constants
To indicate allocation of mutable arrays, the mutable array syntax [var _] uses the var keyword in both the expression and type forms:


let a : [var Nat] = [var 1, 2, 3] ;

As above, the array a above holds three natural numbers, but has type [var Nat].

Allocate a mutable array with dynamic size
To allocate mutable arrays of non-constant size, use the Array.init base library function and supply an initial value:

func init<T>(size : Nat,  x : T) : [var T]

For example:

var size : Nat = 42 ;
let x : [var Nat] = Array.init<Nat>(size, 3);


The variable size does not need to be constant here. The array will have size number of entries, each holding the initial value 3.

Mutable updates
Mutable arrays, each with type form [var _], permit mutable updates via assignment to an individual element. In this case, element index 2 gets updated from holding 3 to instead hold value 42:


let a : [var Nat] = [var 1, 2, 3];
a[2] := 42;
a

Subtyping does not permit mutable to be used as immutable
Subtyping in Motoko does not permit us to use a mutable array of type [var Nat] in places that expect an immutable one of type [Nat].

There are two reasons for this. First, as with all mutable state, mutable arrays require different rules for sound subtyping. In particular, mutable arrays have a less flexible subtyping definition, necessarily. Second, Motoko forbids uses of mutable arrays across asynchronous communication, where mutable state is never shared.


[Query Functions (untuk pengambilan data)]
Query functions
In ICP terminology, update messages, also referred to as calls, can alter the state of the canister when called. Effecting a state change requires agreement amongst the distributed replicas before the network can commit the change and return a result. Reaching consensus is an expensive process with relatively high latency.

For the parts of applications that don’t require the guarantees of consensus, the ICP supports more efficient query operations. These are able to read the state of a canister from a single replica, modify a snapshot during their execution and return a result, but cannot permanently alter the state or send further messages.

Query functions
Motoko supports the implementation of queries using query functions. The query keyword modifies the declaration of a shared actor function so that it executes with non-committing and faster query semantics.

For example, consider the following Counter actor with a read function called peek:


persistent actor Counter {

  var count = 0;

  // ...

  public shared query func peek() : async Nat {
    count
  };

}


The peek() function might be used by a Counter frontend offering a quick, but less trustworthy, display of the current counter value.

Query functions can be called from non-query functions. Because those nested calls require consensus, the efficiency gains of nested query calls will be modest at best.

The query modifier is reflected in the type of a query function:

  peek : shared query () -> async Nat

As before, in query declarations and actor types the shared keyword can be omitted.

A query method cannot call an actor function and will result in an error when the code is compiled. Calls to ordinary functions are permitted.

Composite query functions
Queries are limited in what they can do. In particular, they cannot themselves issue further messages, including queries.

To address this limitation, the ICP supports another type of query function called a composite query.

Like plain queries, the state changes made by a composite query are transient, isolated and never committed. Moreover, composite queries cannot call update functions, including those implicit in async expressions, which require update calls under the hood.

Unlike plain queries, composite queries can call query functions and composite query functions on the same and other actors, but only provided those actors reside on the same subnet.

As a contrived example, consider generalizing the previous Counter actor to a class of counters. Each instance of the class provides an additional composite query to sum the values of a given array of counters:


persistent actor class Counter () {

  var count = 0;

  // ...

  public shared query func peek() : async Nat {
    count
  };

  public shared composite query func sum(counters : [Counter]) : async Nat {
    var sum = 0;
    for (counter in counters.vals())  {
      sum += await counter.peek();
    };
    sum
  }

}


Declaring sum as a composite query enables it call the peek queries of its argument counters.

While update messages can call plain query functions, they cannot call composite query functions. This distinction, which is dictated by the current capabilities of ICP, explains why query functions and composite query functions are regarded as distinct types of shared functions.

Note that the composite query modifier is reflected in the type of a composite query function:

  sum : shared composite query ([Counter]) -> async Nat


Since only a composite query can call another composite query, you may be wondering how any composite query gets called at all?

Composite queries are initiated outside ICP, typically by an application (such as a browser frontend) sending an ingress message invoking a composite query on a backend actor.

The Internet Computer's semantics of composite queries ensures that state changes made by a composite query are isolated from other inter-canister calls, including recursive queries, to the same actor.

In particular, a composite query call rolls back its state on function exit, but is also does not pass state changes to sub-query or sub-composite-query calls. Repeated calls, which include recursive calls, have different semantics from calls that accumulate state changes.

In sequential calls, the internal state changes of preceding queries will have no effect on subsequent queries, nor will the queries observe any local state changes made by the enclosing composite query. Local states changes made by the composite query are preserved across the calls until finally being rolled-back on exit from the composite query.

This semantics can lead to surprising behavior for users accustomed to ordinary imperative programming.

Consider this example containing the composite query test that calls query q and composite query cq.

persistent actor Composites {

  var state = 0;

  // ...

  public shared query func q() : async Nat {
    let s = state;
    state += 10;
    s
  };

  public shared composite query func cq() : async Nat {
    let s = state;
    state += 100;
    s
  };

  public shared composite query func test() :
    async {s0 : Nat; s1 : Nat; s2 : Nat; s3 : Nat } {
    let s0 = state;
    state += 1000;
    let s1 = await q();
    state += 1000;
    let s2 = await cq();
    state += 1000;
    let s3 = state;
    {s0; s1; s2; s3}
  };

}


When state is 0, a call to test returns

{s0 = 0; s1 = 0; s2 = 0; s3 = 3_000}

This is because none of the local updates to state are visible to any of the callers or callees.

*Motoko - Advanced*

[Stable Variables (untuk persistensi data)]
Stable variables and upgrade methods
One key feature of Motoko is its ability to automatically persist the program's state without explicit user instruction, called orthogonal persistence. This not only covers persistence across transactions but also includes canister upgrades. For this purpose, Motoko features a bespoke compiler and runtime system that manages upgrades in a sophisticated way such that a new program version can pick up the state left behind by a previous program version. As a result, Motoko data persistence is not simple but also prevents data corruption or loss, while being efficient at the same time. No database, stable memory API, or stable data structure is required to retain state across upgrades. Instead, a simple stable keyword is sufficient to declare an data structure of arbitrary shape persistent, even if the structure uses sharing, has a deep complexity, or contains cycles.

This is substantially different to other languages supported on the IC, which use off-the-shelf language implementations that are not designed for orthogonal persistence in mind: They rearrange memory structures in an uncontrolled manner on re-compilation or at runtime. As an alternative, in other languages, programmers have to explicitly use stable memory or special stable data structures to rescue their data between upgrades. Contrary to Motoko, this approach is not only cumbersome, but also unsafe and inefficient. Compared to using stable data structures, Motoko's orthogonal persistence allows more natural data modeling and significantly faster data access, eventually resulting in more efficient programs.

Declaring stable variables
In an actor, you can configure which part of the program is considered to be persistent, i.e. survives upgrades, and which part are ephemeral, i.e. are reset on upgrades.

More precisely, each let and var variable declaration in an actor can specify whether the variable is stable or transient. If you don’t provide a modifier, the variable is assumed to be transient by default.

The semantics of the modifiers is as follows:

stable means that all values directly or indirectly reachable from that stable actor variable are considered persistent and automatically retained across upgrades. This is the primary choice for most of the program's state.
transient means that the variable is re-initialized on upgrade, such that the values referenced by this transient variable can be discarded, unless the values are transitively reachable by other variables that are stable. transient is only used for temporary state or references to high-order types, such as local function references, see stable types.
Previous versions of Motoko (up to version 0.13.4) used the keyword flexible instead of transient. Both keywords are accepted interchangeably but the legacy flexible keyword may be deprecated in the future.

The following is a simple example of how to declare a stable counter that can be upgraded while preserving the counter’s value:


actor Counter {

  stable var value = 0;

  public func inc() : async Nat {
    value += 1;
    return value;
  };
}

Starting with Motoko v0.13.5, if you prefix the actor keyword with the keyword persistent, then all let and var declarations of the actor or actor class are implicitly declared stable. Only transient variables will need an explicit transient declaration. Using a persistent actor can help avoid unintended data loss. It is the recommended declaration syntax for actors and actor classes. The non-persistent declaration is provided for backwards compatibility.

Since Motoko v0.13.5, the recommended way to declare StableCounter above is:


persistent actor Counter {

  var value = 0; // implicitly stable!

  public func inc() : async Nat {
    value += 1;
    value;
  };
}

You can only use the stable, transient (or legacy flexible) modifier on let and var declarations that are actor fields. You cannot use these modifiers anywhere else in your program.

When you first compile and deploy a canister, all transient and stable variables in the actor are initialized in sequence. When you deploy a canister using the upgrade mode, all stable variables that existed in the previous version of the actor are pre-initialized with their old values. After the stable variables are initialized with their previous values, the remaining transient and newly-added stable variables are initialized in sequence.

Do not forget to declare variables stable if they should survive canister upgrades as the default is transient if no modifier is declared. A simple precaution is declare the entire actor or actor class persistent.

Persistence modes
Motoko currently features two implementations for orthogonal persistence, see persistence modes.

Stable types
Because the compiler must ensure that stable variables are both compatible with and meaningful in the replacement program after an upgrade, every stable variable must have a stable type. A type is stable if the type obtained by ignoring any var modifiers within it is shared.

The only difference between stable types and shared types is the former’s support for mutation. Like shared types, stable types are restricted to first-order data, excluding local functions and structures built from local functions (such as class instances). This exclusion of functions is required because the meaning of a function value, consisting of both data and code, cannot easily be preserved across an upgrade. The meaning of plain data, mutable or not, can be.

In general, classes are not stable because they can contain local functions. However, a plain record of stable data is a special case of object types that are stable. Moreover, references to actors and shared functions are also stable, allowing you to preserve their values across upgrades. For example, you can preserve the state record of a set of actors or shared function callbacks subscribing to a service.

Converting non-stable types into stable types
For variables that do not have a stable type, there are two options for making them stable:

Use a stable module for the type, such as:
StableBuffer
StableHashMap
StableRBTree
Unlike stable data structures in the Rust CDK, these modules do not use stable memory but rely on orthogonal persistence. The adjective "stable" only denotes a stable type in Motoko.

Extract the state in a stable type, and wrap it in the non-stable type.
For example, the stable type TemperatureSeries covers the persistent data, while the non-stable type Weather wraps this with additional methods (local function types).

persistent actor {
  type TemperatureSeries = [Float];

  class Weather(temperatures : TemperatureSeries) {
    public func averageTemperature() : Float {
      var sum = 0.0;
      var count = 0.0;
      for (value in temperatures.vals()) {
        sum += value;
        count += 1;
      };
      return sum / count;
    };
  };

  var temperatures : TemperatureSeries = [30.0, 31.5, 29.2];
  transient var weather = Weather(temperatures);
};


Discouraged and not recommended: Pre- and post-upgrade hooks allow copying non-stable types to stable types during upgrades. This approach is error-prone and does not scale for large data. Per best practices, using these methods should be avoided if possible. Conceptually, it also does not align well with the idea of orthogonal persistence.
Stable type signatures
The collection of stable variable declarations in an actor can be summarized in a stable signature.

The textual representation of an actor’s stable signature resembles the internals of a Motoko actor type:

actor {
  stable x : Nat;
  stable var y : Int;
  stable z : [var Nat];
};

It specifies the names, types and mutability of the actor’s stable fields, possibly preceded by relevant Motoko type declarations.

You can emit the stable signature of the main actor or actor class to a .most file using moc compiler option --stable-types. You should never need to author your own .most file.

A stable signature <stab-sig1> is stable-compatible with signature <stab-sig2>, if for each stable field <id> : T in <stab-sig1> one of the following conditions hold:

<stab-sig2> does not contain a stable field <id>.
<stab-sig> has a matching stable field <id> : U with T <: U.
Note that <stab-sig2> may contain additional fields or abandon fields of <stab-sig1>. Mutability can be different for matching fields.

<stab-sig1> is the signature of an older version while <stab-sig2> is the signature of a newer version.

The subtyping condition on stable fields ensures that the final value of some field can be consumed as the initial value of that field in the upgraded code.

You can check the stable-compatibility of two .most files containing stable signatures, using moc compiler option --stable-compatible file1.most file2.most.

Upgrade safety
When upgrading a canister, it is important to verify that the upgrade can proceed without:

Introducing an incompatible change in stable declarations.
Breaking clients due to a Candid interface change.
With enhanced orthogonal persistence, Motoko rejects incompatible changes of stable declarations during upgrade attempt. Moreover, dfx checks the two conditions before attempting the upgrade and warns users correspondingly.

A Motoko canister upgrade is safe provided:

The canister’s Candid interface evolves to a Candid subtype.
The canister’s Motoko stable signature evolves to a stable-compatible one.
With classical orthogonal persistence, the upgrade can still fail due to resource constraints. This is problematic as the canister can then not be upgraded. It is therefore strongly advised to test the scalability of upgrades well. Enhanced orthogonal persistence will abandon this issue.

You can check valid Candid subtyping between two services described in .did files using the didc tool with argument check file1.did file2.did.

Upgrading a deployed actor or canister
After you have deployed a Motoko actor with the appropriate stable variables, you can use the dfx deploy command to upgrade an already deployed version. For information about upgrading a deployed canister, see upgrade a canister smart contract.

dfx deploy checks that the interface is compatible, and if not, shows this message and asks if you want to continue:

You are making a BREAKING change. Other canisters or frontend clients relying on your canister may stop working.


In addition, Motoko with enhanced orthogonal persistence implements extra safe guard in the runtime system to ensure that the stable data is compatible, to exclude any data corruption or misinterpretation. Moreover, dfx also warns about dropping stable variables.

Data migration
Often, data representation changes with a new program version. For orthogonal persistence, it is important the language is able to allow flexible data migration to the new version.

Motoko supports two kinds of data migrations: Implicit migration and explicit migration.

Implicit migration
This is automatically supported when the new program version is stable-compatible with the old version. The runtime system of Motoko then automatically handles the migration on upgrade.

More precisely, the following changes can be implicitly migrated:

Adding or removing actor fields.
Changing mutability of the actor field.
Removing record fields.
Adding variant fields.
Changing Nat to Int.
Shared function parameter contravariance and return type covariance.
Any change that is allowed by the Motoko's subtyping rule.
Explicit migration
Any more complex migration is possible by user-defined functionality.

For this purpose, a three step approach is taken:

Introduce new variables of the desired types, while keeping the old declarations.
Write logic to copy the state from the old variables to the new variables on upgrade.
Drop the old declarations once all data has been migrated.
For more information, see the example of explicit migration.

Legacy features
The following aspects are retained for historical reasons and backwards compatibility:

Pre-upgrade and post-upgrade system methods
Using the pre- and post-upgrade system methods is discouraged. It is error-prone and can render a canister unusable. In particular, if a preupgrade method traps and cannot be prevented from trapping by other means, then your canister may be left in a state in which it can no longer be upgraded. Per best practices, using these methods should be avoided if possible.

Motoko supports user-defined upgrade hooks that run immediately before and after an upgrade. These upgrade hooks allow triggering additional logic on upgrade. These hooks are declared as system functions with special names, preugrade and postupgrade. Both functions must have type : () → ().

If preupgrade raises a trap, hits the instruction limit, or hits another IC computing limit, the upgrade can no longer succeed and the canister is stuck with the existing version.

postupgrade is not needed, as the equal effect can be achieved by introducing initializing expressions in the actor, e.g. non-stable let expressions or expression statements.

Stable memory and stable regions
Stable memory was introduced on the IC to allow upgrades in languages that do not implement orthogonal persistence of the main memory. This is the case with Motoko's classical persistence as well as other languages besides Motoko.

Stable memory and stable regions can still be used in combination with orthogonal persistence, although there is little practical need for this with enhanced orthogonal persistence and the future large main memory capacity on the IC.

[Inter-canister calls (jika diperlukan)]
Inter-canister calls
One of the most important features of ICP for developers is the ability to call functions in one canister from another canister. This capability to make calls between canisters, also sometimes referred to as inter-canister calls, enables you to reuse and share functionality in multiple dapps.

For example, you might want to create a dapp for professional networking, organizing community events, or hosting fundraising activities. Each of these dapps might have a social component that enables users to identify social relationships based on some criteria or shared interest, such as friends and family or current and former colleagues.

To address this social component, you might want to create a single canister for storing user relationships then write your professional networking, community organizing, or fundraising application to import and call functions that are defined in the canister for social connections. You could then build additional applications to use the social connections canister or extend the features provided by the social connections canister to make it useful to an even broader community of other developers.

This example will showcase a simple way to configure inter-canister calls that can be used as the foundation for more elaborate projects and use-cases such as those mentioned above.

Example
Consider the following code for Canister1:

import Canister2 "canister:canister2";

persistent actor Canister1 {

  public func main() : async Nat {
    return await Canister2.getValue();
  };

};

Then, consider the following code for Canister2:


import Debug "mo:base/Debug";

persistent actor Canister2 {
  public func getValue() : async Nat {
    Debug.print("Hello from canister 2!");
    return 10;
  };
};

To make an inter-canister call from canister1 to canister2, you can use the dfx command:

dfx canister call canister1 main

The output should resemble the following:

2023-06-15 15:53:39.567801 UTC: [Canister ajuq4-ruaaa-aaaaa-qaaga-cai] Hello from canister 2!
(10 : nat)


Alternatively, you can also use a canister id to access a previously deployed canister by using the following piece of code in canister1:


persistent actor {

  public func main(canisterId: Text) : async Nat {
    let canister2 = actor(canisterId): actor { getValue: () -> async Nat };
    return await canister2.getValue();
  };

};


Then, use the following call, replacing canisterID with the principal ID of a previously deployed canister:

dfx canister call canister1 main "canisterID"


Advanced usage
If the method name or input types are unknown at compile time, it's possible to call arbitrary canister methods using the ExperimentalInternetComputer module.

Here is an example which you can modify for your specific use case:


import IC "mo:base/ExperimentalInternetComputer";
import Debug "mo:base/Debug";

persistent actor AdvancedCanister1 {
  public func main(canisterId : Principal) : async Nat {
    // Define the method name and input args
    let name = "getValue";
    let args = (123);

    // Call the method
    let encodedArgs = to_candid (args);
    let encodedValue = await IC.call(canisterId, name, encodedArgs);

    // Decode the return value
    let ?value : ?Nat = from_candid encodedValue
        else Debug.trap("Unexpected return value");
    return value;
  }
}



import Debug "mo:base/Debug";

persistent actor AdvancedCanister2 {

  public func getValue(number: Nat) : async Nat {
     Debug.print("Hello from advanced canister 2!");
     return number * 2;
  };

};

[HTTPS Outcalls (dokumentasi lengkap)]
Since blockchains are a form of replicated state machine, where each replica must perform the same computations within the same state to make the same transitions each round, doing computations with results from an external source may lead to a state divergence. Tools like oracles have been used in the past to facilitate retrieving external data. However, on ICP, canisters can communicate directly with external servers or other blockchains through HTTPS outcalls. The response of these HTTP calls can then be used by canisters in a way that the replica can safely be updated using the response without the risk of a state divergence.

This guide uses the term HTTPS to refer to both the HTTP and HTTPS protocols. This is because typically all traffic on a public network uses the secure HTTPS protocol.

HTTPS outcalls enable many use cases and have several advantages compared to using third-party services like oracles to handle external requests. HTTPS outcalls use a stronger trust model since there are no external intermediaries required. Most real-world dapps have a need for accessing data stored in off-chain entities, as most digital data is still stored in traditional 'Web 2' servers.

Supported HTTP methods
Currently, HTTPS outcalls support GET, HEAD, and POST methods for HTTP requests. In this guide, you'll look at examples for the GET method.

Cycles
Cycles used to pay for an HTTPS call must be explicitly transferred with the call. They will not be automatically deducted from the caller's balance.

HTTPS outcalls API
A canister can make an HTTPS outcall by using the http_request method with the following parameters:

url: Specifies the requested URL; must be valid per the standard RFC-3986. The length must not exceed 8192 and may include a custom port number.

max_response_bytes: Specifies the maximum size of the request in bytes and must not exceed 2MB. This field is optional; if this field is not specified, the maximum of 2MB will be used.

It is recommended to set max_response_bytes, since using it appropriately can save developers a significant amount of cycles.

method: Specifies the method; currently, only GET, HEAD, and POST are supported.

headers: Specifies the list of HTTP request headers and their corresponding values.

body: Specifies the content of the request's body. This field is optional.

transform: Specifies a function that transforms raw responses to sanitized responses and a byte-encoded context that is provided to the function upon invocation, along with the response to be sanitized. This field is optional; if it is provided, the calling canister itself must export this function.

The returned response, including the response to the transform function if specified, will contain the following content:

status: Specifies the response status (e.g., 200, 404).

headers: Specifies the list of HTTP response headers and their corresponding values.

body: Specifies the response's body.

IPv6
When deploying applications to ICP, HTTPS outcalls can only be made to APIs that support IPv6. You can check if an API supports IPv6 by using a tool such as https://ready.chair6.net/.

HTTP GET
To demonstrate how to use the HTTPS GET outcall, open the ICP Ninja Daily Planner example, which uses HTTPS outcalls to retrieve an "On this day" fact.

In the project's backend/app.mo file, the HTTPS outcall is defined through the following lines:


  // Update function to fetch and store "On This Day" data via HTTPS outcall.
  public func fetchAndStoreOnThisDay(date : Text) : async Result.Result<Text, Text> {
    let currentData : DayData = switch (HashMap.get(dayData, thash, date)) {
      case null { { notes = []; onThisDay = null } };
      case (?data) { data };
    };

    // Perform HTTPS outcall only if needed.
    if (currentData.onThisDay == null) {
      let parts = Iter.toArray(Text.split(date, #char '-'));
      let month = Option.get(Nat.fromText(parts[1]), 1);
      let day = Option.get(Nat.fromText(parts[2]), 1);

      // Prepare the https request.
      // "transform" is used to specify how the HTTP response is processed before consensus tries to agree on a response.
      // This is useful to e.g. filter out timestamps out of headers that will be different across the responses the different replicas receive.
      // You can read more about it here: https://internetcomputer.org/docs/building-apps/network-features/using-http/https-outcalls/overview
      let http_request : IC.http_request_args = {
        // API must support IPv6
        url = "https://byabbe.se/on-this-day/" # Nat.toText(month) # "/" # Nat.toText(day) # "/events.json";
        max_response_bytes = null; //optional for request
        headers = [];
        body = null; //optional for request
        method = #get;
        transform = ?{
          function = transform;
          context = Blob.fromArray([]);
        };
      };

      // Perform HTTPS outcall using roughly 100B cycles.
      // See https outcall cost calculator: https://7joko-hiaaa-aaaal-ajz7a-cai.icp0.io.
      // Unused cycles are returned.
      Cycles.add<system>(100_000_000_000);

      // Execute the https outcall
      let http_response : IC.http_request_result = await IC.http_request(http_request);

...

        // Transforms the raw HTTPS call response to an HttpResponsePayload on which the nodes can run consensus on.
        public query func transform({
            context : Blob;
            response : IC.http_request_result;
        }) : async IC.http_request_result {
            {
            response with headers = []; // not intersted in the headers
            };
        };
    };


The code above is annotated with detailed notes and explanations. Take a moment to read through the code's content and annotations to fully understand the code's functionality.

A few important points to note are:

All methods that include HTTPS outcalls must be update calls because they go through consensus, even if the HTTPS outcall is a GET.

The code above adds 20_949_972_000 cycles. This is typically enough for GET requests, but this may need to change depending on your use case.

The transform function
The transform function in the code above is important because it takes the raw content and transforms it into a raw HTTP payload. This step sets the payload's headers, which include the Content Security Policy and Strict Transport Security headers.

When developing locally, this function doesn't have much of an effect since only the local replica (one node) is making the call.

When HTTPS calls are used on the mainnet, a common error message may appear that indicates not all replicas on the subnet get the same response:

Reject text: Canister http responses were different across replicas, and no consensus was reached


This error occurs when the replicas on the subnet don't all return the same value for a piece of data within the HTTPS response. For example, if you have an application that sends an HTTPS request to the Coinbase API for the current price of a token, due to latency the replicas will not all return the same response. To remedy this, you can request the token price for a specific timestamp to assure that the replicas all return the same response.

Calling the HTTP GET request
Click 'Deploy" within the ICP Ninja code editor to deploy the project, then open the project's frontend URL and click "Get data from the Internet" to call the backend canister's HTTPS outcall method fetchAndStoreOnThisDay.

HTTP POST
To demonstrate how to use the HTTP POST outcall, you'll create a simple canister that has one public method named send_http_post_request(). This method will trigger an HTTP POST request to a public API service where the HTTP request can be inspected, and you can inspect the request's headers and body to validate that it reflects what the canister sent.

The following are some important notes on POST requests:

Since HTTPS outcalls go through consensus, it is expected that any HTTP POST request may be sent several times to its destination. This is because it is common for clients to retry requests for a variety of reasons.

It is recommended that HTTP POST requests add the idempotency keys in the header of the request so that the destination knows which POST requests from the client are identical.

Developers should be cautious using idempotency keys since the destination server may not know how to understand and use them.

To learn more about HTTPS POST requests, view the GitHub HTTPS POST example.

HTTP
Mops packages for HTTP
Mops is an onchain package manager for Motoko. Here are some Mops packages for HTTP and web functionalities:

assets: A library for adding asset canister functionality for your canister.

assets-api: API package for asset canisters.

certified-assets: Certify assets served via HTTP, ensuring the security of query calls on ICP.

certified-cache: A single interface that stores key-value pairs and certifies their hashes for to be used as certified variables or assets.

certified-http: Similar to certified-cache, an interface that stores key-value pairs and certifies their hashes for use as certified assets or variables.

http-loopback: Call canisters using HTTP outcalls.

http-parser: HTTP request parser for parsing URLs, search queries, headers and form data.

http-types: Canister HTTP interface types used in http_request and http_request_update.

ic-assets: Asset canister with v2 certification.

ic-certification: Canister signatures and certification.

ic-websocket-cdk: Websockets on ICP.

idempotency-keys: Package for generating UUIDs.

motoko-certified-assets: ICP certified assets.

promtracker: Prometheus value tracking.

server: A server for Motoko similar to Express.

web-api and web-io: Create HTTP requests and handle responses.

[Blob handling (untuk gambar)]
Blob
Module for working with Blobs: immutable sequence of bytes.

Blobs represent sequences of bytes. They are immutable, iterable, but not indexable and can be empty.

Byte sequences are also often represented as [Nat8], i.e. an array of bytes, but this representation is currently much less compact than Blob, taking 4 physical bytes to represent each logical byte in the sequence. If you would like to manipulate Blobs, it is recommended that you convert Blobs to [var Nat8] or Buffer<Nat8>, do the manipulation, then convert back.

Import from the base library to use this module.


import Blob "mo:base/Blob";

Some built in features not listed in this module:

You can create a Blob literal from a Text literal, provided the context expects an expression of type Blob.
b.size() : Nat returns the number of bytes in the blob b;
b.vals() : Iter.Iter<Nat8> returns an iterator to enumerate the bytes of the blob b.
For example:


import Debug "mo:base/Debug";
import Nat8 "mo:base/Nat8";

let blob = "\00\00\00\ff" : Blob; // blob literals, where each byte is delimited by a back-slash and represented in hex
let blob2 = "charsもあり" : Blob; // you can also use characters in the literals
let numBytes = blob.size(); // => 4 (returns the number of bytes in the Blob)
for (byte : Nat8 in blob.vals()) { // iterator over the Blob
  Debug.print(Nat8.toText(byte))
}


Type Blob
type Blob = Prim.Types.Blob

Function fromArray
func fromArray(bytes : [Nat8]) : Blob

Creates a Blob from an array of bytes ([Nat8]), by copying each element.

Example:


let bytes : [Nat8] = [0, 255, 0];
let blob = Blob.fromArray(bytes); // => "\00\FF\00"


Function fromArrayMut
func fromArrayMut(bytes : [var Nat8]) : Blob


Creates a Blob from a mutable array of bytes ([var Nat8]), by copying each element.

Example:


let bytes : [var Nat8] = [var 0, 255, 0];
let blob = Blob.fromArrayMut(bytes); // => "\00\FF\00"


Function toArray
func toArray(blob : Blob) : [Nat8]

Converts a Blob to an array of bytes ([Nat8]), by copying each element.

Example:


let blob = "\00\FF\00" : Blob;
let bytes = Blob.toArray(blob); // => [0, 255, 0]


Function toArrayMut
func toArrayMut(blob : Blob) : [var Nat8]

Converts a Blob to a mutable array of bytes ([var Nat8]), by copying each element.

Example:


let blob = "\00\FF\00" : Blob;
let bytes = Blob.toArrayMut(blob); // => [var 0, 255, 0]


Function hash
func hash(blob : Blob) : Nat32

Returns the (non-cryptographic) hash of blob.

Example:


let blob = "\00\FF\00" : Blob;
Blob.hash(blob) // => 1_818_567_776

Function compare
func compare(b1 : Blob, b2 : Blob) : {#less; #equal; #greater}


General purpose comparison function for Blob by comparing the value of the bytes. Returns the Order (either #less, #equal, or #greater) by comparing blob1 with blob2.

Example:


let blob1 = "\00\00\00" : Blob;
let blob2 = "\00\FF\00" : Blob;
Blob.compare(blob1, blob2) // => #less

Function equal
func equal(blob1 : Blob, blob2 : Blob) : Bool


Equality function for Blob types. This is equivalent to blob1 == blob2.

Example:


let blob1 = "\00\FF\00" : Blob;
let blob2 = "\00\FF\00" : Blob;
ignore Blob.equal(blob1, blob2);
blob1 == blob2 // => true

Note: The reason why this function is defined in this library (in addition to the existing == operator) is so that you can use it as a function value to pass to a higher order function. It is not possible to use == as a function value at the moment.

Example:


import Buffer "mo:base/Buffer";

let buffer1 = Buffer.Buffer<Blob>(3);
let buffer2 = Buffer.Buffer<Blob>(3);
Buffer.equal(buffer1, buffer2, Blob.equal) // => true


Function notEqual
func notEqual(blob1 : Blob, blob2 : Blob) : Bool


Inequality function for Blob types. This is equivalent to blob1 != blob2.

Example:


let blob1 = "\00\AA\AA" : Blob;
let blob2 = "\00\FF\00" : Blob;
ignore Blob.notEqual(blob1, blob2);
blob1 != blob2 // => true

Note: The reason why this function is defined in this library (in addition to the existing != operator) is so that you can use it as a function value to pass to a higher order function. It is not possible to use != as a function value at the moment.

Function less
func less(blob1 : Blob, blob2 : Blob) : Bool


"Less than" function for Blob types. This is equivalent to blob1 < blob2.

Example:


let blob1 = "\00\AA\AA" : Blob;
let blob2 = "\00\FF\00" : Blob;
ignore Blob.less(blob1, blob2);
blob1 < blob2 // => true

Note: The reason why this function is defined in this library (in addition to the existing < operator) is so that you can use it as a function value to pass to a higher order function. It is not possible to use < as a function value at the moment.

Function lessOrEqual
func lessOrEqual(blob1 : Blob, blob2 : Blob) : Bool


"Less than or equal to" function for Blob types. This is equivalent to blob1 <= blob2.

Example:


let blob1 = "\00\AA\AA" : Blob;
let blob2 = "\00\FF\00" : Blob;
ignore Blob.lessOrEqual(blob1, blob2);
blob1 <= blob2 // => true

Note: The reason why this function is defined in this library (in addition to the existing <= operator) is so that you can use it as a function value to pass to a higher order function. It is not possible to use <= as a function value at the moment.

Function greater
func greater(blob1 : Blob, blob2 : Blob) : Bool


"Greater than" function for Blob types. This is equivalent to blob1 > blob2.

Example:


let blob1 = "\BB\AA\AA" : Blob;
let blob2 = "\00\00\00" : Blob;
ignore Blob.greater(blob1, blob2);
blob1 > blob2 // => true

Note: The reason why this function is defined in this library (in addition to the existing > operator) is so that you can use it as a function value to pass to a higher order function. It is not possible to use > as a function value at the moment.

Function greaterOrEqual
func greaterOrEqual(blob1 : Blob, blob2 : Blob) : Bool


"Greater than or equal to" function for Blob types. This is equivalent to blob1 >= blob2.

Example:


let blob1 = "\BB\AA\AA" : Blob;
let blob2 = "\00\00\00" : Blob;
ignore Blob.greaterOrEqual(blob1, blob2);
blob1 >= blob2 // => true

Note: The reason why this function is defined in this library (in addition to the existing >= operator) is so that you can use it as a function value to pass to a higher order function. It is not possible to use >= as a function value at the moment.

*Geocoding API Docs:*
[Dokumentasi API yang akan digunakan untuk verifikasi lokasi]
Geocoding request and response

bookmark_border

Request
A Geocoding API request takes the following form:


https://maps.googleapis.com/maps/api/geocode/outputFormat?parameters
where outputFormat may be either of the following values:

json (recommended) indicates output in JavaScript Object Notation (JSON); or
xml indicates output in XML
HTTPS is required.

Note: URLs must be properly encoded to be valid and are limited to 16384 characters for all web services. Be aware of this limit when constructing your URLs. Note that different browsers, proxies, and servers may have different URL character limits as well.
Some parameters are required while some are optional. As is standard in URLs, parameters are separated using the ampersand (&) character.

The rest of this page describes geocoding and reverse geocoding separately, because different parameters are available for each type of request.

Geocoding (latitude/longitude lookup) parameters
All reserved characters (for example the plus sign "+") must be URL-encoded.
Required parameters in a geocoding request:

key — Your application's API key. This key identifies your application for purposes of quota management. Learn how to get a key.
You must specify either address or components or both in a request:

address — The street address or plus code that you want to geocode. Specify addresses in accordance with the format used by the national postal service of the country concerned. Additional address elements such as business names and unit, suite or floor numbers should be avoided. Street address elements should be delimited by spaces (shown here as url-escaped to %20):

address=24%20Sussex%20Drive%20Ottawa%20ON
Format plus codes as shown here (plus signs are url-escaped to %2B and spaces are url-escaped to %20):
global code is a 4 character area code and 6 character or longer local code (849VCWC8+R9 is 849VCWC8%2BR9).
compound code is a 6 character or longer local code with an explicit location (CWC8+R9 Mountain View, CA, USA is CWC8%2BR9%20Mountain%20View%20CA%20USA).
components — A components filter with elements separated by a pipe (|). The components filter is also accepted as an optional parameter if an address is provided. Each element in the components filter consists of a component:value pair, and fully restricts the results from the geocoder. See more information about component filtering below.
See the FAQ for additional guidance.

Optional parameters in a Geocoding request:

bounds — The bounding box of the viewport within which to bias geocode results more prominently. This parameter will only influence, not fully restrict, results from the geocoder. (For more information see Viewport Biasing below.)
language — The language in which to return results.
See the list of supported languages. Google often updates the supported languages, so this list may not be exhaustive.
If language is not supplied, the geocoder attempts to use the preferred language as specified in the Accept-Language header, or the native language of the domain from which the request is sent.
The geocoder does its best to provide a street address that is readable for both the user and locals. To achieve that goal, it returns street addresses in the local language, transliterated to a script readable by the user if necessary, observing the preferred language. All other addresses are returned in the preferred language. Address components are all returned in the same language, which is chosen from the first component.
If a name is not available in the preferred language, the geocoder uses the closest match.
The preferred language has a small influence on the set of results that the API chooses to return, and the order in which they are returned. The geocoder interprets abbreviations differently depending on language, such as the abbreviations for street types, or synonyms that may be valid in one language but not in another. For example, utca and tér are synonyms for street and square respectively in Hungarian.
region — The region code, specified as a ccTLD ("top-level domain") two-character value. This parameter will only influence, not fully restrict, results from the geocoder. (For more information see Region Biasing below.) The parameter can also affect results based on applicable law.
components — A components filter with elements separated by a pipe (|). The components filter is required if the request doesn't include an address. Each element in the components filter consists of a component:value pair, and fully restricts the results from the geocoder. See more information about component filtering below.
extra_computations — Use this parameter to specify the following additional features in the response:
ADDRESS_DESCRIPTORS — See address descriptors for more details.
BUILDING_AND_ENTRANCES — See entrances and building outlines for more details.
To enable multiple of these features for the same API request, include the extra_computations parameter in the request for each feature, for example:

extra_computations=ADDRESS_DESCRIPTORS&extra_computations=BUILDING_AND_ENTRANCES
Responses
Geocoding responses are returned in the format indicated by the output flag within the URL request, or in JSON format by default.

In this example, the Geocoding API requests a json response for a query on the address "1600 Amphitheatre Parkway, Mountain View, CA".

This request demonstrates using the JSON output flag:


https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA&key=YOUR_API_KEY
This request demonstrates using the XML output flag:


https://maps.googleapis.com/maps/api/geocode/xml?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA&key=YOUR_API_KEY
Select the tabs below to see the sample JSON and XML responses.

JSON
XML

{
    "results": [
        {
            "address_components": [
                {
                    "long_name": "1600",
                    "short_name": "1600",
                    "types": [
                        "street_number"
                    ]
                },
                {
                    "long_name": "Amphitheatre Parkway",
                    "short_name": "Amphitheatre Pkwy",
                    "types": [
                        "route"
                    ]
                },
                {
                    "long_name": "Mountain View",
                    "short_name": "Mountain View",
                    "types": [
                        "locality",
                        "political"
                    ]
                },
                {
                    "long_name": "Santa Clara County",
                    "short_name": "Santa Clara County",
                    "types": [
                        "administrative_area_level_2",
                        "political"
                    ]
                },
                {
                    "long_name": "California",
                    "short_name": "CA",
                    "types": [
                        "administrative_area_level_1",
                        "political"
                    ]
                },
                {
                    "long_name": "United States",
                    "short_name": "US",
                    "types": [
                        "country",
                        "political"
                    ]
                },
                {
                    "long_name": "94043",
                    "short_name": "94043",
                    "types": [
                        "postal_code"
                    ]
                },
                {
                    "long_name": "1351",
                    "short_name": "1351",
                    "types": [
                        "postal_code_suffix"
                    ]
                }
            ],
            "formatted_address": "1600 Amphitheatre Pkwy, Mountain View, CA 94043, USA",
            "geometry": {
                "location": {
                    "lat": 37.4222804,
                    "lng": -122.0843428
                },
                "location_type": "ROOFTOP",
                "viewport": {
                    "northeast": {
                        "lat": 37.4237349802915,
                        "lng": -122.083183169709
                    },
                    "southwest": {
                        "lat": 37.4210370197085,
                        "lng": -122.085881130292
                    }
                }
            },
            "place_id": "ChIJRxcAvRO7j4AR6hm6tys8yA8",
            "plus_code": {
                "compound_code": "CWC8+W7 Mountain View, CA",
                "global_code": "849VCWC8+W7"
            },
            "types": [
                "street_address"
            ]
        }
    ],
    "status": "OK"
}
Note that the JSON response contains two root elements:

"status" contains metadata on the request. See Status codes below.
"results" contains an array of geocoded address information and geometry information.
Generally, only one entry in the "results" array is returned for address lookups, though the geocoder may return several results when address queries are ambiguous.

Status codes
The "status" field within the Geocoding response object contains the status of the request, and may contain debugging information to help you track down why geocoding is not working. The "status" field may contain the following values:

"OK" indicates that no errors occurred; the address was successfully parsed and at least one geocode was returned.
"ZERO_RESULTS" indicates that the geocode was successful but returned no results. This may occur if the geocoder was passed a non-existent address.
OVER_DAILY_LIMIT indicates any of the following:
The API key is missing or invalid.
Billing has not been enabled on your account.
A self-imposed usage cap has been exceeded.
The provided method of payment is no longer valid (for example, a credit card has expired).
See the Maps FAQ to learn how to fix this.

"OVER_QUERY_LIMIT" indicates that you are over your quota.
"REQUEST_DENIED" indicates that your request was denied.
"INVALID_REQUEST" generally indicates that the query (address, components or latlng) is missing.
"UNKNOWN_ERROR" indicates that the request could not be processed due to a server error. The request may succeed if you try again.
Error messages
When the geocoder returns a status code other than OK, there may be an additional error_message field within the Geocoding response object. This field contains more detailed information about the reasons behind the given status code.

Note: This field is not guaranteed to be always present, and its content is subject to change.
Results
When the geocoder returns results, it places them within a (JSON) results array. Even if the geocoder returns no results (such as if the address doesn't exist) it still returns an empty results array. (XML responses consist of zero or more <result> elements.)

A typical result contains the following fields:

The types[] array indicates the type of the returned result. This array contains a set of zero or more tags identifying the type of feature returned in the result. For example, a geocode of "Chicago" returns "locality" which indicates that "Chicago" is a city, and also returns "political" which indicates it is a political entity. Components might have an empty types array when there are no known types for that address component. The API might add new type values as needed. For more information, see Address types and address components.
formatted_address is a string containing the human-readable address of this location.
Often this address is equivalent to the postal address. Note that some countries, such as the United Kingdom, do not allow distribution of true postal addresses due to licensing restrictions.

The formatted address is logically composed of one or more address components. For example, the address "111 8th Avenue, New York, NY" consists of the following components: "111" (the street number), "8th Avenue" (the route), "New York" (the city) and "NY" (the US state).

Do not parse the formatted address programmatically. Instead you should use the individual address components, which the API response includes in addition to the formatted address field.

address_components[] is an array containing the separate components applicable to this address.
Each address component typically contains the following fields:

types[] is an array indicating the type of the address component. See the list of supported types.
long_name is the full text description or name of the address component as returned by the Geocoder.
short_name is an abbreviated textual name for the address component, if available. For example, an address component for the state of Alaska may have a long_name of "Alaska" and a short_name of "AK" using the 2-letter postal abbreviation.
Note the following facts about the address_components[] array:

The array of address components may contain more components than the formatted_address.
The array does not necessarily include all the political entities that contain an address, apart from those included in the formatted_address. To retrieve all the political entities that contain a specific address, you should use reverse geocoding, passing the latitude/longitude of the address as a parameter to the request.
The format of the response is not guaranteed to remain the same between requests. In particular, the number of address_components varies based on the address requested and can change over time for the same address. A component can change position in the array. The type of the component can change. A particular component may be missing in a later response.
To handle the array of components, you should parse the response and select appropriate values via expressions. See the guide to parsing a response.

postcode_localities[] is an array denoting up to 100 localities contained in a postal code. This is only present when the result is a postal code that contains multiple localities.
geometry contains the following information:
location contains the geocoded latitude, longitude value. For normal address lookups, this field is typically the most important.
location_type stores additional data about the specified location. The following values are currently supported:

"ROOFTOP" indicates that the returned result is a precise geocode for which we have location information accurate down to street address precision.
"RANGE_INTERPOLATED" indicates that the returned result reflects an approximation (usually on a road) interpolated between two precise points (such as intersections). Interpolated results are generally returned when rooftop geocodes are unavailable for a street address.
"GEOMETRIC_CENTER" indicates that the returned result is the geometric center of a result such as a polyline (for example, a street) or polygon (region).
"APPROXIMATE" indicates that the returned result is approximate.
viewport contains the recommended viewport for displaying the returned result, specified as two latitude,longitude values defining the southwest and northeast corner of the viewport bounding box. Generally the viewport is used to frame a result when displaying it to a user.
bounds (optionally returned) stores the bounding box which can fully contain the returned result. Note that these bounds may not match the recommended viewport. (For example, San Francisco includes the Farallon islands, which are technically part of the city, but probably should not be returned in the viewport.)
plus_code (see Open Location Code and plus codes) is an encoded location reference, derived from latitude and longitude coordinates, that represents an area: 1/8000th of a degree by 1/8000th of a degree (about 14m x 14m at the equator) or smaller. Plus codes can be used as a replacement for street addresses in places where addresses do not exist (where buildings are not numbered or streets are not named). The API does not always return plus codes.
When the service does return a plus code, it is formatted as a global code and a compound code:

global_code is a 4 character area code and 6 character or longer local code (849VCWC8+R9).
compound_code is a 6 character or longer local code with an explicit location (CWC8+R9, Mountain View, CA, USA). Do not programmatically parse this content.
Where available, the API returns both the global code and compound code. However, if the result is in a remote location (for example, an ocean or desert) only the global code may be returned.
partial_match indicates that the geocoder did not return an exact match for the original request, though it was able to match part of the requested address. You may wish to examine the original request for misspellings and/or an incomplete address.

Partial matches most often occur for street addresses that do not exist within the locality you pass in the request. Partial matches may also be returned when a request matches two or more locations in the same locality. For example, "Hillpar St, Bristol, UK" will return a partial match for both Henry Street and Henrietta Street. Note that if a request includes a misspelled address component, the geocoding service may suggest an alternative address. Suggestions triggered in this way will also be marked as a partial match.

place_id is a unique identifier that can be used with other Google APIs. For example, you can use the place_id in a Places API request to get details of a local business, such as phone number, opening hours, user reviews, and more. See the place ID overview.
Navigation points
This product or feature is Experimental (pre-GA). Pre-GA products and features might have limited support, and changes to pre-GA products and features might not be compatible with other pre-GA versions. Pre-GA Offerings are covered by the Google Maps Platform Service Specific Terms. For more information, see the launch stage descriptions.
The navigation_points field within the Geocoding response contains a list of points that are useful for navigating to the place. Specifically, they should be used either as the starting or ending points when routing on a road network from or to the place. Each navigation point contains the following values:
location contains the latitude, longitude value of the navigation point. This location will always be very close to the road network and represents an ideal stopping or starting point for navigating to and from a place. The point is intentionally slightly offset from the road's centerline to clearly mark the side of the road where the place is located.
restricted_travel_modes is a list of travel modes that the navigation point is not accessible from:
"DRIVE" is the travel mode corresponding to driving directions.
"WALK" is the travel mode corresponding to walking directions.
Address types and address component types
The types[] array in the result indicates the address type. Examples of address types include a street address, a country, or a political entity. There is also a types[] array in the address_components[], indicating the type of each part of the address. Examples include street number or country. (Below is a full list of types.) Addresses may have multiple types. The types may be considered 'tags'. For example, many cities are tagged with the political and the locality type.

The following types are supported and returned by the geocoder in both the address type and address component type arrays:

street_address indicates a precise street address.
route indicates a named route (such as "US 101").
intersection indicates a major intersection, usually of two major roads.
political indicates a political entity. Usually, this type indicates a polygon of some civil administration.
country indicates the national political entity, and is typically the highest order type returned by the Geocoder.
administrative_area_level_1 indicates a first-order civil entity below the country level. Within the United States, these administrative levels are states. Not all nations exhibit these administrative levels. In most cases, administrative_area_level_1 short names will closely match ISO 3166-2 subdivisions and other widely circulated lists; however this is not guaranteed as our geocoding results are based on a variety of signals and location data.
administrative_area_level_2 indicates a second-order civil entity below the country level. Within the United States, these administrative levels are counties. Not all nations exhibit these administrative levels.
administrative_area_level_3 indicates a third-order civil entity below the country level. This type indicates a minor civil division. Not all nations exhibit these administrative levels.
administrative_area_level_4 indicates a fourth-order civil entity below the country level. This type indicates a minor civil division. Not all nations exhibit these administrative levels.
administrative_area_level_5 indicates a fifth-order civil entity below the country level. This type indicates a minor civil division. Not all nations exhibit these administrative levels.
administrative_area_level_6 indicates a sixth-order civil entity below the country level. This type indicates a minor civil division. Not all nations exhibit these administrative levels.
administrative_area_level_7 indicates a seventh-order civil entity below the country level. This type indicates a minor civil division. Not all nations exhibit these administrative levels.
colloquial_area indicates a commonly-used alternative name for the entity.
locality indicates an incorporated city or town political entity.
sublocality indicates a first-order civil entity below a locality. For some locations may receive one of the additional types: sublocality_level_1 to sublocality_level_5. Each sublocality level is a civil entity. Larger numbers indicate a smaller geographic area.
neighborhood indicates a named neighborhood.
premise indicates a named location, usually a building or collection of buildings with a common name.
subpremise indicates an addressable entity below the premise level, such as an apartment, unit, or suite.
plus_code indicates an encoded location reference, derived from latitude and longitude. Plus codes can be used as a replacement for street addresses in places where they do not exist (where buildings are not numbered or streets are not named). See https://plus.codes for details.
postal_code indicates a postal code as used to address postal mail within the country.
natural_feature indicates a prominent natural feature.
airport indicates an airport.
park indicates a named park.
point_of_interest indicates a named point of interest. Typically, these "POI"s are prominent local entities that don't easily fit in another category, such as "Empire State Building" or "Eiffel Tower".
An empty list of types indicates there are no known types for the particular address component, for example, Lieu-dit in France.

In addition to the above, address components may include the types listed here. This list is not exhaustive, and is subject to change.

floor indicates the floor of a building address.
establishment typically indicates a place that has not yet been categorized.
landmark indicates a nearby place that is used as a reference, to aid navigation.
point_of_interest indicates a named point of interest.
parking indicates a parking lot or parking structure.
post_box indicates a specific postal box.
postal_town indicates a grouping of geographic areas, such as locality and sublocality, used for mailing addresses in some countries.
room indicates the room of a building address.
street_number indicates the precise street number.
bus_station, train_station and transit_station indicate the location of a bus, train or public transit stop.
Note: The Geocoding API isn't guaranteed to return any particular component for an address within our data set. What may be thought of as the city, such as Brooklyn, may not show up as locality, but rather as another component - in this case, sublocality_level_1. What specific components are returned is subject to change without notice. Design your code to be flexible if you are attempting to extract address components from the response.
