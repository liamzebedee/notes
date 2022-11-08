# A dumb primer on recursive ZK proofs 

How do I understand recursive ZK proofs? / A better way.

By **Liam Zebedee ([@liamzebedee](https://twitter.com/liamzebedee))** and **XYZ**

**Acknowledgements**. This came directly out of convos with Lord from Biblioteca, Francesco from Apibara, Guiltygyoza, in the entire StarkNet hacker house at ETHLisbon 2022. Thank you to them, and the organisers of that house.

PS: Don't feel scared by the length - the sections are designed to be skippable.

## Part 1: memoization.

In JavaScript, we have a common optimization pattern called [memoization](https://en.wikipedia.org/wiki/Memoization). For those not familiar, memoization is kind of like caching the results of calling a function, so when you call it again with the same inputs, you get the result from the cache.

Memoization is usually implemented as a higher-order function, meaning it takes a function as input and returns another function. Here's an idiot's implementation of `memoize(x)` in Python:

```py
cache = {}
def memoize(f):
    # The memoized version of f(x).
    def f2(x):
        if not x in cache:
            cache[x] = f(x)
        return cache[x]
    return f2
```

**Learning**: memoization is wrapping a function with a results cache.

## Part 2: ZK proofs - a practical walkthrough in Cairo.

Now let's recap ZK proofs - a shorthand for ZK-STARK's and ZK-SNARK's. 

A ZK proof is a way to prove computation, in such a way that verifying that proof takes less time than running it. For a program that takes $N$ computational steps, we can prove it and verify that proof in $O(log^2 N)$ steps using STARK's - an exponential speedup. I've written a fairly good summary of this on [this Twitter thread](https://twitter.com/liamzebedee/status/1516241618919374851)). 

What does ZK proofing look like? Usually you write a "provable" program using a language like [Cairo](https://www.cairo-lang.org/), which compiles to bytecode, which is executed in a VM that generates a "trace" - basically, a dump of the values of the VM's CPU registers over time. This trace is then proven using a prover (see [Giza](https://github.com/maxgillett/giza), an open-source one I worked on), and out the other side, you get a binary blob. 

For example:

```cairo!
# program.cairo
func fib(n) {
    if n == 0 {
        return 0
    }
    if n == 1 {
        return 1
    }
    return fib(n-1) + fib(n-2)
}
```


Compile, run and generate the trace:

```shell!
# Compile.
cairo-compile ./program.cairo --output ./program.json

# Run.
cairo-run --program=program.json --layout=all --memory_file=memory.bin --trace_file=trace.bin
```

Now prove the trace:

```shell!
giza prove --trace=trace.bin --memory=memory.bin --program=program.json --output=proof.bin
```

The proof is generated as a file, `proof.bin`. The proving step will take longer than running the program, but the verification will be much shorter. We can verify it like so:

```shell!
giza verify --proof=proof.bin
```

Now, how do we know what we're actually verifying? Obviously it could be a proof of anything. In some ZK systems like [circom](https://github.com/iden3/circom), we have two separate compilation outputs - the proving circuit and the verification circuit, the latter of which we would use to authoritively verify a program run. In Starkware, they have a primitive higher up the stack called the "bootloader", which a **known program** that will execute any program given to it. 

How do we know we're running the bootloader? The program hash is exposed as public memory input into the program run, and because it's public memory, we can access it as part of the proof in `proof.bin`. 

This is what it looks like, conceptually:

```cairo!
func bootloader(code: felt*, program_hash: felt, input: felt*) {
    # communicate with the verifier that we are executing this program.
    assert hash(code) == program_hash
    # write program_hash to public_memory
    
    # execute
    push input to memory
    jump code
    # equivalent to f(x), where f=code, x=input
}
```

To illustrate how this works, we might have `bootloader.py` which will setup the bootloader to run our program. Then proving looks the same:

```py!
# This will write the program.json and the input data (fib(20)) to the bootloader's memory slot.
python3 bootloader.py load --program program.json --entrypoint "fib(20)" --output-memory memory.bin

# Now we run using the bootloader, which will run fib(20).
cairo-run --program=bootloader.json --layout=all --memory_file=memory.bin --trace_file=trace.bin

# Same steps for prove.
giza prove --trace=trace.bin --memory=memory.bin --program=program.json --output=proof.bin

# Verify.
# a. Verify we are running program.json, by checking the program_hash == H(program.json) inside proof.bin.
python3 bootloader.py verify-invocation --proof proof.bin --program program.json --entrypoint "fib(20)"
# b. Verify the proof.
giza verify --proof=proof.bin
```

And that's basically the full runthrough of a real ZK-STARK system, minus the bootloader.

**Learning**: We write a program in Cairo, compile it to bytecode, run it to generate a trace, prove the trace, and then verify the proof. To verify what we're verifying, we use something called a bootloader, which communicates the program hash in the proof data via public memory.

## Part 3: Recursive ZK proofs.

So what the fuck is recursive ZK proofing? 

It's alien technology, that's what it is. We like communicating with aliens. But like most aliens, you can't understand what the fuck they're talking about at first.

The first lesson - recursion in proofing does not look like recursion **in any other language you're familiar with**. There is no "call stack" that is growing in size. There is no "calling the same function". So I think the analogy is completely stupid, tbh.

Recursive proofing is this idea of "verifying proofs within proofs". Basically, you have `giza prove`, but this program is actually implemented in Cairo. So it's an API you can call within your ZK programs, which allows you to verify proofs from other program invocations. 

This is fucking magic. Because you can verify a proof that verifies other proofs, which verifies other proofs...and eventually you get a single 20kB file which succinctly verifies an entire blockchain's history. See [ZeroSync for a genuine attempt to do this on Bitcoin](https://github.com/lucidLuckylee/zerosync). 

**But**, it's a terrible analogy. The recursion occurs conceptually, not inside of the programming language. It actually just looks like invoking a pure function. 

The only language that implements recursive proofing right now is Mina. I've [written an example which uses recursion in ZK proofs](https://gist.github.com/liamzebedee/8d2efbb105d3b474dbe752d526cc9d27), and this is what it looks like using their API:

```ts
# From https://gist.github.com/liamzebedee/8d2efbb105d3b474dbe752d526cc9d27

        transfer: {
            privateInputs: [SelfProof],

            method(t1: Transaction, t0: SelfProof<Transaction>) {
                // Verify the state at t=0 was computed correctly.
                t0.verify();

                // State: utxo signature validation.
                const t1_sighash = t1.sighash();
                const msg = [t1_sighash];
```

t0 is a `SelfProof`, basically a class which is a proof. `verify()` is a method. Does that look like recursion to you? No. 

**Learning**: Recursive ZK proofs are extremely powerful. But the recursion occurs conceptually, not like it looks like in normal programming (no call stacks, no calling the same function).


## Part 4: A better analogy for recursive proofs.

So here's the meat - I think memoization is a much better way to imagine recursion in ZK proofing. Coming back to the beginning of our article, this is `memoize(x)`:

```py
cache = {}
def memoize(f):
    # The memoized version of f(x).
    def f2(x):
        if not x in cache:
            cache[x] = f(x)
        return cache[x]
    return f2
```

The funny thing is, a ZK proof is kind of like a secure cache. If I compute `fib(n=100)`, and give you a number, you can't trust me without running the `fib(n=100)` which is $O(N)$ steps. But what if I prove `fib(n=100)` and give you the proof? Then it costs you $O(log^2 N)$ steps to trust me, by verifying the proof. 

What might this look like applied to our memoize helper?

```py
cache = {}
def memoize(f):
    # The "securely" memoized version of f(x).
    def f2(x):
        if not x in cache:
            cache[x] = prove(f(x))
        return verify(cache[x])
    return f2
```

**Learning**: Another way to understand recursive proofs is through the idea of a "secure cache". The cache values are checked automatically using the `verify` function from Starkware.


## Part 5: A better API for recursive proofs.

Recursion is a fitting analogy conceptually, but when programming, we might think of a more ergonomic API design. What if we could annotate any function as `provable`? e.g.

```cairo!
# program2.cairo

#[provable]
func fib(n) {
    if n == 0 {
        return 0
    }
    if n == 1 {
        return 1
    }
    return fib(n-1) + fib(n-2)
}
```

And in other places, we could interact as so:

```cairo!
# program2.cairo

func run_1() {
    proof = fib.prove()
}

func run_2(proof: felt*) {    
    // Verify the proof.
    val = fib.verify(proof)
}
```

But why have `prove`/`verify` splattered throughout our code? Could we go more ergonomic? Why not have this built into the runner? Maybe we could have a shared `cache` for all function invocations, and the runtime would automatically memoize and check the cache. 

Here's how this might work:

 * a public area of memory dedicated to the cache.
 * cairo-run and giza are modified to accept "secure cache entries" - proofs.
 * the Cairo runtime automatically rewrites functions annotated as "provable" to check this cache for results, thus removing the need for the programmer to write "verify".

The potential workflow:

```cairo!
# program3.cairo

#[provable]
func fib(n) {
    if n == 0 {
        return 0
    }
    if n == 1 {
        return 1
    }
    return fib(n-1) + fib(n-2)
}

func main1() {
    val = fib(100)
}

func main2() {
    val = fib(100)
}
```

There are no verify API calls here. The verification occurs automatically inside the Cairo runtime - `cairo-run`. We begin by running the program:

```shell!
# Compile.
cairo-compile ./program3.cairo --output ./program3.json

# Run main1.
cairo-run --program=program3.json --entrypoint "main1" --layout=all --memory_file=memory.bin --trace_file=trace1.bin --cache_file=cache1.json

# Prove main1.
giza prove --trace=trace1.bin --memory=memory.bin --program=program3.json --output=proof1.bin
```

This might generate `cache1.json` looking like:

```json
[
    {
        "name": "fib",
        "code": "00a00e0fd0bca...",
        "hash": "010e0f0dd0bca...",
        "cache": {
            "100": "PROOF_DATA"
        }
    }
]
```

The second invocation, `main2`, we run the same function but we supply the cache of function invocations. 

```shell!
# Run main1.
cairo-run --program=program3.json --entrypoint "main2" --layout=all --memory_file=memory.bin --trace_file=trace2.bin --cache_file=cache1.json

# Prove main1.
giza prove --trace=trace2.bin --memory=memory.bin --program=program3.json --output=proof1.bin
```

And there you have it - an API for recursive proofs that doesn't litter our code. Comparing this against the Mina code, where we have to be **manually passing around proofs** and intermixing our code with these verify functions, this is much more ergonomic. `SelfProof<Transaction>.verify()` vs. `#[provable]`. It feels very clean.
