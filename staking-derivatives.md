# Staking Derivatives

At the inaugural Interchain Conversations, @zaki gave [a talk](http://kalpatech.co/Cosmos_Part_2_1.ogg) explaining the important of staking derivatives, and why #StakingIsDeFi.  Also, as exchange validators like [Poloniex introduce trading on staked atoms](https://medium.com/circle-trader/cosmos-staking-is-live-78879f1523b4) it will be necessary to allow derivatives to be created for staked atoms on any validator, not only custodial validators like Poloniex and Coinbase.

http://kalpatech.co/Cosmos_Part_2_1.ogg

## Assetizing Delegations

In some of the early designs for staking derivatives that @zaki and I were designing, we started with the premise of turning the `Delegation` struct already existing in the gaia codebase into an asset that could be transferred.  However, these delegation assets would have to be non-fungible assets (NFAs) because of [the way that the F1 fee-distribution works](https://cosmos.network/docs/spec).  Because F1 needs to keep track of the last time each individual delegator withdrew, these delegation assets are not fungible.

We can imagine the delegation asset as follows with attributes and capabilities:
```
Delegation non-fungible-asset {
    To_Validator

    Shares mut
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

However, if staking derivative are to be useful in DeFi, it would be desirable to create one in which all derivatives from a single validator are fungible.  While NFA derivatives could be traded OTC, it would be difficult to put them into a [Uniswap-style market](https://medium.com/scalar-capital/uniswap-a-unique-exchange-f4ef44f807bf) or even a traditional orderbook system.  It would furthermore heavily complicate using the derivatives as collateral for a [Kava CDP](https://medium.com/kava-labs/defi-coming-to-cosmos-808034b733be) for example, as the price oracles would have to report prices for each different NFA seperately.

Thus, the search for fungible staking derivatives was on.

> Note that the Delegation struct currently tracks delegator shares, not number of Atoms.  This is so that if a validator gets slashed, the number of outstanding shares stays the same, but the exchange ratio between a validators' shares change.  So if there are currently 10 Atoms in a validator's bonded pool and 10 outstanding shares, the unbonding ratio from shares to Atoms is 1:1.  But if the validator gets slashed by 20%, there are now 8 Atoms in the validator's bonded pool and 10 outstanding shares, so the unbonding ratio is 1:0.8.  And any new atoms that get deposited are credited with the inverse number of shares.  So if someone deposits 4 Atoms, they will recieve 5 shares.

## Delegation Vouchers

At the Cosmos HackAtom Berlin, the [Sikka](https://www.sikka.tech/) and [Chorus One](https://chorus.one/) joint team implemented a mechanism called "delegation vouchers" outlined in [this blog post](https://blog.chorus.one/delegation-vouchers/).  Essentially they successfully create fungible delegation vouchers by removing the F1 fee pool altogether, turning the delegator shares into tokens directly, and autobonding all rewards (thus allowing the delegator shares to appreciate).  However, this is done by adding the constraint that all rewards can only be in the staking token of a chain.  This may be reasonable for some chains, but one of the design goals of the Cosmos Hub is that fees can be [paid in a variety of tokens](https://github.com/cosmos/cosmos/blob/master/Cosmos_Token_Model.pdf), not just Atoms.  This becomes even more of a concern when the Hub upgrades to do [interchain staking](https://github.com/cosmos/ics/issues/27) in which the validators could presumably be being rewards in the native token of another chain.

https://blog.chorus.one/delegation-vouchers/

## Delegation Claims

So, in order to make the vision of a fungible staking derivatives on the Cosmos Hub a reality, we have to make it work within the context of the F1 reward distribution system.

In the original thinking of framing the problem in terms of atributes and capabilities, I had ignored an interesting capability that is currently present in the delegation objects in the Gaia staking module, the ability to `Change_Withdrawal_Address()`.  I assumed it was unimportant because the transfer of the Delegation NFA which contains the `Withdraw()` capability acts as a "changing of a withdrawal address anyways.  (Also, amusingly, I can't quite remember why this feature exists in the first place. I think it may be a remnant of the "`Create Validator on Behalf Of`" system that was removed right before launch of the Hub?)

By incorporating this capability into the design of the staking derivative asset, I think we can begin to make them fungible.  To start, we'll start by designing a derivative asset that is still non-divisble, but is "equivalent" with respect to everything but the amount of shares that the specific asset holds.  Later we will turn these into truly fungible tokens.  We'll also ignore peripheral capabilities like governance voting for simplicity.

In this system, we will continue to have a delegation struct in the staking module keeper, but issue an asset that represents some capabilities on the delegation struct.
```
Delegation struct {
    ID
    To_Validator

    Withdrawal_Address mut
    Shares mut
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

If given the option of buying two different `DelegationClaims` both to the same validator and for the same number of shares, Bob doesn't care who the current `WithdrawalAddress` is or what the `Last_Withdrawal_Time` is, because neither are relevant to his rewards.  All that matters to Bob's rewards is how quickly he actually goes to change the withdrawal address, but the result of that action is the same regardless of which `DelegationClaim` he bought.  This allows these assets to be pseudo-fungible because the *ability to switch the reward address* on any two `DelegationClaims` is the same.  US Dollar Bills have serial numbers on them, but for all practical intents and purposes, we treat them as fungible.  In the same way, equal-amount NFAs that correspond to different delegations with different owners, can be treated as fungible.

Now, if Bob is just trading on these derivatives, maybe it's not worth actually changing the `Withdrawal_Address` because he's planning on holding the asset for such a small period that it's not actually worth claiming the rewards on it, as the amount he'll earn is not worth the cost of making the tx (or the IBC packet if he's on another chain).  On the other hand, if Bob bought this asset to hold, he can change the withdrawal address and start earning off of the asset.  Interestingly, for the seller, Alice, it might be in her incentive to sell to a "lazy buyer" who will be slow to make the change, because until then, she can continue to earn the rewards off of the claims she already sold.

## Fungible Delegation Shares

Now, the one thing we haven't resolved yet is the divisibility of the claims.  If a `DelegationClaim` corresponds to a `Delegation` of 10 shares, there's no way to sell only 5 of the shares.  This is a necessary property to achieve usefulness in DeFi applications.  So, now we will take the strategy thus far and make it usable in the case of a fungible token.

When Alice creates a new delegation to a validator, instead of recieving a single NFA that represents her entire delegation, she receives a tokens corresponding to the delegation of the amount in the Delegation.  The delegation struct in the staking module, instead of being indexed by an ID, will be indexed by "owner" (along with the `To_Validator` of course, as a delegator can be delegated to multiple validators).

```
Delegation struct {
    Owner
    To_Validator

    Shares mut
    Last_Withdrawal_Time mut

    Withdraw() - only Withdrawal_Address
}

DelegationShare token {
    Delegation_Owner
    Delegation_To_Validator

    Unbond()
    Redelegate()
    Change_Owner()
}

func HandleNewDelegation(delegator, amount, validator) {
    del := CreateDelegationStruct(
        Delegation {
            Owner: delegator,
            Shares: amount * conversion ratio,
            Validator: validator,
            Last_Withdraw_Time: time.Now,
        }
    )


    SharesDenom := sdk.CreateNewTokenDenom(
        DelegationShare {
            Delegation_Owner: delegator,
            Delegation_To_Validator: validator,
        }
    )

    bank.Mint(delegator, del.Shares * SharesDenom)
}
```

Now, these new `DelegationShare`s act very similarly to the `DelegationClaim`s introduced earlier, except they only affect the amount of shares in a delegation 1:1 with the `DelegationShares` owned.

Assuming the conversion ratio is currently 1:1, Alice could delegate 10ATOMs to the Sikka validator, and she will recieve 10 `Alice:Sikka:DelShares`.  She can then sell 5 of these to Bob who will now be the owner of 5 `Alice:Sikka:DelShares`.  He can then use these tokens to `ChangeOwner()` of up to 5 of `Shares` in the `Delegation` struct owned by Alice.  When he changes the owner of some of Alice's shares, the mechanism will automatically withdraw Alice's rewards for the amount of shares that Bob is trying to `Change_Owner()` with, credit her with the rewards, and then update her `Delegation.Shares` to remove the amount that Bob is trying to change ownership of.  It will then create Bob his own `Delegation` struct with `Shares` set to the amount of shares he took from Alice and the `Last_Withdrawal_Time` set to the current time.  It will then burn Bob's `Alice:Sikka:DelShares` and instead mint him some `Bob:Sikka:DelShares`.

But what if Bob already has a Delegation struct to the same validator?  In this case, it will withdraw all the rewards for the shares already in Bob's struct and then add his newly obtained shares to it and reset the `Last_Withdrawal_Time` to the current time.

```
func HandleChangeOwnership(validator, fromDelegator, toDelegator, numShares) {

    successful := bank.BurnCoins(toDelegator, numShares * fromDelSharesDenom)
    if not successful {
        throw
    }

    fromDel := GetDelegation(validator, fromDelegator)
    fromDel.Shares -= numShares
    fromDel.WithdrawRewards(numShares)

    toDel := GetDelegation(validator, toDelegator)

    if toDel not found {
        toDel = HandleNewDelegation(delegator, amount, validator)
    } else {
        toDel.WithdrawRewards(toDel.Shares)
        toDel.Shares += numShares
        bank.Mint(toDelegator, toDelSharesDenom)
    }
}
```

In this system, to a buyer Charlie, `Alice:Sikka:DelShares` should be equivalently valued to `Bob:Sikka:DelShares` because they are both 1:1 transformable into `Charlie:Sikka:DelShares`.

Now everything seems great...except two small problems.  

1.  For tokens that are currently in an active redelegation away from another validator, they are slashable for both the current validator's faults and the previous validator's fault, and thus have a higher risk profile and are not fungible with shares that are no longer in an active redelegation.
2.  For newly bonded tokens, they're not subject to slashes that happen before they started bonding.  For example, if a validator commits a fault at block 10, Alice delegates at block 15, and evidence is submitted at block 20, Alice's tokens should not be slashable.  (This is not implemented in the Cosmos Hub yet, but [it should be](https://github.com/cosmos/cosmos-sdk/issues/1440).)  Because these new delegations are slightly less at risk than tokens that have been bonded for longer than an unbonding period, they are not perfectly fungible.

Both of these edge cases involve some funkiness that happens for brand new delegations to a validator.  Thus, the simple solution for resolving both these problems is to not allow a new delegation to issue derivatives until after it has been bonded for at least one unbonding period.

With this design, DelShares from the same `Delegation` are truly fungible and `DelShare`s to the same validator from different `Delegations` are "pseudofungible".  And interesting next step challenge would be to design a system in the Cosmos SDK and other frameworks that allow pseudofungible assets to be treated the same (for example, we want to be able to put pseudo-fungible shares in the same Uniswap pool).  The design for such a system is out of scope for this spec, but some very preliminary discussion can be found [here](https://github.com/cosmos/cosmos-sdk/issues/1980).


## Still Open Questions:

**Can we auto-rebond staking token rewards?**

Currently, in the Cosmos Hub, all inflationary atoms are distributed as rewards.  However, there may be some desire to have inflationary atoms auto-bonded instead.  This would allow the rewards to a validator pool to be automatically added validator's bonded pool and the conversion ratio of shares to atoms be constantly increasing.  Some things around this still need to be thought through such as how it interacts with slashing and that it works in the context of this derivatives system.

**Do peripheral capabilities like `Vote()` go in the `Delegation` or the `DelegationShare`?**

In the spec, we mentioned that we will abstract away peripheral capabilities associated with staking such as the ability to vote in governance.  Where should these capabilites be located, in the `Delegation` or in the `DelegationShare`?

**Should transfers of `DelegationShare`s on the Cosmos Hub automatically trigger `Change_Owner()`?**

If a `DelegationShare` is transferred on another zone (where most DeFi applications will be located), the owner has to come back the Hub to manually execute the changeing of owner.  However, in the cases where a transfer is made on the hub itself, should we just make a hook that automatically changes ownership whenever a share is sent to a new address?

**Tax implications of Forced Withdraws**

Because when you merge `Delegation` structs, it causes a forced withdraw of rewards earned thus far so that it can reset the `Last_Withdrawal_Time`.  This may have tax implications that should be accounted for.  How can we design tags that make it as easy as possible for delegators to take these into account?

---

Special thanks to the [Chorus One](https://chorus.one/) team with whom many discussions took place in the process of designing this spec.