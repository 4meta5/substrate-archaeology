# Membership Proofs

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

## future membership proof primitives
* accumulators
* group signatures
* linkable ring signatures