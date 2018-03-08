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

# Rationale
This BOLT provides an alternative to BOLT #11, and is intended to support a
wider range of use cases.

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
  This use case is covered by BOLT #11, with the following limitations:
  * The destination has to be specified as a node ID. This reveals the node ID
    of the payee to the payer, reducing privacy for the payee. In use cases
    where refunds are a possibility, the payer becomes the payee, so the same
    privacy reduction also happens in the opposite direction.
  * The maximum length of an embedded description of the purpose of the payment
    is 639 bytes. For longer descriptions a hash of the description is included
    instead, but BOLT #11 does not specify how the description itself is to be
    transmitted to the payer.
* One-off donations.
  Same as the first use case, except the payee does not specify the amount
  to be paid. The payer is free to choose an amount.
  With a sufficiently clear purpose of payment and a signature from the
  receiver, this, together with the payment pre-image can act as a proof of
  payment, available to the payer. If payers publish such proofs, this can help,
  for instance, to create public transparency in the minimum amount received
  through donations.
  Since the payment hash is specified, such an invoice can only be used once.
  This use case is covered by BOLT #11, with the same limitations as the first
  use case.
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
  It is possible to *manually* perform this procedure using BOLT #11, but there
  is no standardized protocol for *automatically* following this procedure.
* Recurring payments and streaming of micropayments.
  Same procedure as for the previous use case. In the case of streaming of
  micropayments, payer and payee might want to keep an open communication
  channel and re-use Lightning routing information for performance reasons.
  Examples of recurring payments would be monthly salary payments and
  automatic payments for subscriptions. Examples of streaming of micropayments
  would be paying for the use of a Wifi hot-spot or paying for watching a
  streaming movie on a website.
  As in the previous use case, it is possible with BOLT #11 to create a new
  invoice for every new payment, but it would be really inconvenient if there
  is no way to automate this.

## Overview
To eliminate the limitations of BOLT11 in these use cases, this BOLT specifies a
protocol where payee gives a URL to one or more potential payers. Payers can use
the URL to open a communication channel to payee to request invoices.
The same URL can be accessed multiple times and can be used by multiple payers
for multiple payments. The URL will typically be short enough to fit in a QR
code.

In cases where anonymity of the payee has to be preserved, an URL scheme has to
be used which protects this anonymity. An example of this would be an URL that
points to a TOR hidden service.

In order to not reveal the payee node ID to the payer, payee can send an invoice
to payer which has a partial onion route as destination instead of a node ID.
The partial onion route ends at the payee, but it can start at some other node.
All the payer has to do is prepend this partial route with extra hops, so that
the complete route starts at the payer. Note that this makes the invoice
considerably larger; this would have been infeasible in BOLT #11.

Similarly, allowing for much larger description fields than in BOLT #11 is not a
problem anymore.

To allow for convenient and potentially anonymous refunds, the communication
channel can be kept open, and the payer can send a refund invoice through the
same communication channel.

Finally, the communication channel may be used to exchange routing information
between payer and payee, potentially making it easier to find a route.

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

# URL specification
TODO
* Support for the same address types as in BOLT #7 (incl. TOR)
* Inclusion of the public key?
* Optional inclusion of an invoice ID, for replacement of BOLT #11 functionality
* Fall-back modes? E.g. on-chain payment

        payer                            payee
          |<-- URL ------------------------|
          |    |- hostname                 |
          |    |- port       (optional)    |
          |    |- pubkey     (optional?)   |
          |    |- Invoice ID (optional)    |
          |
        Connect

# Protocol specification
TODO
* Feature bits, for extensibility
* Maybe let this be an extension to the regular peer protocol, so a node only
  has to keep a single port open
* DoS protection

        payer                            payee
          |--- Get invoice --------------->|
          |    |- amount        (optional) |
          |    |- purpose       (optional) |
          |    |- refund pubkey (optional) |
          |    |- expiry time   (optional) |
          |    |- Invoice ID    (optional) |
          |                                |
          |<-- Invoice (signed) -----------|
          |    |- amount        (optional) |
          |    |- purpose       (optional) |
          |    |- refund pubkey (optional) |
          |    |- payment hash             |
          |    |- expiry time              |
          |                                |
          |--- Get destination ----------->|
          |    |- payment hash             |
          |                                |
          |<-- Destination ----------------|
          |    |- partial onion route      |
          |                                |
        Start payment

Repeat if desired, or close the connection afterwards.
Payee can also ask for an invoice; typically in the case of a refund: a refund
invoice should be signed with the refund key so that it invalidates the original
invoice as proof of payment.

Any request for an invoice can be denied.


# Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
