# Origins

The [`collective`](https://crates.parity.io/pallet_collective/) module demonstrates how a native origin type. This can be used to authenticate different types of decisions with different signalling requirements. The module's `Trait`:

```rust
pub trait Trait<I=DefaultInstance>: frame_system::Trait {
	/// The outer origin type.
	type Origin: From<RawOrigin<Self::AccountId, I>>;

	/// The outer call dispatch type.
	type Proposal: Parameter + Dispatchable<Origin=<Self as Trait<I>>::Origin>;

	/// The outer event type.
	type Event: From<Event<Self, I>> + Into<<Self as frame_system::Trait>::Event>;
}
```

The [`frame_collective::RawOrigin`](https://crates.parity.io/pallet_collective/enum.RawOrigin.html) type:

```rust
pub enum RawOrigin<AccountId, I> {
    Members(MemberCount, MemberCount),
    Member(AccountId),
    _Phantom(PhantomData<I>),
}
```

We can use this structure to define the logic behind consensus thresholds.

## Consensus Thresholds

In order to implement consensus thresholds that correspond to the `RawOrigin` variants, it is necessary to instantiate a unit struct that implements [`EnsureOrigin`](https://crates.parity.io/sp_runtime/traits/trait.EnsureOrigin.html).

Corresponding the second variant of the enum (`RawOrigin::Member`), a unit struct [`EnsureMember`](https://crates.parity.io/pallet_collective/struct.EnsureMember.html) is declared in [`collective`](https://crates.parity.io/pallet_collective/):

```rust
pub struct EnsureMember<AccountId, I=DefaultInstance>(sp_std::marker::PhantomData<(AccountId, I)>);
```

and it implements the [`EnsureOrigin`](https://crates.parity.io/sp_runtime/traits/trait.EnsureOrigin.html) trait.

```rust
impl<
	O: Into<Result<RawOrigin<AccountId, I>, O>> + From<RawOrigin<AccountId, I>>,
	AccountId,
	I,
> EnsureOrigin<O> for EnsureMember<AccountId, I> {
	type Success = AccountId;
	fn try_origin(o: O) -> Result<Self::Success, O> {
		o.into().and_then(|o| match o {
			RawOrigin::Member(id) => Ok(id),
			r => Err(O::from(r)),
		})
	}
}
```

## open questions:
* how can `origin` be used to verify membership?
* it seems like vote counting is not usually done in these implementations, but rather these instances provide verification that there is already enough support for a given proposal
* how is this possible if it doesn't have context with respect to which proposal is being voted on `=>` how does it know how many members support the proposal for `Members(n, m)` unless these values are only passed in within the runtime method itself which is how it's done...
* in that case, what is the point of this? just an explicit check? It seems easier to encode the checks in the runtime methods but it's also less explicit than using this handler instead, but I think it would be easier to verify errors
* what errors are ever returned from these `EnsureOrigin` implementations
* it seems to be generally easier to not use these hooks and encode all permissions in local runtime methods like `collective`, but this doesn't really promote ease of reuse for these vote counting algorithms

**theory**
* the way to do it is to have the checks in the runtime but abstract vote counting algorithms out to methods that are associated with trait implementations `=>` this allows us to organize default implementations by shape 

### dynamic origin instantiation
> calling  https://crates.parity.io/frame_support/macro.impl_outer_origin.html within runtime method to instantiate a new `Origin` for recipients and propose the related runtime upgrade to update some module that tracks all the recipient `Origin`s