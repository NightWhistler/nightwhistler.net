---
title: "BlockChain from Scratch, Part 2: Adding a PoW Algorithm"
layout: post
---

Now we have a basic blockchain, which shows the concept of how a chain works.
In its current form it’s not terribly useful however. It’s essentially a distributed linked-list, with very little guarantees.

In the next section we’ll build on the naive chain implementation to create a blockchain that can actually be put to some use.

## What was so naive about the naivechain?

As you may recall from the start of this article, the blockchain consensus model is "longest chain wins".
So what if you wanted to change history and generate your own chain, starting at the GenesisBlock, but with your own data?

At the moment putting things in the chain is essentially "free", meaning that it’s trivial to start over and generate your own chain. This is where the concept of Proof of Work comes in.

## Proof of work

In real-word blockchains like BitCoin, adding data to the chain is not "free". The problem with someone being able to start a new chain is mitigated here by making it difficult to put new blocks in the chain. For a block to be valid, we need to show that we’ve done a certain amount of work to create the block. Often this is done by placing certain restrictions on what the hash of the block should look like.

A fairly straight-forward way to implement Proof of Work is to require the hash of the block to be smaller than a specific value. This means that to add a block to the chain, we need some way to manipulate the hash of the block. This is done by adding a *nonse*. A nonse is a random value that doesn’t hold any meaning in itself. Here we’ll just increment a counter.

First, let’s update the `BlockChain` class we saw earlier.

```scala
object BlockChain {

  type DifficultyFunction = Block => BigInt
  type HashFunction = Block => BigInt

  val defaultDifficultyFunction = NaiveCoinDifficulty
  val defaultHashFunction = SimpleSHA256Hash

  def apply( difficultyFunction: DifficultyFunction = defaultDifficultyFunction, hashFunction: HashFunction = defaultHashFunction )
    = new BlockChain(Seq(GenesisBlock), difficultyFunction, hashFunction)

}
```

In the companion object I added 2 new type aliases: `DifficultyFunction` and `HashFunction`. Both take a Block and return a BigInt.

Then, I added 2 fields to the `BlockChain` class itself:

```scala
case class BlockChain private(val blocks: Seq[Block], difficultyFunction: DifficultyFunction, hashFunction: HashFunction) {
```

`DifficultyFunction` and `HashFunction` are simple type aliases:

```scala
type DifficultyFunction = Block => BigInt
type HashFunction = Block => BigInt
```

This uses the fact that a hash-code is essentially just a number. This means that to implement a difficulty-function we can require it to be below a certain threshold. As the chain gets longer, we lower the required value making it harder to add new blocks to the chain.

```scala
private def validBlock(newBlock: Block, previousBlock: Block, messages: Set[BlockMessage]) =
  previousBlock.index + 1 == newBlock.index &&
  previousBlock.hash == newBlock.previousHash &&
  (newBlock.timestamp - (new Date().getTime / 1000)) < TWO_HOURS &&
  previousBlock.timestamp <= newBlock.timestamp &&
  hashFunction(newBlock) == newBlock.hash &&
  newBlock.hash < difficultyFunction(newBlock) && 
  ! newBlock.messages.exists( messages.contains(_))
```
The comparison is very simple: the hash needs to be smaller than the required difficulty for that block.

For completeness sake, I’ve added the actual implementation of the difficulty-function here:

```scala
/**
  * This is the difficulty function used by naivecoin,
  * which raises the difficulty based on the block-index.
  *
  * The difficulty goes up every X blocks.
  */
object NaiveCoinDifficulty extends DifficultyFunction {
  val BASE_DIFFICULTY: BigInt = BigInt("f" * 64, 16)
  val EVERY_X_BLOCKS = 5
  val POW_CURVE = 5

  override def apply(block: Block): BigInt = {
    val blockSeriesNumber = ((BigInt(block.index) + 1) / EVERY_X_BLOCKS ) + 1
    val pow = blockSeriesNumber.pow(POW_CURVE)

    BASE_DIFFICULTY / pow
  }
}

/**
  * A difficultyfunction that will never succeed
  */
object ImpossibleDifficulty extends DifficultyFunction {
  override def apply(block: Block): BigInt = 0
}

/**
  * A difficulty function that will always succeed
  */
object NoDifficulty extends DifficultyFunction {
  override def apply(b: Block): BigInt = NaiveCoinDifficulty.BASE_DIFFICULTY
}
```

## BlockMessages

When we need to do this much work to add a new Block to the chain, it doesn’t make much sense to have a single string as the payload for a Block. I extended the original definitions a bit to now give each message a unique ID and to have a block contain a sequence of messages:

```scala
case class BlockMessage( data: String, id: String = UUID.randomUUID().toString)

object GenesisBlock extends Block(0, BigInt(0), 1497359352, Seq(BlockMessage("74dd70aa-2ddb-4aa2-8f95-ffc3b5cebad1","Genesis block")), 0,
  BigInt("2b33684ac1ce0a93a54410de84d4114a3882362bcec35a2a9b588630811ae92b", 16))

case class Block(index: Long, previousHash: BigInt, timestamp: Long, messages: Seq[BlockMessage], nonse: Long, hash: BigInt)
```

