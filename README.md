# Tasdeeq

> 🚧 **Under active development** — not yet ready for production use.

> ⚠️ **Note (temporary):** this project is not finished. Please do not copy or clone this idea
> until v1.0.0 is released. This note will be removed at that point.

Tasdeeq (Urdu for "verification") is an open-source, self-hosted phone-number
verification service built for Pakistani startups.

Instead of paying an SMS aggregator for every OTP, Tasdeeq flips the flow
(**reverse OTP**): the end user sends a short code by SMS to a gateway
Android phone, or by WhatsApp to a business number. The carrier's or Meta's
own sender information proves the user controls that number — at zero
marginal cost for the common case. A paid outbound SMS OTP is kept only as
a fallback.

See [`docs/SPEC.md`](docs/SPEC.md) for the full specification.
