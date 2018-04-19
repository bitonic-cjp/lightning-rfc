# BOLT #12: Payment Protocol for Lightning Payments

A payer/payee communication protocol for organizing various types of payments
over the Lightning network.

# Table of Contents

  * [Status](#status)
  * [Rationale](#rationale)
  * [URL specification](#url-specification)
  * [Protocol specification](#protocol-specification)
  * [Authors](#authors)

# Status
This BOLT is an unfinished draft, and will be changed in the future.

# Introduction
This BOLT provides an alternative to [BOLT #11](11-payment-encoding.md), and is
intended to support a wider range of use cases.

## Use cases
The following use cases were identified:

* One-off payments in exchange for goods or services.
  Payee writes an invoice, specifying the amount to be paid, the purpose of the
  payment, the destination to which funds have to be sent, and the payment hash.
  With a sufficiently clear purpose of payment and a signature from the
  receiver, this, together with the payment pre-image can act as a proof of
  payment, available to the payer. This can help, for instance, for dispute
  settlement on who owns certain goods.
  Since the payment hash is specified, such an invoice can only be used once.
  This use case is covered by [BOLT #11](11-payment-encoding.md), with the
  following limitations:
  * The destination has to be specified as a node ID. This reveals the node ID
    of the payee to the payer, reducing privacy for the payee. In use cases
    where refunds are a possibility, the payer becomes the payee, so the same
    privacy reduction also happens in the opposite direction.
  * The maximum length of an embedded description of the purpose of the payment
    is 639 bytes. For longer descriptions a hash of the description is included
    instead, but [BOLT #11](11-payment-encoding.md) does not specify how the
    description itself is to be transmitted to the payer.
* One-off donations.
  Same as the first use case, except the payee does not specify the amount
  to be paid. The payer is free to choose an amount.
  With a sufficiently clear purpose of payment and a signature from the
  receiver, this, together with the payment pre-image can act as a proof of
  payment, available to the payer. If payers publish such proofs, this can help,
  for instance, to create public transparency in the minimum amount received
  through donations.
  Since the payment hash is specified, such an invoice can only be used once.
  This use case is covered by [BOLT #11](11-payment-encoding.md), with the same
  limitations as the first use case.
* Payments initiated by payer.
  Payer contacts payee to perform a payment and sends a description of the
  purpose of the payment and the amount to be paid.
  Payee writes an invoice for the payment, embedding the payer-supplied
  information in such a way that it is clear the payee *received* the
  information, but does not necessarily *agree* to it. Payment continues in the
  same way as in the first use case.
  This is useful for payments that were not expected by the payee, for instance
  spontaneous donations to a payee who has not set up any other system for
  accepting donations. It is also useful in cases where it would be inconvenient
  for the payee to create a new invoice for every new transaction. For instance,
  if the user of an USD/BTC exchange places an order for buying BTC, and wants
  the BTC immediately paid out through Lightning, the user doesn't want to
  have to manually write an invoice and submit it to the exchange at the moment
  the order is executed.
  It is possible to *manually* perform this procedure using
  [BOLT #11](11-payment-encoding.md), but there is no standardized protocol
  for *automatically* following this procedure.
* Recurring payments and streaming of micropayments.
  Same procedure as for the previous use case. In the case of streaming of
  micropayments, payer and payee might want to keep an open communication
  channel and re-use Lightning routing information for performance reasons.
  Examples of recurring payments would be monthly salary payments and
  automatic payments for subscriptions. Examples of streaming of micropayments
  would be paying for the use of a WiFi hot-spot or paying for watching a
  streaming movie on a website.
  As in the previous use case, it is possible with
  [BOLT #11](11-payment-encoding.md) to create a new invoice for every new
  payment, but it would be really inconvenient if there is no way to automate
  this.

## Overview
To eliminate the limitations of [BOLT #11](11-payment-encoding.md) in these use
cases, this BOLT specifies a protocol where payee gives a URL to one or more
potential payers. Payers can use the URL to open a communication channel to
payee to request invoices. The same URL can be accessed multiple times and can
be used by multiple payers for multiple payments. The URL will typically be short
enough to fit in a QR code.

In cases where anonymity of the payee has to be preserved, an URL scheme has to
be used which protects this anonymity. An example of this would be an URL that
points to a TOR hidden service.

In order to not reveal the payee node ID to the payer, payee can send an invoice
to payer which has a partial onion route as destination instead of a node ID.
The partial onion route ends at the payee, but it can start at some other node.
All the payer has to do is prepend this partial route with extra hops, so that
the complete route starts at the payer. Note that this makes the invoice
considerably larger; this would have been infeasible in
[BOLT #11](11-payment-encoding.md).

Similarly, allowing for much larger description fields than in
[BOLT #11](11-payment-encoding.md) is not a problem anymore.

To allow for convenient and potentially anonymous refunds, the communication
channel can be kept open, and the payer can send a refund invoice through the
same communication channel.

Finally, the communication channel may be used to exchange routing information
between payer and payee, potentially making it easier to find a route.


## Proof of Payment
A signed invoice, together with the corresponding payment preimage, can be used
by the payer as a proof of payment. The primary use case for a proof of payment
is to act as evidence in front of a dispute settlement provider, in case there
is disagreement between payer and payee on whether or not the payee is subject
to the obligations specified in the description field of an invoice. Scenarios
where either payer or payee is hacked, resulting in a theft of funds by the
hacker, are of particular interest:

* If the payee is hacked, and the payee's computers send an not-correctly-signed
  invoice to the payer, the payer should not perform the payment.
* If the payee is not hacked, sends out a correctly signed invoice but refuses
  to fulfill the obligations in the invoice despite being paid, the payer should
  be able to enforce the obligations in the invoice by showing the proof of
  payment.
* If the payee is hacked, and the payee's computers send a correctly-signed
  invoice to the payer (which sends funds to the hacker), this is
  indistinguishable from the previous case from the point of view of both payer
  and dispute settlement provider. Therefore, the end result should be the same.
  In other words: the payee is responsible for protecting his own computers
  against hackers. Note that, in order to prevent a hacker from profiting by
  sending out extremely unreasonable invoices to himself, a dispute settlement
  provider may place limits on how unreasonable the terms in an invoice can be,
  and order a refund instead, in case a too unreasonable invoice is disputed.
* If the payer is not hacked, does not pay the payee but still claims to have
  paid, and demands the payee to fulfill the obligations in a fake
  proof of payment, the proof of payment will not be correct: either the
  signature is not valid, or the preimage is not valid, or both. The payer
  should not be able to enforce the obligations in the invoice by showing the
  invalid proof of payment.
* If the payer is hacked, and the payer's computer accepts a fake invoice
  (which sends funds to the hacker), this is indistinguishable from the previous
  case from the point of view of the dispute settlement provider. Therefore, the
  end result should be the same. In other words: the payer is responsible for
  protecting his own computer against hackers.


## Public Key Infrastructure
In order to make a proof of payment useful, it must be possible for a dispute
settlement provider to verify the proof of payment. In particular, it must be
possible to verify that the payment was performed to one particular payee, who
agreed to deliver whatever was specified in the description in the invoice.

The invoice is digitally signed, but this is only useful if the digital
signature corresponds to a key that can be proven to be owned by the payee.
"Be proven" really means "there is proof that is accepted by the dispute
settlement provider". It would be good practice for dispute settlement providers
to specify in advance what their policies are regarding the acceptance of such
proofs.

Such proof will typically have to consist of a key certificate that consists of
a digital signature of the payee's public key by the key of a certificate
authority that is recognized by the dispute settlement provider.

These considerations lead to a quite centralized approach, with the risk that
centralized entities like the dispute settlement provider can impose arbitrary
demands on other participants, with the penalty that they will otherwise be
unable to participate in the system. However, competition *is* possible:
people excluded from participation by one dispute settlement provider can found
a new one and organize around it.

A different dispute settlement method would be one of a decentralized reputation
system, where the only direct penalty would be a loss of reputation. Such a
system would combine better with a PGP-style Web of Trust. Participants would
not only be required to verify each others' identities and sign keys, but also
to check whether the obligations in valid proofs of payments of others have been
fulfilled. It is not clear how all parts of such a system would function in
detail, but it does seem clear that offering only a single authentication path
for the key of the payee is insufficient.

Finally, there is the option of not doing any key authentication. A payer might
perform key pinning to detect future hacks, but not otherwise perform any
checks. Arguably, this makes the proof of payment useless for later dispute
settlement by a third party, but it still enables payer and payee to perform
payments.

A Web of Trust is the most generic PKI shape, and it supports all these modes
of operation. For instance, when using a single dispute settlement provider,
there needs to be a chain of trust from that provider to the certificate that
proves the link between the public key and the payee identity.

TBD is: how is information about the chain of trust transferred to the payer?
TLS style (embedded in the protocol) or PGP style (external, prior to the
transaction)?


## Transport Layer protocol
The protocol described here can be built on top of any transport layer protcol
that provides reliable bidirectional data streams, like TCP. However, selection
of an appropriate transport layer protocol *is* relevant, because it has an
impact on, for instance, the privacy of participants, and the possibility of
censorship. It is expected to be more important for this protocol than for the
peer communication protocol because it is expected to be easier to find
Lightning peers who support your uncommon transport protocol than it is to find
merchants who support your uncommon transport protocol *and* sell what you want
at a decent price.

Some examples of transport layer protocols that can be considered:

* TCP/IP:
  Offers little privacy for client nor server.
  IP addresses and host names may be taken down.
  A TOR-using client can connect to a TCP/IP server through a TOR exit node.
* TOR hidden services:
  Offers substantial privacy for client and server.
  Hidden service names can normally not be taken down, unless the operator is
  hacked or identified.
  A TCP/IP-using client can connect to a hidden service through a gateway like
  [onion.to](https://onion.to/).
* Bluetooth (e.g. RFCOMM):
  Only suitable for payments in physical proximity.
  Provides privacy and protection against being taken down by being local-only
  and temporary.


## Security considerations
Data exchanged over the communication channel has to be properly encrypted and
signed. Authenticity of the keys of the payee has to be verified by the payer.
This is necessary to prevent sending funds to an attacker who pretends to be
the payee.
Key authentication can be done by sending this key over another secure channel,
e.g. a HTTPS website or by physically handing it over, or through a suitable
public key infrastructure. The key can be handed over as part of the URL, at
the cost of making the URL longer. In some URL schemes, e.g. .onion addresses,
a suitable key may already intrinsically be part of the URL.

For refunds, it must be ensured that the payee of the refund is same as the
payer of the initial transaction. Also, the refund should invalidate the proof
of payment of the first transaction. This can be done by including a "refund"
public key in the signed invoice of the first payment, which must correspond to
the signature of the refund invoice.

The payee should never blindly copy the purpose of a payment from information
supplied by the payer: otherwise a malicious payer can trick the payee into
signing a "proof of payment" the payee never agreed to. It must always be clear
who is the author of the "purpose of payment" text.


# URI specification

## Format
A lnpay URI has the following format:

&lt;uri&gt; = lnpay:[&lt;pubkeyhash&gt;]?[id=&lt;id&gt;&]r0=&lt;host&gt;[&r1=&lt;host&gt;[...]]

### pubkeyhash
The HASH160 of a public key on Bitcoin's `secp256k1` curve.
Used for invoice signing and as trust root for the encryption.
TODO: encoding

### id
TODO: allowed characters

### host
TCP/IP:
&lt;host&gt; = tcp:{&lt;dnsname&gt;|&lt;ipv4&gt;|&lt;ipv6&gt;}[:&lt;portnumber&gt;]

TOR:
&lt;host&gt; = tor:&lt;TOR-pubkeyhash&gt;.onion[:&lt;portnumber&gt;]

Bluetooth:
&lt;host&gt; = bt:&lt;MAC&gt;[/&lt;ResourceName&gt;]

TODO: default port number (and Bluetooth resource name?).

## Usage
Typically, the payee generates an URI, and sends this URI to the payer, for
instance through a QR code, NFC, or a link on a website or in an e-mail.
However, it is not necessarily the payee who generates the URI. The URI is just
a means of bootstrapping a connection.

As soon as two parties are connected, they are peers, with the only (optional)
difference being that the URI-receiving party has received the pubkeyhash of the
URI-sending party.

The party writing the URI SHOULD be reachable on all hosts provided with the
r0, r1, ... arguments, for at least as long as needed to finish the transaction.

TODO
* Fall-back modes? E.g. on-chain payment, BOLT11

## Rationale

TODO


# Encryption layer

On top of the selected transport layer, an encryption layer is applied.
This encryption layer is identical to [BOLT #8](08-transport.md), except for
the following differences:
* Prior to following [BOLT #8](08-transport.md), the following data is exchanged:

  ```
  &lt;- s
  ```

  The responder sends its static public key to the initiator.
  If a `pubkeyhash` value is given in the URI, the initiator MUST verify
  that HASH160(s) equals the `pubkeyhash` value, and terminate the connection
  otherwise.
* `prologue` is the ASCII string `lnpay` instead of `lightning`

## Rationale

TODO


# Messaging layer

On top of the encryption layer, a messaging layer is applied.
This messaging layer is identical to [BOLT #1](01-messaging.md), with the
exception of the set of message types. The following message groups are defined
instead of the ones listed in [BOLT #1](01-messaging.md):
* Setup & Control (types `0`-`31`):
  identical to the message types in [BOLT #1](01-messaging.md).
  Global feature flags are identical to those defined in [BOLT #9](09-features.md).
  There are currently no local feature flags.
* Identification & Authentication (types `32768`-`33023`):
  messages related to identification and authentication
  (described below)
* Invoice management (types `33024`-`33279`):
  Messages related to requesting, sending and updating invoices.
  (described below)
* Routing (types `33280`-`33536`):
  Messages related to setting up a route.
  (described below)


## Rationale

TODO


# Identification & Authentication

TODO
* X.509 data
* Optional behavior regarding external WoT, certificate pinning


# Invoice management

TODO
* Full-featured replacement invoices, signed by both parties
* To Null state if contract is fulfilled (optional)
* On reconnect, specify which invoice to talk about

        payer                            payee
          |--- Get invoice --------------->|
          |    |- amount+curr.  (optional) |
          |    |- purpose MIME  (optional) |
          |    |- purpose       (optional) |
          |    |- refund pubkey (optional) |
          |    |- expiry time   (optional) |
          |    |- Invoice ID    (optional) |
          |                                |
          |<-- Invoice (signed) -----------|
          |    |- amount+curr.             |
          |    |- purpose MIME  (optional) |
          |    |- purpose       (optional) |
          |    |- purpose author           |
          |    |- refund pubkey (optional) |
          |    |- payment hash             |
          |    |- expiry time              |

Repeat if desired, or close the connection afterwards.
Payee can also ask for an invoice; typically in the case of a refund: a refund
invoice should be signed with the refund key so that it invalidates the original
invoice as proof of payment.

Any request for an invoice can be denied.


# Routing

TODO
* Offer multiple partial onion routes, in case some fail

        payer                            payee
          |--- Get destination ----------->|
          |    |- payment hash             |
          |                                |
          |<-- Destination ----------------|
          |    |- partial onion route      |


TODO
* DoS protection


# Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