The unique ID allows us to check if a message is already in the chain or not. In [section\_title](#_mining_for_real) I’ll explain why this is important.

## Mining for real

In the original naivechain implemented, whichever node received the request to add a message to the chain would simply add the message and then broadcast the result. Now that we’ve added proof of work, this is no longer the best strategy to follow.

When a request to add a new message to the chain comes in, the node will try to mine a block for it itself, but in the mean time it will also forward the request to all its peers. Each of the peers will also start trying to mine a valid new block. Whoever finds the block first will then forward it to all peers, which at that point look which messages are in the new block and stop all mining attempts for that block.

If a new request to add a block to the chain comes in while we are still mining old blocks, we simply start a new miner for *all the outstanding messages, including the new message*.

The moment a new block is added to the blockchain, we stop all our open miners, since none of them would produce a valid result anymore at that point. We then start a single new miner for all the outstanding messages.

Let’s look at the `MiningWorker` class first:

```scala
object MiningWorker {
  case class MineBlock(blockChain: BlockChain, messages: Seq[BlockMessage], startNonse: Long = 0, timeStamp: Long = new java.util.Date().getTime )

  case class MineResult(block: Block)

  case object StopMining

  def props( reportBackTo: ActorRef ): Props = Props(classOf[MiningWorker], reportBackTo)
}

class MiningWorker(reportBackTo: ActorRef) extends Actor {

  val logger = Logger("MiningActor")

  var keepMining = true

  override def receive: Receive = {

    case StopMining => keepMining = false

    case MineBlock(blockChain, messages, startNonse, timeStamp) =>
      if ( startNonse == 0 ) {
        logger.debug(s"Starting mining attempt for ${messages.size} messages with index ${blockChain.latestBlock.index +1}")
      }

      startNonse.until(startNonse+100).flatMap { nonse =>
        blockChain.attemptBlock(messages, nonse)
      }.headOption match {
        case Some(block) =>
          logger.debug(s"Found a block with index ${block.index} after ${new java.util.Date().getTime - timeStamp} ms and ${block.nonse +1} attempts.")
          reportBackTo ! MineResult(block)
          context.stop(self)

        case None if keepMining => self ! MineBlock(blockChain, messages, startNonse + 100, timeStamp)

        case None =>
          logger.debug(s"Aborting mining for messages block ${blockChain.latestBlock.index +1}")
          context.stop(self)

      }
  }
}
```

It tries to add a block to the blockchain, incrementing the nonse in blocks of 100. After each 100 nonses, it checks if it should go on or not. This allows us to stop the worker when the work it is doing no longer has value (i.e. the blockchain has changed in the mean time).

When it successfully finds a block, it returns it in a `MineResult` message.

The `Mining` trait spawns a worker for each distinct set of `BlockMessage` objects to add to the chain.

```scala
object Mining {
  case class MineBlock(messages: Seq[BlockMessage] )

  case object BlockChainInvalidated
}

trait Mining {
  this: BlockChainCommunication with PeerToPeer with CompositeActor =>

  var miners: Set[ActorRef] = Set.empty
  var messages: Set[BlockMessage] = Set.empty

  def createWorker( factory: ActorRefFactory ): ActorRef = factory.actorOf(MiningWorker.props(self))

  receiver {
    case BlockChainInvalidated =>
      logger.debug("The blockchain has changed, stopping all miners.")
      miners.foreach( _ ! StopMining )

      messages = messages.filterNot( blockChain.contains(_))

      if ( ! messages.isEmpty ) {
        self ! MineBlock(messages.toSeq)
      }

    case MineBlock(requestMessages) =>

      logger.debug(s"Got mining request for ${requestMessages.size} messages with current blockchain index at ${blockChain.latestBlock.index}")
      val filtered = requestMessages.filterNot( blockChain.contains(_))

      //We only need to start mining if any new messages are in the message.
      if ( ! (filtered.toSet -- messages).isEmpty ) {
        messages ++= filtered

        //Tell all peers to start mining
        peers.foreach( p => p ! MineBlock(messages.toSeq))

        //Spin up a new actor to do the mining
        val miningActor = createWorker(context)
        context.watch(miningActor)
        miners += miningActor

        miningActor ! MiningWorker.MineBlock(blockChain, messages.toSeq)
      } else logger.debug("Request contained no new messages, so not doing anything.")

    case MineResult(block) =>
      if ( blockChain.validBlock(block) ) {
        logger.debug(s"Received a valid block from the miner for index ${block.index}, adding it to the chain.")
        messages = messages -- block.messages
        handleBlockChainResponse(Seq(block))
      } else logger.debug(s"Received an outdated block from the miner for index ${block.index}.")

    case Terminated(deadActor) =>
      miners -= deadActor
      logger.debug(s"Still running ${miners.size} miners for ${messages.size} messages")

      if ( miners.size == 0  && ! messages.isEmpty ) {
        val request = MineBlock(messages.toSeq)
        messages = Set.empty
        self ! request
      }
  }
}
```

When a `MiningWorker` returns a result, we see which messages are in the result. These messages are removed from the queue of messages to add. We then add the new `Block` to the chain through the same procedure as if the `Block` had come from a peer by calling the `handleBlockChainResponse` method. This leads to a `BlockChainInvalidated` event which causes all currently running workers to be stopped. We then start a single new worker for all the messages that currently aren’t in the chain yet.

# Conclusion

Blockchains are a very interesting way to implement distributed consensus and can be used for several purposes. Apart from the obvious crypto-currencies, they can for example be used to implement an audit-trail where it can be shown that items in the trail have not been forged, since it would take too much computing power to realistically forge them.

For me this was a valuable learning experience in how to use Akka to deal with all the concurrency issues that pop up when you’re spawning mining workers that then need to be stopped again.
