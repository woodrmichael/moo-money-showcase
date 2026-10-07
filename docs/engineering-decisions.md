# Selected engineering decisions

Five implementation case studies covering reconciliation algorithms, financial data modeling, authorization, synchronization, and event processing.

## 1. Replacing a bank connection without losing account history

**Problem.** Reconnecting a bank can import overlapping history with different transaction identifiers, descriptions, or posting dates. Simply appending the new history creates duplicates; deleting the old history loses categories, notes, tags, attachments, and splits. Repeated purchases with the same amount make matching especially ambiguous.

**Choice.** Account replacement is a preview-and-confirm workflow. Candidate transactions must agree on amount, direction, and currency, then receive scores based on date proximity, merchant, and description. The matcher produces deterministic, maximum-cardinality one-to-one pairings: each transaction can be used once, and an earlier pairing can be reassigned to avoid leaving another valid match stranded.

For example, one imported purchase may match either of two old transactions while another can match only one. A greedy first choice can leave the second purchase unmatched. Reassigning the flexible match preserves both pairings. Scores guide candidate order; the algorithm does not claim to find the globally highest-scoring assignment.

The preview separates clear matches from transactions needing review and compares user customizations, including split contents rather than just split counts. Confirmation rechecks a signed fingerprint of the preview, rejects changed data or an unfinished import, and validates the user's decisions against the proposed pairings. A scoped operation identifier lets a retry recover a completed result. Provider cleanup happens after the database operation, so a failed cleanup call does not report a committed replacement as failed.

**Tradeoff.** Similarity is evidence, not proof that two purchases are identical. A review step adds work for the user but keeps uncertain matches and customization conflicts visible. Separating database completion from external cleanup also requires persistent recovery state.

**Regression coverage.** Tests cover legitimate repeated purchases, reassignment of a tempting first match, different currencies and directions, three-decimal amounts, conflicting split contents, unfinished imports, operation-ID reuse, attachment preservation, and cleanup failure after a committed replacement.

## 2. Currency conversion with explicit precision and uncertainty

**Problem.** Accounts can use different currencies, and historical totals need rates appropriate to the requested date. Exchange-rate data can have gaps or become stale. Treating a missing rate as zero produces a misleading financial result, while assuming every currency uses two decimal places loses precision.

**Choice.** The currency layer preserves the original currency identity and distinguishes official currencies, provider-defined units, and missing currency data. Conversion uses decimal arithmetic and rounds to the target currency's minor-unit precision, including currencies with zero or three decimal places.

A rate resolution carries both the value and its context: requested date, actual rate date, source, and availability status. Recent rates can be carried forward within a bounded window, with freshness exposed in the result. Missing amounts or unusable rates remain unknown rather than becoming zero.

Financial reads use persisted rate data instead of calling the external rate provider on a page request. Background refresh fills the cache, and concurrent preparation work can share an in-flight task without mixing requests for different base currencies.

**Tradeoff.** Decoupling provider calls makes page reads less dependent on provider latency and outages, but some conversions may temporarily be unavailable. The API and UI have to represent incomplete results. Carrying recent rates improves availability while requiring an explicit freshness policy.

**Regression coverage.** Tests cover target-currency rounding, unknown amounts, carried and expired rates, missing-cache reads without provider calls, background refresh failures, concurrent work sharing, and isolation between base currencies.

## 3. Household membership is an authorization decision

**Problem.** People can share financial data while keeping separate logins. A session can remain valid after a member leaves a household, so a household reference in a token cannot be the only access decision.

**Choice.** The API verifies authenticated sessions and checks that the user still belongs to the referenced household before granting protected household access. Services and data contracts carry household scope through financial reads and mutations. Membership checks use a short-lived cache with explicit invalidation when membership changes.

Account ownership is a separate concept: an owner label supports filtering and attribution inside a shared household. It does not create private financial records between household members.

**Tradeoff.** Live membership checks add database work. A short-lived cache reduces repeated reads but introduces freshness concerns, so membership changes must invalidate cached decisions and tests must cover old sessions.

**Regression coverage.** Middleware tests include access with an old token after household membership is removed. Provider sync tests also check that background synchronization uses household-scoped clients.

## 4. Transaction synchronization must be safe to replay

**Problem.** Financial-provider updates arrive in pages and can include additions, modifications, and removals. A failure after writing a page but before saving progress can cause that page to be replayed. Provider data can also change during pagination, and a pending transaction can later receive a different posted identifier.

**Choice.** The synchronization service processes changes using stable transaction identities and reconciliation logic. It advances the persisted provider cursor only after all pages and the durable notification event have been stored. If provider data changes during pagination, the service restarts from the original cursor with bounded retries.

This treats the cursor as a checkpoint for completed work. It does not assume that every write in a synchronization run is one database transaction; replay safety matters precisely because intermediate work may already exist.

**Tradeoff.** Delaying the checkpoint means retries can repeat provider reads and reconciliation. Stable identities and preservation of user edits require more logic than blindly inserting provider rows, but that complexity protects transaction history and user organization.

**Regression coverage.** Sync tests cover failed inserts, updates, and deletions without cursor advancement; notification enqueue failure; pagination mutation recovery; retry-safe tag copying; and split reconciliation across retries.

## 5. Subscription webhooks need durable processing state

**Problem.** Billing events can arrive more than once, overlap in processing, or arrive out of order. An older subscription event should not overwrite a newer cancellation or access decision. A verified event also needs to belong to the correct household.

**Choice.** Stripe webhook signatures are verified before processing. A database-backed event ledger claims each event with a processing lease and records completion or failure. Household bindings are checked against subscription and checkout context. Ordered subscription updates reject stale state, and conflicting events with equal timestamps are reconciled against Stripe's current subscription state. Subscription changes invalidate cached access decisions.

The ledger provides durable deduplication and retry coordination. Side effects still need replay-safe behavior; an event ledger alone is not an exactly-once guarantee across external services.

**Tradeoff.** This adds persistent event state, lease handling, and reconciliation calls. Those mechanisms make failures more explicit and retriable, while requiring care around partial completion and concurrent updates.

**Regression coverage.** Billing tests cover duplicate invoice activity, retry behavior after partial failure, and stale subscription events that must not restore cancellation fields or undo cleanup decisions.
