

# Loot Fog of War, private-information games, and ZK STARK's.

By **Liam Zebedee ([@liamzebedee](https://twitter.com/liamzebedee) aka SugarLord)**.

**Context**: Loot Assassins and their [fog-of-war problem](https://twitter.com/lordOfAFew/status/1588204458420338689)

# Intro.

If you've ever played Age of Empires, Guild Wars, or any other real-time strategy (RTS) game, you know it's a fun time. Gathering resources, assembling armies and then plotting attacks against other players is awesome - and it's especially fun given a multiplayer environment, what sorts of strategies emerge. In recent years, money has become more a part of online gaming than ever before - for example, [EVE Online's $1M+ battle](https://www.vice.com/en/article/9kn745/eve-online-million-dollar-battle). 

The natural question to ask is - why not on-chain? Blockchains are in essence, an open game platform, where anyone can permissionlessly build new game items and lore. Imagine a version of World of Warcraft, where anyone could create their own quests, with their own items, game mechanics, and so on. This is being made today in a project/network called [Loot / the Lootverse](https://www.lootproject.com/). What's more - blockchains connect you with a wide range of infrastructure - simply implementing a game that adheres to the ERC20 standard for currency, or the ERC721 standard for items, means your game's items and currency are automatically displayed in a wider web of apps - users can show off their character's items in social networks like [Farcaster](https://www.farcaster.xyz/), they can trade items on [Uniswap](uniswap.org/) or [OpenSea](https://opensea.io/), and much more. 

The challenges with building rich on-chain games really fall into a couple of categories:

 1. Scaling computation.
 2. Scaling state.
 3. Private information.

Computation/state are notoriously expensive on Ethereum. You can't run a physics engine on Ethereum, for example. However, there's a new technology which is making this *exponentially* cheaper - STARK's. STARK's are a type of cryptography that you can use to prove computation - something very useful in a trustless environment like blockchain. Using STARK's, we can verify computation in $O(log^2 N)$ through checking a proof, as opposed to verifying it naively by re-rerunning it, which is $O(N)$. So computation is getting a *lot* cheaper. 

What about state? Our systems are getting ripped apart and specialised by the day. The insanely high cost of storing data (state) is mainly due to the lack of horizontal scalability in blockchains - since every node must transmit and store every transaction, it's very expensive per byte. Again, STARK's have changed the game here - using something called recursive ZK proofs, it's possible to aggregate the proven state roots of multiple blockchains in parallel - without needing to transfer the raw state itself. This essentially will unlock horizontal scalability, something we've had in web2 databases for a long time. The general term people are using here is L3's, but honestly I think it's a dumb term. (side-note: I'm building a horizontally-scalable decentralized database based this called [Goliath](https://glissco.notion.site/Goliath-whitepaper-7cf50163301a42c7b264b02f248a6a07), check it out)

The last problem is "private information". And this is the most interesting and why I'm writing this post.

### Private information in games.

When you're playing Age of Empires, there's this concept of "fog of war" - the part of the map you cannot see. It's *essential* to the game - you have no idea what the other players are doing, what their army looks like, etc. This is _private information_. 

When you put a game (or financial protocol) on-chain, you are essentially publishing a bug bounty. The blockchain is an extremely adversarial environment, where every strategy is being explored by a decentralized community of agents - simple trades are [frontrun and "sandwiched"](https://coinmarketcap.com/alexandria/article/what-are-sandwich-attacks-in-defi-and-how-can-you-avoid-them), agents run specialised software to simulate pending transactions and re-submit them as their own if they can claim the profit, and then there is *another* layer of metagame where there are other players who engineer [contracts to purposefully honeypot](https://www.mev.wiki/attempts-to-trick-the-bots/salmonella) those who simulate and hijack other tx's. Point is - if there is a vector, it will be exploited.

All public information, can, and will, be exploited. But we *need* our Age of Empires clone to have private information - if any player (human or some weird deep reinforcement learning AI) could see where the other players army is, they could win easily. Would that EVE Online battle still work if all the info was public? No.  

### The "fog of war" problem.

So how could we implement it? First, let's work with a simple definition of the problem-

**Problem**. There are two players, each building armies. We want to build an on-chain game where the players can move their armies around a map. _If the armies meet_, meaning their coordinates match up, a battle is started and their positions are revealed. Until that point, all information about the location of armies is private from other players. 

You can imagine the state looks something like this:

```python!
class Game:
    player1: Player
    player2: Player

class Player:
    army_pos: [x, y]
```

The actual game logic is on-chain, and might look like this: 

```cairo!
# army.cairo
func move_army(x: felt, y: felt) {
    players[msg.sender].army_pos = [x,y];
    
    if(players[0].army_pos == players[1].army_pos) {
        commence_battle();
    }
}
```

Now obviously, the player state in this contract is public. How do we make the state private? 

### Basic solutions - encryption, hashing.

The first intuition might be to encrypt it, but remember - the logic can't be verified on-chain unless it is publicly decrypted - which would make the information public, thus negating our objective. There are methods of doing computation on encrypted data called [homomorphic encryption](https://en.wikipedia.org/wiki/Homomorphic_encryption), but they aren't ready yet.

The second intuition might be to hash it - thus obscuring the information from public view. 

```cairo!
# army.cairo

# position_hash is set as hash(x ++ y).
func move_army(position_hash: felt) {
    players[msg.sender].army_pos = position_hash;
    
    if(players[0].army_pos == players[1].army_pos) {
        commence_battle();
    }
}
```

This _could work_! But it's essentially [security-by-obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity) - anyone could go to the effort of precomputing all of the hashes for positions on the map, and basically reverse-engineer a player's position (this is called a [rainbow table](https://en.wikipedia.org/wiki/Rainbow_table)). 

What if we salted the hash? So each hash is completely unique. This is interesting.

```cairo!
# army.cairo
# position_hash is set as hash(x ++ y ++ salt), where salt=random().
func move_army(position_hash: felt) {
    players[msg.sender].army_pos = position_hash;
    
    if(players[0].army_pos == players[1].army_pos) {
        commence_battle();
    }
}
```

But then the position collision logic would never work. Hmmmm. 

### A trusted dungeon-master to custody private info?

What if there was another party that we trusted to store our private information (positions)? And they performed the army collision logic? This would make the game centralized, but let's play with the idea. Let's call them the **dungeon master**. How would that look? 

```python!
# dungeon_master.py

# On-chain.
class ArmyContract:
    def commence_battle(player1, player2):
        # calls on-chain contract.

# Off-chain.
class DungeonMaster:
    players = []
        
    def move_army(x: felt, y: felt):
        players[msg.sender].army_pos = [x,y]
    
        if(players[0].army_pos == players[1].army_pos):
            ArmyContract.commence_battle()
```

The `DungeonMaster` stores the player's locations, and runs the collision detection itself. When it detects two armies have met, it commences a battle by calling the on-chain contract. 

The problem here is that (1) the logic is not verifiable and (2) the state is not verifiable. 

### A trustless dungeon-master.

What if we built something like a state channel? Basically, players sign transactions and submit them off-chain to the DungeonMaster. The DungeonMaster stores the latest state, and when there is a collision, the it submits those transactions on-chain, updating their positions to the latest state, and then commences the battle. This is in essence, an [optimistic rollup](https://ethereum.org/en/developers/docs/scaling/optimistic-rollups/) with a centralized operator that can custody private state. 

That's cool! But the problem is it's complex. Is there a better way? 

#### Fraud proofs and Validity proofs.

Optimistic rollups are based on optimism - you optimistically assume that the state which is posted on-chain was computed correctly, and you hope that someone is verifying it was. If there is fraud, then the complete transaction is re-run on-chain in $O(N)$ time (in practice, it's a bit more efficient due to [interactive fraud proofing](https://medium.com/offchainlabs/interactive-fraud-proofs-arbitrums-secret-sauce-debc3b019418) but it's essentially the same worst-case).

A better approach is validity proofs, which are based in ZK. A ZK proof is by nature, nihilistic - it doesn't give a shit whether you believe in maths or not, but if you do (which we do) - you can prove computation in $O(log^2 N)$ efficiency. This is much better. 

### A trustless VM dungeonmaster.

So imagine this: the DungeonMaster operates as a Cairo VM. It processes each transaction. When there is a collision, it generates a proof of processing all of these transactions, and submits it to the remote blockchain - StarkNet. 

What does this look like? We return to our original Cairo model:

```cairo!
# army.cairo
func move_army(x: felt, y: felt) {
    players[msg.sender].army_pos = [x,y];
    
    if(players[0].army_pos == players[1].army_pos) {
        commence_battle();
    }
}
```

#### On the DungeonMaster chain.

Let's imagine the "position" state as its own little state machine, where players invoke the `next_position` function to move, and our proof basically returns the latest position:

```cairo!
# dungeon-master/army.cairo
func next_position(x: felt, y: felt) {
    players[msg.sender].army_pos = [x,y];
    
    if(players[0].army_pos == players[1].army_pos) {
        commence_battle_hook();
    }
}
```

This contract is deployed in a local Cairo VM. Every transaction submitted will change the state of this VM. When a collision is detected, we need to do two things: 

 1. generate a proof of the latest state and
 2. submit this proof to the remote blockchain. 

We need some way to interrupt and hook this event. For now, we're just going to assume we've got this figured out (maybe using events), and call it `commence_battle_hook`. 

`commence_battle_hook` will save the current state of the VM (trace), prove it using a prover like [Giza](https://github.com/maxgillett/giza), and then send that proof to the remote StarkNet blockchain to commence the battle. (see my article on [recursive proofing to see how proving might look](https://hackmd.io/@liamzebedee/BydanTDSi)).

#### On the StarkNet chain.

On the StarkNet chain, we will process this proof, containing the latest state - the positions of the two players - and then commence the battle. 

```cairo!
# starknet/army.cairo

# Applies the latest "position" state from the Dungeon Master, shown by the proof.
func process_moves(move_proof: felt*) {
    (valid, (x, y)) = next_position.verify(move_proof);
    assert valid;
    players[msg.sender].army_pos = [x,y];
}

func commence_battle(player1_position_proof: felt*, player2_position_proof: felt*) {
    process_moves(player1_position_proof);
    process_moves(player2_position_proof);
    
    if(players[0].army_pos == players[1].army_pos) {
        commence_battle();
    }
}
```

And there we have it! The dungeon master is in essence its own little blockchain, with private state that it custodies. Players interact with the DM, and when there is a collision, it proves it, and sends it to a remote blockchain (StarkNet), which will commence the battle. The dungeon master cannot perform malicious behaviour, because it runs a verifiable state machine, whose transitions are proven using STARK's.

This example glosses over a couple of security concerns for simplicity's sake, though they're all quite addressable: 

 * **How do we authenticate tx's?** The user signs txs, just like in a rollup. The signature validation would occur inside `next_position`.
 * **How do we ensure fair sequencing and fair ordering?** Right now, these off-chain transactions can be re-ordered or even dropped entirely. Like a regular blockchain, there needs to be a sequencer - arguably this could just be the DM. This sequencer would sign every tx it processes, and probably have a stake that could be slashed if it is shown that it has behaved inconsistently (signing more than 1 different sequence of txs).


## Conclusion.

Private information in video games is pretty damn essential. And on-chain games are going to be awesome, given we can mess around with real money and DAO's. How do we get private info though? We can tradeoff a little bit of trust by using a trusted 3rd party, the dungeon master, to custody our private information. Unlike a centralized party that might just sign state updates (like a ChainLink oracle), we're only trusting this third party to keep our information private - the rest of their job (computation) is secured by STARK validity proofs. 

This whole example is really just a stepping stone on the way to what people keep calling "L3 scaling". I wrote it up pretty quickly to illustrate the direction this is all heading in - towards an even more generalised cross-chain state transfer using ZK proofs. 

### Future research.

### A generalised state machine.

Can we go harder? What if we didn't have to write a `process_moves` function? What would something more generalised look like? 

_This is left as an exercise to the guilty gyoza._

<!-- But it gets *even better*. 

ZK proofs can prove other ZK proofs. To illustrate, let's return to our last idea:

> the DungeonMaster stores the latest state, and when there is a collision, it submits those transactions on-chain, updating their positions to the latest state, and then commences the battle.

Instead of processing $N$ txs on-chain, a ZK proof will be $log (N)$. It looks something like this:
 -->
 
 
### MPC, Homomorphic computation

Honestly private info and multi-party computation / FHE (fully homomorphic encryption) could very much solve this, though we don't know how yet. 

There are a couple groups specialised in this area that'd have some good ideas on this:

 - [cronokirby](https://twitter.com/messages/1712298445-1102909424912478208)
 - [henry](https://twitter.com/hdevalence) and the [penumbra](https://twitter.com/penumbrazone) team




