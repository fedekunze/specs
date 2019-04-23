# A Slashing Insurance Scheme for BPoS

**Author**: Federico Kunze - <federico@tendermint.com>

**Date**: April 22, 2019

## Abstract

The current proposal aims to create an insurance mechanism that refunds delegators in case one of her delegated validators is slashed. This model effectively provides incentive for both validators and delegators by providing them another source of revenue and solving risk averse holding strategies.

## Problem

### Delegator risk aversion due to slashing

Part of the security of the Cosmos Hub, involves that once a validator gets slashed due to an infraction (eg: double signing or downtime), the delegator bond also gets slashed.

This outcome can prevent risk averse users from delegating their staking tokens (`ATOM` in the Cosmos Hub), even when their tokens get diluted due to inflation.

 Even though validators or wallets could already implement insurance for their users using an off-chain implementation (either using cryptocurrencies or fiat), this is not as desirable as having it enforced in-protocol. This is because the insurance terms might not be publicly available, differences in regulatory legislations can have different outcomes and that there's no transparent way to check if the issuer had enough funds to pay for the insurance at a given date.

### Network Security

This holding strategy also has negative effects on the total security of the network as less coins are staked. This will become particularly relevant after `ATOM` transfers are enabled on the Hub.

### Centralization and commmoditization of validators

