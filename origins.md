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

It still isn't completely obvious to me how this verifies membership. It is likely that the invocation is limited to an existing member by checks inside of the runtime methods.

### dynamic origin instantiation (idea)
> calling  https://crates.parity.io/frame_support/macro.impl_outer_origin.html within runtime method to instantiate a new `Origin` for recipients and propose the related runtime upgrade to update some module that tracks all the recipient `Origin`s