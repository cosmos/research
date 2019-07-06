# Staking Derivatives

At the inaugural Interchain Conversations, @zaki gave a talk explaining the important of staking derivatives, and why #StakingIsDeFi.  Also, as exchange validators like [Poloniex introduce trading on staked atoms](https://medium.com/circle-trader/cosmos-staking-is-live-78879f1523b4) it will be necessary to allow derivatives to be created for staked atoms on any validator, not only custodial validators like Poloniex and Coinbase.

## Assetizing Delegations

In some of the early designs for staking derivatives that @zaki and I were designing, we started with the premise of turning the `Delegation` struct already existing in the gaia codebase into an asset that could be transferred.  However, these delegation assets would have to be non-fungible assets (NFAs) because of [the way that the F1 fee-distribution works](https://cosmos.network/docs/spec).  Because F1 needs to keep track of the last time each individual delegator withdrew, these delegation assets are not fungible.

We can imagine the delegation asset as follows with attributes and capabilities:
```
Delegation non-fungible-asset {
    To_Validator

    Amount mut
    Last_Withdrawal_Time mut
    
    Unbond()
    Redelegate()
    Withdraw()
    Vote()
    GetSlashed()
}
```
where the holder of this asset is able to call any of the capabilities on that object (i.e. withdraw rewards, unbond, etc).

It is because of the `Last_Withdrawal_Time` attribute of a delegation object that they are not fungible as a Delegation asset with an earlier `Last_Withdrawal_Time` is entitled to more rewards from the F1 rewards pool than a delegation object with a later `Last_Withdrawal_Time`.

However, if staking derivative are to be useful in DeFi, it would be desirable to create one in which all derivatives are fungible.  While NFA derivatives could be traded OTC, it would be difficult to put them into a [Uniswap-style market](https://medium.com/scalar-capital/uniswap-a-unique-exchange-f4ef44f807bf) or even a traditional orderbook system.  It would furthermore heavily complicate using the derivatives as collateral for a [Kava CDP](https://medium.com/kava-labs/defi-coming-to-cosmos-808034b733be) for example, as the price oracles would have to report prices for each different NFA seperately.

Thus, the search for fungible staking derivatives was on.

## Delegation Vouchers

At the Cosmos HackAtom Berlin, the [Sikka](https://www.sikka.tech/) and [Chorus One](https://chorus.one/) joint team implemented a mechanism called "delegation vouchers" outlined in [this blog post](https://blog.chorus.one/delegation-vouchers/).  Essentially they successfully create fungible delegation vouchers by removing the F1 fee pool altogether and autobonding all rewards.  However, this is done by adding the constraint that all rewards can only be in the staking token of a chain.  This may be reasonable for some chains, but one of the design goals of the Cosmos Hub is that fees can be [paid in a variety of tokens](https://github.com/cosmos/cosmos/blob/master/Cosmos_Token_Model.pdf), not just Atoms.  This becomes even more of a concern when the Hub upgrades to do [interchain staking](https://github.com/cosmos/ics/issues/27) in which the validators could presumably be being rewards in the native token of another chain.

https://blog.chorus.one/delegation-vouchers/

## DelegationClaims

So, in order to make the vision of a fungible staking derivatives on the Cosmos Hub a reality, we have to make it work within the context of the F1 reward distribution system.

In the original thinking of framing the problem in terms of atributes and capabilities, I had ignored an interesting capability that is currently present in the delegation objects in the Gaia staking module, the ability to `Change_Withdrawal_Address()`.  I assumed it was unimportant because the transfer of the Delegation NFA which contains the `Withdraw()` capability acts as a "changing of a withdrawal address anyways.  (Also, amusingly, I can't quite remember why this feature exists in the first place. I think it may be a remnant of the "`Create Validator on Behalf Of`" system that was removed right before launch of the Hub?)

By incorporating this capability into the design of the staking derivative asset, I think we can begin to make them fungible.  To start, we'll start by designing a derivative asset that is still non-divisble, but is "equivalent" with respect to everything but the amount of bonded Atoms that the specific asset holds.  Later we will turn these into truly fungible tokens.  We'll also ignore peripheral capabilities like governance voting for simplicity.

In this system, we will continue to have a delegation struct in the staking module keeper, but issue an asset that represents some capabilities on the delegation struct.
```
Delegation struct {
    ID
    To_Validator

    Withdrawal_Address mut
    Amount mut
    Last_Withdrawal_Time mut

    Withdraw() - only Withdrawal_Address
}

DelegationClaim non-fungible-asset {
    DelegationID

    Unbond()
    Redelegate()
    Change_Withdrawal_Address()
}
```

In this model, when Alice bonds some Atoms to a validator, it generates her a new `Delegation` struct with her address as the `Withdrawal_Address` AND sends her account an `DelegationClaim` NFA.  Now at any time, as so long as the `Delegation.Withdrawal_Address` is pointed to her address, she can call the `Withdraw()` function and withdraw the rewards that have been acculated since the `Delegation.Last_Withdrawal_Time`.  Furthermore, as long as she is in possession of the associated `DelegationClaim` asset, she can also `Unbond()` and `Redelegate()`; essentially she is the owner of the underlying bonded Atoms.

Now if Bob wants to buy Alice's position, he could pay her and obtain the `DelegationClaim` asset that corresponds to Alice's `Delegation`.  Now this is where the interesting part comes, by obtaining the `DelegationClaim` asset, Bob has become the effective owner of the underlying bonded Atoms (due to his ability the unbond the bonded atoms).  However, he hasn't started receiving the rewards yet.  The `Delegation` struct in the Hub staking module still lists Alice as the `Withdrawal_Address`.  Thus by obtaining the `DelegationClaim` NFA, Bob has NOT become the recipient of rewards earned by *his* bonded atoms.  But his NFA does have the capability to `Change_Withdrawal_Address()`, which means as the possessor of the `DelegationClaim` he can send a transaction on the Hub to change the `Withdrawal_Address` of the associated `Delegation` to his own address.  When he sends the `Change_Withdrawal_Address()` the `Delegation` will `Withdraw()` the rewards to the previous owner, Alice, set `Withdrawal_Address` to Bob, and set `Last_Withdrawal_Time` to the current time.  Until Bob actually decides to send that transaction, Alice actually continues to earn the rewards.  Bob has to do action manually, because we can't trigger it at the time-of-transfer as the transfer could have occured on a different chain than the Hub.

```
(claim DelegationClaim) func Change_Withdrawal_Address(newAddress) {
    delegation := GetDelegation(claim.DelegationID)
    delegation.Withdraw()
    delegation.Withdrawal_Address = newAddress
    delegation.Last_Withdrawal_Time = time.Now
}
```

If given the option of buying two different `DelegationClaims` both to the same validator and for the same amount, Bob doesn't care who the current `WithdrawalAddress` is or what the `Last_Withdrawal_Time` is, because neither are relevant to his rewards.  All that matters to Bob's rewards is how quickly he actually goes to change the withdrawal address, but the result of that action is the same regardless of which `DelegationClaim` he bought.  This allows these assets to be pseudo-fungible because the *ability to switch the reward address* on any two `DelegationClaims` is the same.  US Dollar Bills have serial numbers on them, but for all practical intents and purposes, we treat them as fungible.  In the same way, equal-amount NFAs that correspond to different delegations with different owners, can be treated as fungible.

Now, if Bob is just trading on these derivatives, maybe it's not worth actually changing the `Withdrawal_Address` because he's planning on holding the asset for such a small period that it's not actually worth claiming the rewards on it, as the amount he'll earn is not worth the cost of making the tx (or the IBC packet if he's on another chain).  On the other hand, if Bob bought this asset to hold, he can change the withdrawal address and start earning off of the asset.  Interestingly, for the seller, Alice, it might be in her incentive to sell to a "lazy buyer" who will be slow to make the change, because until then, she can continue to earn the rewards off of the claims she already sold.

## Fungible DelegationTokens

Now, the one thing we haven't resolved yet is the divisibility of the claims.  If a `DelegationClaim` corresponds to a `Delegation` of 10 ATOMs, there's no way to sell only 5 of the ATOMs.  This is a necessary property to achieve usefulness in DeFi applications.  So, now we will take the strategy thus far and make it usable in the case of a fungible token.

When Alice creates a new delegation to a validator, instead of recieving a single NFA that represents her entire delegation, she receives a tokens corresponding to the delegation of the amount in the Delegation.  The delegation struct in the staking module, instead of being indexed by an ID, will be indexed by "owner" (along with the `To_Validator` of course, as a delegator can be delegated to multiple validators).

```
Delegation struct {
    Owner
    To_Validator

    Amount mut
    Last_Withdrawal_Time mut

    Withdraw() - only Withdrawal_Address
}

DelegationToken token {
    Delegation_Owner
    Delegation_To_Validator

    Unbond()
    Redelegate()
    Change_Owner()
}

func HandleNewDelegation(delegator, amount, validator) {
    CreateDelegationStruct(
        Delegation {
            Owner: delegator,
            Amount: amount,
            Validator: validator,
            Last_Withdraw_Time: time.Now,
        }
    )

    TokensDenom := CreateNewTokenDenom(
        DelegationToken {
            Delegation_Owner: delegator,
            Delegation_To_Validator: validator,
        }
    )

    Send(delegator, amount * VoucherDenom)
}
```

Now, these new `DelegationToken`s act very similarly to the `DelegationClaim`s introduced earlier, except they only affect the amount of tokens in a delegation 1:1 with the `DelegationTokens` owned.

So, for example, Alice could delegate 10ATOMs to the Sikka validator, and she will recieve 10 `Alice:Sikka:DelTokens`.  She can then sell 5 of these to Bob who will now be the owner of 5 `Alice:Sikka:DelTokens`.  He can then use these tokens to `ChangeOwner()` of up to 5 of `Amount` in the Alice's `DelegationToken`







<!-- In this model, every time a delegation object is updated, rewards must be withdrawn. This can be a transfer of vouchers into the delegation object (new delegation, receipt of vouchers from another user, undelegating, redelegating or a claim against your delegation object from someone you transfered vouchers to). -->



## Still Open Questions:

Can we auto-rebond staking token rewards?

Do peripheral capabilities like Vote() go in the 

Hook to change sending of thingie

Tax implications of forced withdraws

Dealing with accounting of slashes.  It's actually not