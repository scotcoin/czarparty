# The Czarparty Protocol

## Summary

Czarparty is a suite of financial tools in a protocol built on top of the
Czarcoin blockchain and using the blockchain as a service for the reliable
publication and timestamping of its messages.

The reference implementation is `czarpartyd`, which is hosted at
<https://github.com/Czarcoin/czarpartyd>.


## Transactions

Every Czarparty message has the following identifying features:
* One source address
* One destination address
* A quantity of czarcoins sent from the source to the destination, if it exists.
* A fee, in czarcoins, paid to the czarcoin miners who include the transaction in
  a block.
* Up to 80 bytes of ‘data’, imbedded in specially constructed transaction
  outputs.

Every Czarcoin transaction carrying a Czarparty transaction has the following
possible outputs: a single destination output, zero or more data outputs, and
optional change outputs. The first output before the first data output is the
destination output. Change outputs (outputs after the last data output) have no
importance to Czarparty. All data outputs must appear in direct succession.

For identification purposes, every Czarparty transaction’s ‘data’ field is
prefixed by the string ‘CZRPARTY’, encoded in UTF‐8. This string is long enough
that transactions with outputs containing pseudo‐random data cannot be mistaken
for containing valid Czarparty transaction data. In testing (i.e. using the
TESTCOIN Czarparty network on any blockchain), this string is ‘XX’.

Czarparty data may be stored in three different types of outputs, or in some
mixtures of those formats. Multi‐signature data outputs are one‐of‐two outputs
where the first public key is that of the sender, so that the value of the
output is redeemable, and the second public key encodes the data, zero‐padded
and prefixed with a length byte.
	`OP_RETURN` data output format
	pay‐to‐pubkeyhash data output format
		encryption

The existence of the destination output, and the significance of the size of
the Czarcoin fee and the Czarcoins transacted, depend on the Czarparty message
type, which is determined by the four bytes in the data field that immediately
follow the identification prefix. The rest of the data have a formatting
specific to the message type, described in the source code.

Every Czarparty transaction must, moreover, have a unique and unambiguous
source address: all of the inputs to a Czarcoin transaction which contains a
Czarparty transaction must be the same—the unique source of the funds in the
Czarcoin transaction is the source of the Czarparty transaction within.

The source and destination of a Czarparty transaction are Czarcoin addresses,
and any Czarcoin address may receive any Czarparty asset (and send it, if it
owns any).

All messages are parsed in order, one at a time, ignoring block boundaries.

Orders, bets, order matches and bet matches are expired at the end of blocks.

* Currently only Pay‐to‐Pubkey‐Hash addresses are supported.


## Non‐Czarparty transactions

czarpartyd supports the construction of two kinds of transactions that are
not themselves considered Czarparty transactions:

* CZR sends
* CZR dividends to Czarparty assets

Neither of these two transactions is constructed with a data field, and in the
latter, multiple ‘destination’ outputs are used.


## mempool transactions

Always have block index = 9999999 (`config.MEMPOOL_BLOCK_INDEX`).

DB changes never persist across sessions.

No matching for orders or bets.


## Assets

All assets except CZR and XZR have the following properties:

* Asset name
* Asset ID
* Description
* Divisiblity
* Callability
* Call date (if callable)
	* may be delayed with later issuances
* Call price (if callable) (non‐negative)
	* may be increased with later issuances


Asset names are strings of uppercase ASCII characters that, when encoded as a
decimal integer, are greater than 26^3 and less than or equal to 256^8: all
asset names, other than ‘CZR’ and ‘XZR’ must be at least four letters long;
asset names may not begin with the character ‘A’. Thus, some thirteen‐character
asset names are valid, but no fourteen‐character names are.

Assets may be either divisible or indivisible, and divisible assets are
divisible to eight decimal places. Assets also come with descriptions, which
may be changed at any time.

Assets may be ‘callable’: callable assets may be forcibly ‘called back’ by
their present issuer, after their *call date*, for their *call price* (in XZR),
these values being set at the time of the asset’s first issuance.

Callable assets may be called back after their call date has been first passed
by a block in the blockchain.

Call prices are specified to six decimal place of precision, and are a ratio of
XZR and the unit (not czarcoins) of the callable asset.



## Quantities, Prices, Fractions

* max int

* oversend, overbet, overorder
	* not czrpay, callback (impossible, because of rounding), issuance (fragile!), dividend (?!)



## Expirations

* max expiration times

* at beginning of block (before txes are parsed)


## Transaction Statuses

*Offers* (i.e. orders and bets) are given a status `filled` when their
`give_remaining`, `get_remaining`, `wager_remaining`, `counterwager_remaining`,
`fee_provided_remaining` or `fee_required_remaining` are no longer positive
quantities.

