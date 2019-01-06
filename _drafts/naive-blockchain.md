---
title: "BlockChain from Scratch, Part 1: A naive BlockChain"
layout: post
---
So, BlockChains seem to be everywhere and from the way they are being described sometimes it seems like soon even your fridge will be running a BlockChain (presumably to keep a ledger of how many beers you take out!).

Most of the coverage of blockchains I’ve read was very high-level though and gave me some global insight into how they worked…​ but most of the sources I found stayed away from the actual implementation details.
A few months ago I saw a really good talk by Stefan Tilkov though ([@stilkov](https://twitter.com/stilkov)) which finally gave me some deeper insights. Knowing myself though, I knew that the best way to really grasp how blockchains worked was to implement one.

Apart from that I’ve been a big fan of the actor model ever since I did a Coursera course on reactive programming a few years back. I had been looking for a project to dive deeper into Akka and considering how event-driven and concurrent blockchains are this seemed like a perfect fit.

This article takes you through the implementation I made. It owes much to [Naivechain](https://github.com/lhartikk/naivechain), an excellent JavaScript implementation I found on Github. I took that as a starting point and then expanded on it.

The full source code containing all the snippets shown below can be found on Github:

<https://github.com/NightWhistler/naivechain-scala>

So, let’s get to the code!

## What is a BlockChain?

Seen from a very high level, a blockchain is a distributed database which implements [*eventual consistency*](https://en.wikipedia.org/wiki/Eventual_consistency). Blockchains are fully distributed, meaning that there is no central source of authority. Truth is determined by majority rule. This means that is more than half the nodes in a blockchain agree on a specific state, eventually the whole system will assume this state. Blockchains can be used to store any type of data, though they are best known for storing lists of transactions for cryptocurrencies.

As the name suggests, a blockchain is a collection of blocks which form a linked list. Each block contains some data, the hash of the full block and the hash of its predecessor.

This is what a blockchain looks like:

{% include image.html url="/assets/images/Basic_blockchain.svg" description="Basic blockchain structure" %}

Notice the first block, called the Genesis Block. This block is fixed, and acts as the ultimate "source of truth" for a blockchain. As long as all clients agree on the definition of the Genesis block, all other blocks can be checked against that.

{% include note.html content="In the theoretical case that clients do not agree on a Genesis block, you would simply get a blockchain per Genesis block. This is even true when the clients share the same P2P network, since a client will reject any block that can’t be validated against its own chain." %}

## Basic structure

I’ll start with some of the case classes representing the base concepts:

### BlockChain

The first case class is the `BlockChain` class. It encapsulates a `Seq[Block]`, where the head is always the most recent block.

```scala
case class BlockChain private( val blocks: Seq[Block] ) {
  ...
}
```

The constructor is private, since not every `Seq[Block]` would make a valid `BlockChain`. Adding data is done through the `addBlock` method, which validates any new blocks to be added to the chain. I’ll explain the details of verification in [section\_title](#_chain_verification).

### Block

Next is the `Block` case class.

```scala
case class Block(
  index: Int,
  previousHash: String,
  timestamp: Long,
  data: String,
  hash: String
)
```

It has the following fields:

index
: The index of this block in the chain. It’s not strictly necessary to store the index of blocks, but it makes validation of the chain much quicker and easier.

previousHash
: The hash-code of the previous block in the chain, stored as a base64-encoded string.

timestamp
: When this block was created. Again, you could have a blockchain without storing timestamps of the blocks, but it gives extra data for the validation.

data
: The actual content of the block, which we’ll model as a string for the time being.

hash
: The hash-code of this block, which includes all the fields.

### Hashing

The hash calculation is very simple, and not very efficient:

```scala
def calculateHashForBlock( block: Block ) =
  calculateHash(block.index, block.previousHash, block.timestamp, block.data)

def calculateHash(index: Int, previousHash: String, timestamp: Long, data: String) =
  s"$index:$previousHash:$timestamp:$data".sha256.hex
```

We just create a String from all the fields we wish to include, and then calculate the SHA256 hash for that String.

### GenesisBlock

The `GenesisBlock` is an object, since there can there can always only be 1.
It extends Block, with hard-coded values.

```scala
object GenesisBlock extends Block(0, "0", 1497359352, "Genesis block", "ccce7d8349cf9f5d9a9c8f9293756f584d02dfdb953361c5ee36809aa0f560b4")
```

Note that the hash of the previous block is set to "0", since this is the start of the chain. The hash value of the Genesis block is set here by manually calculating it. The Genesis block doesn’t really need to have a correct hash, since it’s validated by just comparing it to the hard-coded value here, but for aesthetic reasons I made sure the hash value is actually correct.

## Chain verification

To verify is a specific blockchain is valid, you start at the most recent block and calculate its hash code. This includes all the fields in the block, except for the hashcode itself. Most importantly this includes the hashcode of the previous block. This is vital to the working of a blockchain, since it means that if a single block in the chain were to be altered *all the subsequent blocks* would need to be altered too for the chain to be valid.

When the most recent block is successfully verified, we verify the previous block and so on until we reach the Genesis block. If the Genesis block of the chain equals our own Genesis block, the chain is considered valid.

```scala
@tailrec
def validChain( chain: Seq[Block] ): Boolean = chain match {
  case singleBlock :: Nil if singleBlock == GenesisBlock => true
  case head :: beforeHead :: tail if validBlock(head, beforeHead) => validChain(beforeHead :: tail)
  case _ => false
}

def validBlock(newBlock: Block, previousBlock: Block) =
  previousBlock.index + 1 == newBlock.index &&
  previousBlock.hash == newBlock.previousHash &&
  calculateHashForBlock(newBlock) == newBlock.hash
```

The `validChain` method recursively walks the chain, checking each block against its predecessor. For each block, the hash is checked against the content.

{% include note.html content="When new blocks arrive, we only need to verify them until we reach a block we’ve already verified. Because the hash of the block is dependant on the hash of the block before it, it is by extension dependant on the whole chain before it. This means there is only 1 valid chain which can lead to a any specific block hash." %}

## Peer discovery

Our blockchain clients form a very simple Peer-To-Peer network: when a client boots up, it announces itself to a fixed *seed-host*, which then sends it a list of currently known hosts in the network. The client stores this list of hosts, and then proceeds to announces itself to all of them.

**PeerToPeer.scala.**

```scala
trait PeerToPeer {
  this: CompositeActor =>
  ...
  var peers: Set[ActorRef] = Set.empty

  def broadcast(message: Any ): Unit = {
    peers.foreach( _ ! message )
  }

  receiver {

    case AddPeer(peerAddress) =>
      logger.debug(s"Got request to add peer ${peerAddress}")
      context.actorSelection(peerAddress).resolveOne().map( ResolvedPeer(_) ).pipeTo(self)

    case ResolvedPeer(newPeerRef: ActorRef) =>
      context.watch(newPeerRef)

      if ( ! peers.contains(newPeerRef) ) {
        //Introduce ourselves
        newPeerRef ! HandShake

        //Ask for its friends
        newPeerRef ! GetPeers

        //Tell our existing peers
        broadcast(AddPeer(newPeerRef.path.toSerializationFormat))

        //Add to the current list of peers
        peers += newPeerRef
      } else logger.debug("We already know this peer, discarding")

    case Peers(peers) => peers.foreach( self ! AddPeer(_))

    case HandShake =>
      logger.debug(s"Received a handshake from ${sender().path.toStringWithoutAddress}")
      peers += sender()

    case GetPeers => sender() ! Peers(peers.toSeq.map(_.path.toSerializationFormat))
    case Terminated(actorRef) =>
      logger.debug(s"Peer ${actorRef} has terminated. Removing it from the list.")
      peers -= actorRef

  }
}
```

When a node receives a new peer, it also tells all its current peers. If the node already knows the peer, it takes no action so that the process stops at this point.
This way of announcing new peers is fairly brute force, but it makes sure that new peers are known throughout the network quickly.

## Blockchain communication & Consensus

One of the main innovations of BitCoin was the way in which consensus is reached between nodes.
The basic consensus model for a blockchain is "longest chain wins".

Let’s imagine a scenario where we have 3 nodes, who currently agree on the state of the blockchain:

{% include image.html url="/assets/images/nodes_before_add.svg" description="Initial blockchain network with 3 nodes in agreement" %}

Now, say that node `N0` receives a new block `X`, and node `N2` receives a block `Y` at *exactly* the same time.

{% include image.html url="/assets/images/add_data.svg" description="Two nodes receive a new block simultaneously" %}

Both nodes will then add the block to their chain, and try to send the new block to their peers.
Both `N0` and `N2` will reject each other’s blocks, since their chain already has a block at this index.
The state of `N1` is effectively random: it will either accept `X` or `Y`, whichever arrives first.

At this moment, we have a split and the network is in an inconsistent state. It will remain in this state until new data is added to the network.

Now, imagine `N0` receives another data block `Z`.

It will add this block to its version of the blockchain, and then send out the block to all its peers.
Now, if we assume that `N1` had in fact accepted `X`, then it will now see the new block `Z` and simply add it to the chain.

For `N2` the story is slightly more complicated. It sees a new block `Z`, but it has a predecessor `X` instead of `Y`.
At this point `N2` will ask its peers for a full copy of their blockchain. When this chain comes in, it will validate it.
If the new chain is both valid and longer than its own chain, it will adopt the new chain and discard its own copy.

Note that at this point, `Y` has effectively disappeared from the network.

If you have heard that in BitCoin you can consider a transaction to be finally recorded after 2 blocks, this is why.

```scala
object BlockChainCommunication {

  case object QueryLatest
  case object QueryAll

  case class ResponseBlockChain(blockChain: BlockChain)
  case class ResponseBlock(block: Block)

}

trait BlockChainCommunication {
  this: PeerToPeer with CompositeActor =>

  import BlockChainCommunication._

  val logger = Logger("PeerToPeerCommunication")

  var blockChain: BlockChain

  receiver {
    case QueryLatest => sender() ! responseLatest
    case QueryAll => sender() ! responseBlockChain

    //FIXME: This is inefficient
    case ResponseBlock(block) => handleBlockChainResponse(Seq(block))
    case ResponseBlockChain(blockChain) => handleBlockChainResponse(blockChain.blocks)
  }

  def handleBlockChainResponse( receivedBlocks: Seq[Block] ): Unit = {
    val localLatestBlock = blockChain.latestBlock
    logger.info(s"${receivedBlocks.length} blocks received.")

    receivedBlocks match {
      case Nil => logger.warn("Received an empty block list, discarding")

      case latestReceivedBlock :: _ if latestReceivedBlock.index <= localLatestBlock.index =>
        logger.debug("received blockchain is not longer than received blockchain. Do nothing")

      case latestReceivedBlock :: Nil if latestReceivedBlock.previousHash == localLatestBlock.hash =>
         logger.info("We can append the received block to our chain.")
            blockChain.addBlock(latestReceivedBlock) match {
              case Success(newChain) =>
                blockChain = newChain
                broadcast(responseLatest)
              case Failure(e) => logger.error("Refusing to add new block", e)
            }
      case _ :: Nil =>
            logger.info("We have to query the chain from our peer")
            broadcast(QueryAll)

      case _ =>
            logger.info("Received blockchain is longer than the current blockchain")
            BlockChain(receivedBlocks) match {
              case Success(newChain) =>
                blockChain = newChain
                broadcast(responseBlockChain)
              case Failure(s) => logger.error("Rejecting received chain.", s)
            }
    }
  }

  def responseLatest = ResponseBlock(blockChain.latestBlock)

  def responseBlockChain = ResponseBlockChain(blockChain)

}
```

{% include note.html content="You probably noticed that by itself, 'longest chain wins' may work as a way of reaching consensus, but it would be trivially simple to 'falsify history' by just starting a new chain and adding more elements than the longest current chain. This is where difficulty functions come in, which I’ll show in [section\_title](#p-going-cynical)." %}

## Mining blocks

Mining is the process of adding a new block to the blockchain. In this first naive version, there is no proof of work needed, so a block can simply be added to the chain:

```scala
object Mining {
  case class MineBlock( data: String )
}


trait Mining {
  this: BlockChainCommunication with PeerToPeer with CompositeActor =>

  receiver {
    case MineBlock(data) =>
      blockChain = blockChain.addBlock(data)
      val peerMessage = ResponseBlock(blockChain.latestBlock)
      broadcast(peerMessage)
      sender() ! peerMessage
  }
}
```

When we receive a request to add a new block to the chain, we simply call `addBlock` on our current blockchain, and broadcast the result.

## Nuts & Bolts

Now that we’ve seen all the blockchain-specific stuff, we just need to put it all together into a working application that we can use. For a UI we’ll offer simple REST services that we can call with `curl`, and the P2P communication will be implemented using Akka remote actors.

### Actor communication

Peer to Peer communication is handled through Akka remote actors.
We configure Akka to listen on port 2552

This is a matter of setting the right values in your TypeSafe config file:

**reference.conf.**
```
akka {
  loglevel = DEBUG
  actor {
    provider = remote
  }
  remote {
    enabled-transports = ["akka.remote.netty.tcp"]
    netty.tcp {
      port = 2552
    }
 }
}
```

### Web interface

To implement the REST interface, Akka-HTTP makes for a natural fit. I used JSON4s here to serialize the case classes to JSON.

```scala
val routes =
  get {
    path("blocks") {
      val chain: Future[Seq[Block]] = (blockChainActor ? QueryAll).map {
        //This is a bit of a hack, since JSON4S doesn't serialize case objects well
        case ResponseBlockChain(blocks) => blocks.slice(0, blocks.length -1) :+ GenesisBlock.copy()
      }
      complete(chain)
    }~
    path("peers") {
      complete( (blockChainActor ? GetPeers).mapTo[Peers] )
    }~
    path("latestBlock") {
      complete( (blockChainActor ? QueryLatest).map {
        case ResponseBlock(GenesisBlock) => GenesisBlock.copy()
        case ResponseBlock(block) => block
      })
    }
  }~
  post {
   path("mineBlock") {
     entity(as[String]) { data =>
       logger.info(s"Got request to add new block $data")
       val blockMessage = BlockMessage(data)
       blockChainActor ! MineBlock(Seq(blockMessage))
       complete(blockMessage)
    }
  }~
  path("addPeer") {
    entity(as[String]) { peerAddress =>
      logger.info(s"Got request to add new peer $peerAddress")
      blockChainActor ! AddPeer(peerAddress)
      complete(s"Added peer $peerAddress")
    }
  }
}
```

This implements the following endpoints:

/blocks
: Returns a list of all the blocks in the blockchain, i.e. shows the blockchain

/peers
: Returns a list of all the peers this node knows off

/latestBlock
: Returns the block that has most recently been added to the chain.

/mineBlock
: Takes the request body and adds it to the chain, creating a new block.

/addPeer
: Convenience method to register a new peer through the web-interface

We can query the blockchain like this:

    $ curl http://localhost:9000/blocks

Which will give the following output:

    [{"index":0,"previousHash":"0","timestamp":1497359352,"data":"Genesis block","hash":"ccce7d8349cf9f5d9a9c8f9293756f584d02dfdb953361c5ee36809aa0f560b4"}]

### Stackable traits

In the previous code examples, you may have noticed a reference to the `CompositeActor` class. This is an extra base class I added to implement the *stackable traits* pattern.

This pattern allows us to decompose the behaviour of an actor into several traits, which can then be tested in isolation.

This part is not really essential for the behaviour of the blockchain, so feel free to skip ahead.

It works like this:

```scala
class CompositeActor extends Actor {
  var receivers: Receive = emptyBehavior
  def receiver(next: Receive) { receivers = receivers orElse next }
  final def receive = receivers
}
```

This allows us to do this:

```
class BlockChainActor( var blockChain: BlockChain ) extends CompositeActor with PeerToPeer
  with BlockChainCommunication with Mining {
  override val logger = Logger("BlockChainActor")
}
```

{% include note.html content="You may wonder why so much behaviour is packed into a single Actor: the main reason is that it makes it very easy to reason about changes to the blockchain. It means that inside the context of the node, there is only 1 'true' blockchain." %}

So why does this work?

The normal Akka `receive` method actually returns a `PartialFunction` (`Receive` is an alias).
This means that it implements the `isDefinedAt()` method, which will return `true` or any object you have written a `case` statement for, and `false` for anything else.

`PartialFunction` objects can be chained using the `orElse` method, meaning that if the left-hand method is not defined for a specific
argument, it will try the right-hand method.

Each of the traits look like this:

```scala
trait PeerToPeer {
  this: CompositeActor =>
  ...
  receiver {
    ...
  }
}
```

  - We require any class implementing this trait to be a `CompositeActor` through usage of a `self-type`. This is so we can call the `receive` method.
  - This is not a method definition! It’s a method call to the `receive` method in CompositeActor, passing a new `PartialFunction` as an argument.

