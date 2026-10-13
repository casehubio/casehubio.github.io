---
layout: post
title: "CommercePlatform — Shopping Carts and the Naming Problem"
date: 2026-10-04
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [commerce, spi, real-world-knowledge, platform-spi]
---

# CommercePlatform — Shopping Carts and the Naming Problem

The third connector SPI for the Real-World Knowledge Platform landed today. CommercePlatform follows LocationPlatform (places, geocoding, directions) and sits alongside it as part of the Phase 3 infrastructure — giving the knowledge pipeline structured access to product catalogues, shopping carts, checkout flows, and order tracking.

Five capability sub-interfaces: `ProductSearch`, `ProductDetails`, `Cart`, `Checkout`, `OrderTracking`. The first two are read-only — browse and inspect products without authentication. The last three are user-scoped: each call takes a `userId` and the reference implementation maintains isolated state per user. This mirrors how BankPlatform handles `accountInformation(userId)` and `paymentInitiation(userId)`, but the commerce domain adds a wrinkle that banking doesn't have.

## The naming collision

The capability interface for cart operations is naturally called `Cart`. The model record representing the cart's contents is also naturally called `Cart`. Java's type system handles this fine with qualified names, but reading `Cart` methods that return `Cart` objects creates genuine confusion about which `Cart` you're looking at — the capability or the data.

I considered three options. Qualifying every return type with `io.casehub.connectors.commerce.model.Cart` works but makes method signatures unreadable. Renaming the capability interface to `CartOperations` breaks the convention every other SPI follows — `PlaceSearch`, `Geocoding`, `AccountInformation` are all noun-phrase names, not verb-phrases. Renaming the model record to `ShoppingCart` preserves the convention on both sides: the capability stays `Cart`, the data type gets a more specific name that's arguably clearer anyway.

`ShoppingCart` it is. None of the other SPIs had this collision because their capability names don't overlap with their model types — `PlaceSearch` returns `Place`, `AccountInformation` returns `AccountInfo`, `Checkout` returns `CheckoutResult`. Commerce is the first SPI where the domain concept names both the operation and the object.

## The reference implementation

Eight pre-loaded UK products across four categories — electronics, books, home, clothing. Sony headphones, a Kindle, a Barbour jacket, a Le Creuset Dutch oven. The seed data follows the same London-centric theme as the location-ref module's places, keeping the test universe internally consistent.

The backend abstraction (`CommerceBackend` → `InMemoryCommerceBackend`) separates storage from SPI wiring, same pattern as location-ref. Cart state lives in a `ConcurrentHashMap` keyed by userId. The compound operations on it (read-modify-write in `addToCart`) aren't atomic, which would matter in production but doesn't in a reference implementation that exists for testing and simulation.

Monetary values use `BigDecimal` throughout — the `Money` record carries amount and currency. No floating-point anywhere in the financial path. This was a deliberate choice worth noting because the simplest implementation would use `double`, and that would be wrong for any downstream consumer doing price comparison or total calculation.

## What this opens up

With LocationPlatform and CommercePlatform both landed, the knowledge pipeline in neocortex can start wiring up real-world entity acquisition across two domains — places and products. TravelPlatform (the third SPI in the design spec) completes the set, but the pipeline can be prototyped against the two that exist now.

The more interesting question is how product entities interact with place entities in MindMap. A restaurant found via LocationPlatform might sell products found via CommercePlatform. The identity resolution layer in the knowledge pipeline will need to handle cross-domain entity relationships — but that's a neocortex problem, not a connectors one. The connectors boundary holds: pure fetch, no aggregation, no persistence.
