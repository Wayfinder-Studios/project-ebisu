# Design Specification

*(Draft in progress — D-05 stack note: v1 tokens and components assume TailwindCSS.)*

## Imagery tokens (D-16, minimal — full component spec deferred)

* Logo 1:1 (rounded, placeholder initial), banner 3:1 / 16:9 crop, campaign cover 4:3 / 16:9 crop.
* Public R2 URLs with `onError` placeholder fallback and reserved aspect-ratio box (no layout shift).
* Missing/broken images never block checkout, verification, or dispatch flows.