Because order matches pending CZR payment may be expired, orders involving
Czarcoin cannot be filled, but remain always with a status `open`.


## Message Types

* Send
* Order
* CZRPay
* Issue
* Broadcast
* Bet
* Dividend
* Burn
* Cancel
* Callback


### Send

A **send** message sends a quantity of any Czarparty asset from the source
address to the destination address. If the sender does not hold a sufficient
quantity of that asset at the time that the send message is parsed (in the
sequence of transactions), then the send is filled partially.

czarpartyd supports sending czarcoins, for which no data output is used.


### Order

An ‘order’ is an offer to *give* a particular quantity of a particular asset
and *get* some quantity of some other asset in return. No distinction is drawn
between a ‘buy order’ and a ‘sell order’. The assets being given are escrowed
away immediately upon the order being parsed. That is, if someone wants to give
1 XZR for 2 CZR, then as soon as he publishes that order, his balance of XZR is
reduced by one.

When an order is seen in the blockchain, the protocol attempts to match it,
deterministically, with another open order previously seen. Two matched orders
are called a ‘order match’. If either of a order match’s constituent orders
involve Czarcoin, then the order match is assigned the status ‘pending’ until
the necessary CZRPay transaction is published. Otherwise, the trade is
completed immediately, with the protocol itself assigning the participating
addresses their new balances.

All orders are *limit orders*: an asking price is specified in the ratio of how
much of one would like to get and give; an order is matched to the open order
with the best price below the limit, and the order match is made at *that*
price. That is, if there is one open order to sell at .11 XZR/ASST, another
at .12 XZR/ASST, and another at .145 XZR/CZR, then a new order to buy at .14
XZR/ASST will be matched to the first sell order first, and the XZR and CZR
will be traded at a price of .11 XZR/ASST, and then if any are left, they’ll be
sold at .12 XZR/ASST. If two existing orders have the same price, then the one
made earlier will match first.

All orders allow for partial execution; there are no all‐or‐none orders. If, in
the previous example, the party buying the czarcoins wanted to buy more than the
first sell offer had available, then the rest of the buy order would be filled
by the latter existing order. After all possible order matches are made, the
current (buy) order is listed as an open order itself. If there exist multiple
open orders at the same price, then order that was placed earlier is matched
first.

Open orders expire after they have been open for a user‐specified number of
blocks. When an order expires, all escrowed funds are returned to the parties
that originally had them.

Order Matches waiting for Czarcoin payments expire after twenty blocks
(originally otherwise); the constituent orders are replenished.

In general, there can be no such thing as a fake order, because the assets that
each party is offering are stored in escrow. However, it is impossible to
escrow czarcoins, so those attempting to buy czarcoins may ask that only orders
which pay a fee in czarcoins to Czarcoin miners be matched to their own. On the
other hand, when creating an order to sell czarcoins, a user may pay whatever
fee he likes. Partial orders pay partial fees. These fees are designated in the
code as `fee_required` and `fee_provided`, and as orders involving CZR are
matched (expired), these fees (required and provided) are debited
(sometimes replenished), in proportion to the fraction of the order that is
matched. That is, if an order to sell 1 CZR has a `fee_provided` of 0.01 CZR (a
1%), and that order matches for 0.5 CZR initially, then the
`fee_provided_remaining` for that order will thenceforth be 0.005 CZR.
*Provided* fees, however, are not replenished upon failure to make CZR
payments, or their anti‐trolling effect would be voided.

Payments of czarcoins to close order matches waiting for czarcoins are done with
the a **CZRpay** message, which stores in its data field only the string
concatenation of the transaction hashes which compose the Order Match which it
fulfils.


### Issue

Assets are issued with the **issuance** message type: the user picks a name and
a quantity, and the protocol credits his address accordingly. The asset name
must either be unique or be one previously issued by the same address. When
re‐issuing an asset, that is, issuing more of an already‐issued asset, the
divisibilities and the issuing address must match.

The rights to issue assets under a given name may be transferred to any other
address. 

Assets may be locked irreversibly against the issuance of further quantities
and guaranteeing its holders against its inflation. To lock an asset, set the
description to ‘LOCK’ (case‐insensitive).

Issuances of any non‐zero quantity, that is, issuances which do not merely
change, e.g., the description of the asset, involve a debit (and destruction)
of now 0.5 XZR.


### Broadcast

A **broadcast** message publishes textual and numerical information, along
with a timestamp, as part of a series of broadcasts called a ‘feed’. One feed
is associated with one address: any broadcast from a given address is part of
that address’s feed. The timestamps of a feed must increase monotonically.

