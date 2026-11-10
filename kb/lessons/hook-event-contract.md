# Hook Event Contract Lessons

## Accepted Names Need a Producer

A hook event can pass validation and never fire. Check the accepted event vocabulary against events emitted by the real producers; constructing an event directly for a dispatcher test cannot establish that a producer exists. See the [hook event decisions](../concepts/hooks-and-plugins.md).

## Repeated Fixes Mean Parallel Owners

When repeated fixes expose different versions of the same fact, inspect its owners and consolidate where possible. Then review the whole system: a projection joining facts from two stores can also create divergence even when each store has one owner. See [review discipline](verification-and-review-discipline.md#from-repeated-finding-to-redesign) and [state design](state-design-lessons.md#a-link-that-spans-two-stores).
