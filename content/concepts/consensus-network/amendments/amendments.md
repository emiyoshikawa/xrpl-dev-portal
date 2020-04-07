# Amendments

[Introduced in: rippled 0.31.0][]

The Amendments system provides a means of introducing new features to the decentralized XRP Ledger network without causing disruptions. The amendments system works by utilizing the core consensus process of the network to approve any changes by showing continuous support before those changes go into effect. An amendment normally requires **80% support for two weeks** before it can apply.

When an Amendment has been enabled, it applies permanently to all ledger versions after the one that included it. You cannot disable an Amendment, unless you introduce a new Amendment to do so.

For a complete list of known amendments, their statuses, and IDs, see: [Known Amendments](known-amendments.html).

## Background

The [consensus process](consensus.html) depends on all servers in the network reaching exactly the same results when processing the same set of transactions. If some validators process transactions in a different way than others, this could lead to delays in validation or even full outages as the network is unable to agree on a consensus outcome.

Any changes to transaction processing could cause servers to build a different ledger from the same set of transactions. If some _validators_ (`rippled` servers [participating in consensus](rippled-server-modes.html#reasons-to-run-a-validator)) have upgraded to a new version of the software while other validators use the old version, this could cause anything from minor inconveniences to full outages. In the minor case, a minority of servers spend more time and bandwidth fetching the actual consensus ledger because they cannot build it using the transaction processing rules they already know. In the worst case, [the consensus process][] might be unable to validate new ledger versions because servers with different rules could not reach a consensus on the exact ledger to build.

Amendments solve this problem, so that new features can be enabled only when enough validators support those features.

Users and businesses who rely on the XRP Ledger can also use Amendments to provide advance notice of changes in transaction processing that might affect their business. However, API changes that do not impact transaction processing or [the consensus process][] do not need Amendments.

[the consensus process]: consensus.html


## About Amendments

An amendment is a fully-functional feature or change, waiting to be enabled by the peer-to-peer network as a part of the consensus process. A `rippled` server that wants to use an amendment has code for two modes: without the amendment (old behavior) and with the amendment (new behavior).

Amendments are only appropriate for changes to **transaction processing**. For changes that do not affect servers' ability to build the same exact ledger from the same set of inputs, amendments are unnecessary and undesirable. For example, API changes do not require an amendment.

Every amendment has a unique identifying hex value and a short name. The short name is for human use, and is not used in the amendment process. Two servers can support the same amendment ID while using different names to describe it. An amendment's name is not guaranteed to be unique.

By convention, Ripple's developers use the SHA-512Half hash of the amendment name as the amendment ID.


## Amendment Process

In the amendment process, validators express their preferences for specific amendments, the network tracks when the amendment crosses the 80% threshold for support, and the amendment becomes permanently enabled if it sustains that support for two weeks.

### Amendment Lifecycle

![Amendment lifecycle: Open for Voting ←→ Expected → Enabled (final); or, Open for Voting → Rejected (final)](img/amendment-process.png)

1. A new amendment is said to be _Open for Voting_ when it has a complete implementation in a stable release of an XRP Ledger server such as `rippled`.
2. An amendment becomes _Expected_ if at least 80% of trusted validators vote in favor of an amendment. An Expected amendment has a target timestamp two weeks after it gained 80% support.
3. While an amendment is _Expected_, if support drops below 80%, the amendment returns to _Open for Voting_ status. If it regains support of 80% of validators, it can return to being Expected.
4. An Expected amendment becomes permanently _Enabled_ if  sustains support of at least 80% of validators through its target timestamp. It becomes recorded in the ledger and can never be disabled.

The XRP Ledger protocol does not include a way to permanently reject a proposed amendment. However, if an amendment has not been enabled and new XRP Ledger server versions do not have an implementation for the proposed amendment, it can be assumed that the amendment won't be enabled in the future. If this happens, the amendment is said to be _Rejected_.

### Technical Details

Validators vote for amendments every time there is a _flag ledger_, approximately every 15 minutes in typical circumstances. A flag ledger is defined as a ledger version whose [ledger index][] is evenly divisible by 256. Votes are sent and tallied in the same way as [Fee Voting](fee-voting.html), over the course of the ledger versions before and after the flag ledger:

1. **Flag ledger - 1:** Alongside their [validations for this ledger version](consensus.html#validation), validators send votes in favor of amendments they would like to see enabled. If a validator does not vote in favor of an amendment, that is the same as voting against the amendment.

2. **Flag ledger:** This ledger has no special contents. However, during this time, servers look at the votes of their trusted validators and evaluate which proposed amendments, if any, have crossed an important threshold.
3. **Flag ledger +1:** For each amendment that crossed an important threshold, the server inserts an [EnableAmendment pseudo-transaction][] into this ledger. The pseudo-transaction's `Flags` field indicates what type of event occurred:

    - `tfGotMajority` flag: Support for the amendment has increased to at least 80% of trusted validators.
    - `tfLostMajority` flag: Support for the amendment has decreased to less than 80% of trusted validators.
    - No flags: The amendment has maintained ≥80% support continuously for at least two weeks. Enable the amendment.

4. **Flag ledger +2:** If an amendment was enabled, its rules take effect starting with this ledger version.

A server only inserts an [EnableAmendment pseudo-transaction][] to enable an amendment if all of the following conditions are met:

- The amendment has not already been enabled.
- A previous ledger includes an EnableAmendment pseudo-transaction for this amendment with the `tfGotMajority` flag.
- The previous ledger in question is an ancestor of the current ledger.
- The previous ledger in question has a close time that is at least **two weeks** before the close time of the latest flag ledger.
- There are no EnableAmendment pseudo-transactions for this amendment with the `tfLostMajority` flag in the consensus ledgers between the `tfGotMajority` pseudo-transaction and the current ledger.

Theoretically, a `tfLostMajority` EnableAmendment pseudo-transaction could be included in the same ledger as the pseudo-transaction to enable an amendment. In this case, the pseudo-transaction with the `tfLostMajority` pseudo-transaction has no effect.

### Retiring Enabled Amendments
[Introduced in: rippled 1.5.0][]

After an amendment has been enabled, it cannot be disabled. However, XRP Ledger servers still have the implementation for the pre-amendment behavior. This code applies when evaluating older ledger versions from before the amendment became enabled. These alternate code paths also apply on [parallel networks](parallel-networks.html) that have not enabled the same set of amendments.

There is complexity and maintenance burden involved in supporting code for processing transactions with and without every possible combination of amendments. Therefore, Ripple has proposed a policy for "retiring" enabled amendments after sufficient time has passed:

1. After at least **two years** have passed since the amendment became enabled, remove the code for processing transactions without the amendment. This is called _unconditionalizing_ the amendment.

    When an amendment has become unconditionalized, the XRP Ledger server cannot accurately replay historical ledgers from before the amendment, because the server no longer has the implementation of the legacy behavior. Any cases where the amendment would have applied produce different results. To accurately replay historical transactions from before the amendment became enabled, you must compile a version of the XRP Ledger server that is contemporary with the historical ledger.

2. Introduce a new "roll-up" amendment to fully retire unconditionalized amendments. The only effect of this amendment is to remove unconditionalized amendments from the lists of known amendments in the ledger and the [feature method][].

    Ripple proposes introducing a new roll-up amendment annually, in the next stable release after unconditionalizing the amendments to retire. The first such amendment will be `featureRetire2017Amendments`, and it retires amendments that were enabled in 2017 or earlier.


## Amendment Voting

Each version of `rippled` is compiled with a list of known amendments and the code to implement those amendments. By default, `rippled` supports known amendments and opposes unknown amendments. Operators of `rippled` validators can [configure their servers](#configuring-amendment-voting) to explicitly support or oppose certain amendments, even if those amendments are not known to their `rippled` versions.

To become enabled, an amendment must be supported by at least 80% of trusted validators continuously for two weeks. If support for an amendment goes below 80% of trusted validators, the amendment is temporarily rejected. The two week period starts over if the amendment regains support of at least 80% of trusted validators. (This can occur if validators vote differently, or if there is a change in which validators are trusted.) An amendment can gain and lose a majority any number of times before it becomes permanently enabled. An amendment cannot be permanently rejected, but it becomes very unlikely for an amendment to become enabled if new versions of `rippled` do not have the amendment in their known amendments list.

As with all aspects of the consensus process, amendment votes are only taken into account by servers that trust the validators sending those votes. At this time, Ripple (the company) recommends only trusting the validators on the validator list that Ripple publishes at <https://vl.ripple.com>. For now, trusting only those validators is enough to coordinate with Ripple on releasing new features.

### Configuring Amendment Voting

You can temporarily configure an amendment using the [feature method][]. To make a persistent change to your server's support for an amendment, change your server's `rippled.cfg` file.

Use the `[veto_amendments]` stanza to list amendments you do not want the server to vote for. Each line should contain one amendment's unique ID, optionally followed by the short name for the amendment. For example:

```
[veto_amendments]
C1B8D934087225F509BEB5A8EC24447854713EE447D277F69545ABFA0E0FD490 Tickets
DA1BD556B42D85EA9C84066D028D355B52416734D3283F85E216EA5DA6DB7E13 SusPay
```

Use the `[amendments]` stanza to list amendments you want to vote for. (Even if you do not list them here, by default a server votes for all the amendments it knows how to apply.) Each line should contain one amendment's unique ID, optionally followed by the short name for the amendment. For example:

```
[amendments]
4C97EBA926031A7CF7D7B36FDE3ED66DDA5421192D63DE53FFB46E43B9DC8373 MultiSign
42426C4D4F1009EE67080A9B7965B44656D7714D104A72F9B4369F97ABF044EE FeeEscalation
```


### Amendment Blocked

When an amendment gets enabled for the network after the voting process, servers running earlier versions of `rippled` that do not know about the amendment become "amendment blocked" because they no longer understand the rules of the network. Servers that are amendment blocked:

* Cannot determine the validity of a ledger
* Cannot submit or process transactions
* Do not participate in the consensus process
* Do not vote on future amendments

Becoming amendment blocked is a security feature to protect applications that depend on XRP Ledger data. Rather than guessing and maybe misinterpreting a ledger after new rules have been applied, `rippled` reports that it does not know the state of the ledger because it does not know how the amendment works.

The amendments that a `rippled` server is configured to vote for or against have no impact on whether the server becomes amendment blocked. A `rippled` server always follows the set of amendments enabled by the rest of the network, to the extent possible. A server only becomes amendment blocked if an enabled amendment is not included in the amendment definitions compiled into the server's source code -- in other words, if the amendment is newer than the server.

If your server is amendment blocked, you must [upgrade to a new version](install-rippled.html) to sync with the network.


#### How to Tell If Your `rippled` Server Is Amendment Blocked

One of the first signs that your `rippled` server is amendment blocked is an `amendmentBlocked` error that is returned [when you submit a transaction](submit.html). Here's an example `amendmentBlocked` error:

```
{
   "result":{
      "error":"amendmentBlocked",
      "error_code":14,
      "error_message":"Amendment blocked, need upgrade.",
      "request":{
         "command":"submit",
         "tx_blob":"479H0KQ4LUUXIHL48WCVN0C9VD7HWSX0MG1UPYNXK6PI9HLGBU2U10K3HPFJSROFEG5VD749WDPHWSHXXO72BOSY2G8TWUDOJNLRTR9LTT8PSOB9NNZ485EY2RD9D80FLDFRBVMP1RKMELILD7I922D6TBCAZK30CSV6KDEDUMYABE0XB9EH8C4LE98LMU91I9ZV2APETJD4AYFEN0VNMIT1XQ122Y2OOXO45GJ737HHM5XX88RY7CXHVWJ5JJ7NYW6T1EEBW9UE0NLB2497YBP9V1XVAEK8JJYVRVW0L03ZDXFY8BBHP6UBU7ZNR0JU9GJQPNHG0DK86S4LLYDN0BTCF4KWV2J4DEB6DAX4BDLNPT87MM75G70DFE9W0R6HRNWCH0X075WHAXPSH7S3CSNXPPA6PDO6UA1RCCZOVZ99H7968Q37HACMD8EZ8SU81V4KNRXM46N520S4FVZNSJHA"
      },
      "status":"error"
   }
}
```

The following `rippled` log message also indicates that your server is amendment blocked:

```
2018-Feb-12 19:38:30 LedgerMaster:ERR One or more unsupported amendments activated: server blocked.
```

If you are on `rippled` version 0.80.0+, you can verify that your `rippled` server is amendment blocked using the [`server_info`](server_info.html) command. In the response, look for `result.info.amendment_blocked`. If `amendment_blocked` is set to `true`, your server is amendment blocked.

**Example JSON-RPC Response:**

```
{
    "result": {
        "info": {
            "amendment_blocked": true,
            "build_version": "0.80.1",
            "complete_ledgers": "6658438-6658596",
            "hostid": "ip-10-30-96-212.us-west-2.compute.internal",
            "io_latency_ms": 1,
            "last_close": {
                "converge_time_s": 2,
                "proposers": 10
            },
...
        },
        "status": "success"
    }
}
```

If your server is not amendment blocked, the `amendment_blocked` field is not returned in the response.

**Caution:** `rippled` versions older than 0.80.0 do not include the `amendment_blocked` field, even if your server is amendment blocked.


#### How to Unblock an Amendment-Blocked `rippled` Server

Upgrade to the `rippled` version that supports the amendments that are causing your server to be amendment blocked. Ripple recommends that you [upgrade to the newest `rippled` version](install-rippled.html) to unblock your server and enable it to sync with the network again.

Depending on the scenario, you may be able to (and want to) unblock your server by upgrading to a `rippled` version that is older than the newest version. This is possible if the older version supports the amendments that are blocking your `rippled` server.

**Warning:** If the newest `rippled` version provides security or other urgent fixes, you should upgrade to the newest version as soon as possible.

To determine if you can unblock your `rippled` server by upgrading to a version older than the newest version, find out which features are blocking your server and then look up the `rippled` version that supports the blocking features.

To find out which features are blocking your `rippled` server, use the [`feature`](feature.html) admin command. Look for features that have `"enabled" : true` and `"supported" : false`. These values for a feature mean that the amendment is currently enabled (required) in the latest ledger, but your server does not know how to support, or apply, the amendment.

**Example JSON-RPC Response:**

```json
{
    "result": {
        "features": {
            "07D43DCE529B15A10827E5E04943B496762F9A88E3268269D69C44BE49E21104": {
                "enabled": true,
                "name": "Escrow",
                "supported": true,
                "vetoed": false
            },
            "08DE7D96082187F6E6578530258C77FAABABE4C20474BDB82F04B021F1A68647": {
                "enabled": true,
                "name": "PayChan",
                "supported": true,
                "vetoed": false
            },
            "1562511F573A19AE9BD103B5D6B9E01B3B46805AEC5D3C4805C902B514399146": {
                "enabled": false,
                "name": "CryptoConditions",
                "supported": true,
                "vetoed": false
            },
            "157D2D480E006395B76F948E3E07A45A05FE10230D88A7993C71F97AE4B1F2D1": {
                "enabled": true,
                "supported": false,
                "vetoed": false
            },
...
            "67A34F2CF55BFC0F93AACD5B281413176FEE195269FA6D95219A2DF738671172": {
                "enabled": true,
                "supported": false,
                "vetoed": false
            },
...
            "F64E1EABBE79D55B3BB82020516CEC2C582A98A6BFE20FBE9BB6A0D233418064": {
                "enabled": true,
                "supported": false,
                "vetoed": false
            }
        },
        "status": "success"
    }
}
```

In this example, conflicts with the following features are causing your `rippled` server to be amendment blocked:

* `157D2D480E006395B76F948E3E07A45A05FE10230D88A7993C71F97AE4B1F2D1`

* `67A34F2CF55BFC0F93AACD5B281413176FEE195269FA6D95219A2DF738671172`

* `F64E1EABBE79D55B3BB82020516CEC2C582A98A6BFE20FBE9BB6A0D233418064`

To look up which `rippled` version supports these features, see [Known Amendments](known-amendments.html).


## Testing Amendments

If you want to see how `rippled` behaves with an amendment enabled, before that amendment gets enabled on the production network, you can run use `rippled`'s config file to forcibly enable a feature. This is intended for development purposes only.

Because other members of the consensus network probably do not have the feature enabled, you should not use this feature while connecting to the production network. While testing with features forcibly enabled, you should run `rippled` in [stand-alone mode](rippled-server-modes.html#reasons-to-run-a-rippled-server-in-stand-alone-mode).

To forcibly enable a feature, add a `[features]` stanza to your `rippled.cfg` file. In this stanza, add the short names of the features to enable, one per line. For example:

```
[features]
MultiSign
TrustSetAuth
```


## See Also

- **Concepts:**
    - [Known Amendments List](known-amendments.html)
    - [Introduction to Consensus](intro-to-consensus.html)
- **Tutorials:**
    - [Run rippled as a Validator](run-rippled-as-a-validator.html)
    - [Contribute Code to rippled](contribute-code-to-rippled.html)
- **References:**
    - [Amendments ledger object](amendments-object.html)
    - [EnableAmendment pseudo-transaction][]
    - [feature method][]
    - [server_info method][]

<!--{# common link defs #}-->
{% include '_snippets/rippled-api-links.md' %}			
{% include '_snippets/tx-type-links.md' %}			
{% include '_snippets/rippled_versions.md' %}
