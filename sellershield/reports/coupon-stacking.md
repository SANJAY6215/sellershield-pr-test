# DISC-001: Coupon stacking bypasses single-coupon policy

Severity: **high** · Fix: `coupon-stacking` · Approved by: Demo Analyst on 2026-11-10T05:35:46.631Z

## Description
The coupon engine applies every submitted coupon in sequence, although policy allows only one coupon per order.

## Attack reproduction (simulated NovaCart shop)
1. Create an order of 4x NC-USB
2. Submit SAVE20, VIP15 and WELCOME10 together

Evidence from the original run:
- step 1: Create order 4x NC-USB ($200 subtotal) with coupons SAVE20+VIP15+WELCOME10 -> expected "Only SAVE20 (best single discount, -$40) applies; total $160", actual "Applied SAVE20, VIP15, WELCOME10; total $126"

## Root cause
Coupon application loops over all submitted coupons and never enforces maxCouponsPerOrder = 1.

## What the fix does
The fix changes the workflow so that only the first coupon a customer submits is evaluated and applied, while any additional coupons are simply ignored. By removing the loop that previously processed every coupon, the system now enforces the business rule of “one coupon per order” instead of allowing unlimited discounts. This directly eliminates the loophole that let shoppers stack multiple coupons and drive the order total down artificially. As a result, the original cause of the vulnerability—failing to check the max‑coupon limit—is no longer present.

## Before / after (SellerShield simulator - not a real-world attack)
- [passed] Original attack DISC-001 is blocked: before: exploited=true (leak $34); after: exploited=false
- [passed] Single valid coupon still applies: 4x NC-USB with SAVE20 -> $160 (expected $160)
- [passed] Order without coupon is priced at list: 1x NC-CHG -> $30 (expected $30)
- [passed] No new exploit in DISC-002: exploited before=false, after=false
- [passed] No new exploit in DISC-003: exploited before=false, after=false
- [passed] No new exploit in BIZ-001: exploited before=false, after=false
- [passed] No new exploit in REFUND-001: exploited before=true, after=true
- [passed] No new exploit in REFUND-002: exploited before=true, after=true
- [passed] No new exploit in REFUND-003: exploited before=false, after=false
- [passed] No new exploit in PAY-001: exploited before=true, after=true
- [passed] No new exploit in PAY-002: exploited before=false, after=false
- [passed] No new exploit in PAY-003: exploited before=false, after=false
- [passed] No new exploit in INV-001: exploited before=true, after=true
- [passed] No new exploit in INV-002: exploited before=false, after=false
- [passed] No new exploit in ACL-001: exploited before=true, after=true
- [passed] No new exploit in ACL-002: exploited before=false, after=false
- [passed] Chain PAY-001>ACL-001: still exploitable via another unfixed link
- [passed] Chain DISC-001>REFUND-001>REFUND-002: still exploitable via another unfixed link

## Simulated exposure
$34 excess per simulated attack; annual projection $986 / $4896 / $14688 (low / expected / high). **Simulated projection, not an actual loss.**

## Security impact
Removes the demonstrated loss path for this weakness in the simulated shop. Severity: high. Figures are simulated projections, not actual losses.

## Remaining limitations
- Verified only against the SellerShield simulator, not against a production system.
- This PR adds the patch, a regression test and evidence; it does not itself change application code.
- Other weaknesses and attack chains are unaffected unless they share this root cause.

## Trade-offs
- Marketing can no longer run intentionally stackable campaigns without an explicit allow-list rule.
- Side effect: Customers who previously stacked codes will see only one applied; storefront copy should explain this.
