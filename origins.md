# Origins (proofs of membership)

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

## tangent: membership checks

> I've been thinking about membership checks for a while because I've expected the introduction of group signatures, but there are other priorities at the moment (and it is probably possible but not implemented yet, maybe need to fork `application-crypto`)

In the past, it seemed like I would always write the membership check as an auxiliary function in the `impl<T: Trait> Module<T>` block like

```rust
pub fn is_member(who: &T::AccountId) -> bool {
    Self::members().contains(who)
}
```

This calls a vector in runtime storage and checks if it contains the passed in `AccountId`, returning the result. A common pattern is to call `is_member` in an `ensure!` block to verify permissions before an action is exercised by a member. As an example, only members can vote,

```rust
fn vote(origin, vote: bool) -> DispatchResult {
    let voter = ensure_signed(origin)?;
    ensure!(Self::is_member(&voter), Error::<T, I>::NotAMember);
    ..
}
```

If this runtime method needs the `members` vector again, then it will need to do *another* call to storage to get this vector. This is weak sauce! In the new `society` PR, `is_member` is changed to conserve the lookup:

```rust
pub fn is_member(members: &Vec<T::AccountId>, who: &T::AccountId) -> bool {
    members.binary_search(who).is_ok()
}
```

The new structure passes in a reference to the vector and the item to be searched. The method's body applies binary search on the vector and the outcome is returned.

It should be noted that this structure is beneficial when the members vector is used again in the runtime method. In this case, `society` does multiple membership checks in `vouch` for fine-grained permissions control and error management:

```rust
// Check user is not already a member.
let members = <Members<T, I>>::get();
ensure!(!Self::is_member(&members, &who), Error::<T, I>::AlreadyMember);
// Check sender can vouch.
ensure!(Self::is_member(&members, &voucher), Error::<T, I>::NotMember);
ensure!(!<Vouching<T, I>>::exists(&voucher), Error::<T, I>::AlreadyVouching);
```

I might be shooting myself in the foot because the `society` PR isn't merged yet, but I don't think that would compromise this clever way to save a lookup when requiring a vector two or more times in the same runtime method.

## back to origins

* my code isn't compiling :(