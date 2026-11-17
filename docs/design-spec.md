# Design Specification

*(Draft in progress. Scope, roles and rules live in [`function-list.md`](function-list.md); this file covers only how things look. V1 tokens and components assume TailwindCSS.)*

## Imagery tokens (minimal; full component spec deferred)

* Shop logo 1:1 (rounded, placeholder initial) and banner 3:1 / 16:9 crop (EB-019). Product photos 4:3 / 16:9 crop (EB-010).
* Public Cloudflare R2 URLs with `onError` placeholder fallback and reserved aspect-ratio box (no layout shift).
* Missing/broken images never block checkout (EB-007), payment verification (EB-012) or hand-out (EB-016).