Bets are made on the numerical values in a feed, which values may be the prices
of a currency, or parts of a code for describing discrete possible outcomes of
a future event, for example. One might describe such a code with a text like,
‘US QE on 2014-01-01: dec=1, const=2, inc=3’ and announce the results with ‘US
QE on 2014-01-01: decrease!’ and a value of 1. The schema for more complicated
bets may be published off‐chain.

The publishing of a single broadcast with a textual message equal to ‘LOCK’
(case‐insensitive) locks the feed, and prevents it both from being the source
of any further broadcasts and from being the subject of any new bets. (If a
feed is locked while there are open bets or unsettled bet matches that refer to
it, then those bets and bet matches will expire harmlessly.)

A feed is identified by the address which publishes it.

Broadcasts with a value of -2 cancel all open bets on the feed.
Broadcasts with a value of -3 cancel all pending bet matches on the feed. (This is equivalent to waiting for two weeks after the deadline.)
Broadcasts with any other negative value are ignored for the purpose of bet settlement, but they still update the last broadcast time.


### Bet

There are (currently) two kinds of **bets**. The first is a wager that the
value of a particular feed will be equal (or not equal) to a certain value —
the *target value* — at the *deadline*. The second is a contract for difference
with a definite settlement date. Both simple Equal/NotEqual Bets and Bull/Bear
CFDs have their wagers put in escrow upon being matched, and they are settled
when the feed that they rely on passes the deadline. CFDs, actually, may be
force‐liquidated before then if the feed value moves so much that the escrow is
exhausted.

CFDs may be leveraged, and their leverage level is specified with 5040 equal to
the unit and stored as an integer: a leverage level of 5040 means that the
wager should be leveraged 1:1; a level of 10080 means that a one‐point increase
in the value of a feed entails a two‐point increase (decrease) in the value of
the contract for the bull (bear).

CFDs have no target value, and Equal/NotEqual Bets cannot be leveraged. However,
for two Bets to be matched, their leverage levels, deadlines and target values
must be identical. Otherwise, they are matched the same way that orders are,
except a Bet’s *odds* are the multiplicative inverse of an order’s price
(odds = wager/counterwager): each Bet is matched, if possible, to the open
Bet with the highest odds, as much as possible.

Target values must be non‐negative, and Bet Matches (contracts) are not
affected by broadcasts with a value of -1.

Bets cannot have a deadline later that the timestamp of the last broadcast of
the feed that they refer to.

Bets expire the same way that orders do, i.e. after a particular number of
blocks. Bet Matches expire 2016 blocks after a block is seen with a block
timestamp after its deadline.

Betting fees are proportional to the initial wagers, not the earnings. They are
taken from, not added to, the quantities wagered.

* Because of the block time, and the non‐deterministic way in which
  transactions are ordered in the blockchain, all contracts must be not be
  incrementally settled, but the funds in question must be immediately put into
  escrow, and there must be a settlement date. Otherwise, one could see a price
  drop coming, and ‘fight’ to hide the funds that were going to be deducted.

Feed fees are deducted from the final settlement amount.


### Dividend

A dividend payment is a payment of some quantity of any Czarparty asset
(including CZR) to every holder of a an asset (except CZR or XZR) in proportion
to the size of their holdings. Dividend‐yielding assets may be either divisible
or indivisible. A dividend payment to any asset may originate from any address.
The asset for dividend payments and the assets whose holders receive the
payments may be the same. Czarcoin dividend payments do not employ the
Czarparty protocol and so are larger and more expensive (in fees) than all
other dividend payments.


### Burn

Balances in Czarparty’s native currency, ‘XZR’, will be initialised by
‘burning’ czarcoins in miners’ fees during a particular period of time using the
a **burn** message type. The number of XZR earned per czarcoin is calculated
thus: 

	XZR_EARNED = CZR_BURNED * (1000 * (1 + .5 * ((END_BLOCK - CURRENT_BLOCK) / (END_BLOCK - START_BLOCK))

`END_BLOCK` is the block after which the burn period is over (**block #283810**) and
`START_BLOCK` is the block with which the burn period begins (**block #278310**). The earlier the
burn, the better the price, which may be between 1000 and 1500 XZR/CZR.

Burn messages have precisely the string ‘ProofOfBurn’ stored in the
`OP_RETURN` output.

* new data‐less burn

* burn period is over


### Cancel

Open offers may be cancelled, which cancellation is irrevocable.

A *cancel* message contains only the hash of the Czarcoin transaction that
contains the order or bet to be cancelled. Only the address which made an offer
may cancel it.


### Callback

*Callbacks are currently disabled on Czarparty mainnet, as the logic by
which they are parsed is currently undergoing revision and testing.*