As today, we’ve seen a increasing amount of [centralization](<https://cosmos.bigdipper.live/voting-power-distribution>) in the Cosmos Hub into a few number of validators that control most of the power in the network. The existing centralization has caused a huge competition between validators to get more delegations, pushing them to reduce their commission earned from rewards. This affects primarily small validators who might not be able to sustain their business at such low rates.

 This model provides another differentiator for validators since day one, by allowing them to create different combination of insurance terms, thus reducing the current commoditization of validator services we’ve seen today.

### Incentives to highly secure validators

Because of the low existing mandatory percentage on signed blocks per window (currently 5% per 10000 blocks),  **delegators don't have an incentive to delegate on highly secure and available validators**. With the proposed insurance mechanism these validators can be create more expensive insurances with higher coverage, allowing them get more earnings for their secure architectures.

This model could also help raise the mentioned percentage as until now there was no incentive on increasing their own penalties. It would be then expected that the amount of insurances bought to be inversely proportional to the signed block percentage requirement.

## Proposal: Slashing Insurance

The proposed solution for this problem to create an in-protocol insurance mechanism that refunds delegators in the event of slashing.

- Validators issue new insurance terms, each of which contain a `Price` (which could be defined in `ATOM` or another coin denomination), `Duration` for which the insurance is active, `SlashingInfractions` for which infractions does the insurance terms apply and the `Coverage` percentage if a slashing occurs.

- Delegators buy these insurances from validators **only if they have delegation to the insurer**. This insurance is due once the duration period ends.

- If the insurance term hasn’t change, delegators can extend the duration of their insurance for another period.

- In the event of slashing, refunds are payed to each delegator,  from oldest to newest issuance date of the insurance (i.e by using a FIFO queue).

- Each delegator gets payed  `refund = SlashingInfraction * Validator.Power * Coverage​$ (1)`, which is deducted from the validator operator's balance, where `SlashingCondition` is a reference to the value of the slashing infraction parameter.

## Specification

Here's  an early specification of the insurance types and messages based on the descriptions stated in this document. This is not final and should be only considered as a guide to understand how the insurance mechanism works.

### Types

A validator can create a set of insurance tiers, each one composed by one `InsuranceTerm`. An  `InsuranceTerm` is composed by a set of rulesets that define an insurance contract with a validator.

```go
type InsuranceTerm struct {
  ID                  uint64          // ID of the insurance term
  Coverage           sdk.Dec         // total percent of the amount slashed refunded in case of a slashing event
  Price               sdk.Coins       // price of the insurance tier
  Duration            time.Duration   // duration of the insurance coverage
  SlashingInfractions []Infraction    // for which slashing condition (param) this term applies
}
```

Once a delegator buys (therefore accepts) the terms, it creates a "contract" between her and the validator, that refunds `Coverage` % of the amount slashed while the insurance is active. This `Insurance` contract is due on `EndDate`, were `EndDate = StartDate + Term.Duration`.

```go
type Insurance struct {
  Delegator   sdk.AccAddress // address of the buyer
  Validator   sdk.ValAddress // address of the insurer validator
  StartDate   time.Time      // start date of the insurance
  Term        InsuranceTerm  // terms of the insurance
}
```

### Messages

The messages for this module are validator and delegator specific that handle the CRUD actions for the `InsuranceTerms` and `Insurances` respectively.

#### Validator Specific Msgs

- `MsgCreateInsuranceTerm`: create a new insurance tier

- `MsgEditInsuranceTerm`: update the `Price`, `Coverage` or `Duration` of an `InsuranceTerm`. This also automatically replaces the insurance term `ID` to a new one.

- `MsgDeleteInsuranceTerm`: Delete an existing `InsuranceTerm`from the validator.

> **Note**: `MsgEditInsuranceTerm` and  `MsgDeleteInsuranceTerm`changes won't affect delegators who already purchased or subscribed to the insurance original terms.

#### Delegator Specific Msgs

- `MsgBuyInsurance` : buy a `Insurance` with a defined `Term` and  `Price` from a validator.

- `MsgExtendInsurance`: extend the `EndDate` of the insurance by `Term.Duration` , paying `Term.Price` to the validator. If the original insurance term is deleted or modified, this will return an error.

## Future Improvements

Below are further ideas for improvements to this model that could be interesting to research and implement while the network evolves:

1. **Subscription Service**: This model could eventually be extended in the future through on-chain subscription services. The delegator could pay a periodical subscription fee (that may or may not be equal to the `Term.Price`) This would imply creating new handlers and message types for subscribing and unsubscribing the insurance.
2. **Update Insurance**: In case a user wants to upgrade or downgrade his current insurance tier to comply with other terms.
3. **Slashing Curves**: the idea of slashing curves is that validators get slashed with an amount that is a function of their power weight. The current specification of insurance should be compatible with a slashing cuve implementation.
4. **Acummulative Refunds**: if the validator is slashed multiple times, one could add a weight to the refund amount.

## FAQ

### What if an insured delegator unbonds or redelegates while having an active insurance with a validator ?

The insurance will remain active until either the unbonding period or the duration of the insurance ends, after which is deleted from the state. If the unbonding period ends before the insurance duration ends, the delegator is then refunded `refund = Insurance.Term.Price/(EndDate - (UnbondTime + UnbondingPeriod))` for what was left of the active insurance.

If an infraction evidence is submitted during the unbonding period, the insurer validator can still get slashed and thus, they will need to refund the amount committed on the insurance.

### What if a validator is slashed multiple times during the lenght of the insurance ?

Every insurance refund needs to be payed every time a slashing is performed on the validator. If the insurer validator is slashed multiple times (before getting tombstoned) they need to pay for each refund.

If the validator commits multiple infractions on the same slashing period and before the evidence is discovered, he gets slashed only once, for the one of the highest amount. In consequence, the delegator gets refunded once.

Alternatively, if every infraction is discovered before the next infraction is commited, the validator will have to refund for each of them until they run out of funds. However, this is still under discussion if whether an insurance should be valid once or multiple times.

See the [slashing specification](<https://cosmos.network/docs/spec/slashing/>) and [timelines](<https://cosmos.network/docs/spec/slashing/01_concepts.html#ascii-timelines>) for more details.

### What happens if the validator doesn't have enough balance to refund all the insurances ?

We can somewhat mitigate this case by setting a cap on the validator's total number of insurance contracts signed with delegators based on the validator's self-bond balance.  

Given a validator liability for the total refund amount  `Total Refund  = ∑ refundᵢ`, delegators can buy new insurances while `Validator.SelfBond ≥ TotalRefund​` holds.

There are two cases that can modify this cap, when `1.` the validator self bond stays the same or increases, and `2.` when her self bond decreases due to self unbond or redelegations.

1. **Insurer self-bond stays the same or increases**:

   The cap described above will effectively prevent the insurer from running out of funds in the event of slashing.

2. **Insurer self-bond decreases**:

   If the validator self unbonds or redelegates and the condition above is not met, they won't be able to pay all her obligations to insured delegators. They still won't be able to accept new insurance contracts until the the condition holds. To prevent this case we can prevent the validator from unbonding or redelegating shares worth more than `MaxUnbond = Validator.SelfBond - TotalRefund` if they have active insurances. This will keep the condition satisfied and thus the validator will have enough funds to refund his insured delegators.

### What if a validator with active insurances gets tombstoned ?

After a validator is [tombstombed](<https://cosmos.network/docs/spec/slashing/07_tombstone.html>) due to multiple infractions (eg. double signing), all the insurance contracts to the validator affected are paid and then become finalized. This is because as the validator is kicked from the validator set for an unlimited period of time, delegators need to unbond or redelegate.

### What if a validator with active insurances gets jailed ? 

Jailing (*i.e* getting kicked off the validator set for a certain period of time) can occur due to several conditions:

- due to an infraction (double sign or downtime)
- if the validator self unbonds and his tokens from shares are now lower than his minimum self-bond: `Validator.Tokens < Validator.MinSelfDelegation​`

If the jailing was due to the first, the validator needs to refund the amount defined on each of the insurance terms.

### Can a delegator that operates a validator "self-buy" an insurance for himself ?

We can't really prevent a validator from self-buying an insurance for himself either by directly buying it or by transfering funds to another account and then buying an insurance.

The worst case scenario is desincentiviced for two reasons:

1. **It causes an economic cycle**: "self-buying" an insurance would mean that the validator also "self-pays" for his refund by transfering the coins to himself, which has no economic benefit for them.

2. **It caps new insurances**: as described above, buying an insurance decreases the total amount that the validator is able to refund in case of a slashing event. Thus reducing the cap of the total amount of insurances that are other delegators are able to buy from them, preventing them from getting more income.

### Could this model increase the griefing attack vectors on validators ?

One could buy an insurance from a validator and then have an incentive to attack them in order to claim the refund if the insurer is slashed. However, this is not economically profitable if the slashing amount coverage defined on the insturance terms is lower or equal than 100% because only bonded delegators can buy an insurance from a insurer validator.

It is then recomended that only validators that are highly confident of the sucurity and availability of their setups, and that have succesfully tested them on adversarial testnets such as Game of Stakes, to issue terms with coverage higher than 100%, as they could be subject to these attack vectors.