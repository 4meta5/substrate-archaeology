# Origins

Adding an origin involves implementing [`EnsureOrigin`](https://crates.parity.io/sp_runtime/traits/trait.EnsureOrigin.html) for a blanket struct with the calling parameters. In [`frame_collective`](https://crates.parity.io/pallet_collective/enum.RawOrigin.html):

```rust
pub enum RawOrigin<AccountId, I> {
    Members(MemberCount, MemberCount),
    Member(AccountId),
    _Phantom(PhantomData<I>),
}
```

The docs https://crates.parity.io/pallet_collective/enum.RawOrigin.html tell you more about each variant, but the interesting part is the other scaffolding required to implement the logic behind the variants.

For the second variant of the enum (`RawOrigin::Member`), a struct `EnsureMember`](https://crates.parity.io/pallet_collective/struct.EnsureMember.html) is declared:

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

It still isn't completely obvious to me how this verifies membership. It is likely that the inovocation is limited to an existing member by runtime method checks.

## membership module

* https://crates.parity.io/frame_support/macro.impl_outer_origin.html is called in `construct_runtime` and I'd like to understand more about how it is called
