---
layout: post
title: "Where's My Bag? Device Location and Asset Tracking on LocationPlatform"
date: 2026-11-20
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [location, spi, device-tracking, airtag, design]
---

# Where's My Bag? Device Location and Asset Tracking on LocationPlatform

LocationPlatform already knows about places — search for restaurants, geocode an address, get driving directions. What it doesn't know is where *you* are, or where your things are. "Find restaurants near me" shouldn't require hardcoding coordinates.

I added two new capability sub-interfaces to LocationPlatform: `DevicePosition` for host device GPS and `AssetTracking` for Bluetooth trackers like AirTags, Tiles, and SmartTags. The design question was whether these belong together or apart.

## Two Capabilities, Not One

The issue originally proposed a single `DeviceLocation` interface combining `currentPosition()`, `trackedDevices()`, and `devicePosition(deviceId)`. I split it into two independent capabilities because the provider landscape is fundamentally different for each.

GPS providers know where the device is. Tracker providers know where your stuff is. A browser geolocation bridge can do the first but not the second. A FindMy integration can do the second but not the first. Forcing them into a single interface means every provider stubs out half its methods — the kind of design that looks clean on a whiteboard and falls apart the moment a second implementation arrives.

The split keeps `supports()` honest. A provider either does device position or asset tracking or both, and the gating is explicit:

```java
interface DevicePosition {
    Position currentPosition();
}

interface AssetTracking {
    List<TrackedDevice> trackedDevices();
    Position devicePosition(String deviceId);
}
```

Both follow the established LocationPlatform pattern — inner interfaces, user-scoped accessors, NoOp singletons that throw `UnsupportedCapabilityException`.

## The Battery Problem

The interesting design constraint came from the `TrackedDevice` model. Every tracker API reports battery information, but the granularity varies wildly. Apple FindMy gives you a percentage and a charging status. Tile gives you a coarse three-level indicator: `LEVEL1`, `LEVEL2`, or `NONE`. SmartTag broadcasts a byte in its BLE advertisement frame.

We researched the actual APIs — FindMy.py (reverse-engineered), pytile (unofficial Tile API), SmartTag BLE advertisements (UUID 0xFD5A), Google Geolocation API, and the W3C Geolocation spec. None of these have official public APIs, which tells you something about the state of asset tracking as a platform.

The model we landed on:

```java
record BatteryInfo(BatteryStatus status, Integer levelPercent) {}
enum BatteryStatus { FULL, GOOD, LOW, CRITICAL, UNKNOWN }
```

`status` is always populated — a normalised coarse level that every provider can map to. `levelPercent` is nullable — populated by providers that have the data (FindMy), null for coarse-only providers (Tile). No false precision, no empty fields that every implementation ignores.

## Position vs Coordinates

The codebase already has `Coordinates(lat, lng)` as a simple value record. For device location I added `Position(Coordinates location, double accuracyMeters, Instant timestamp)`. The research showed that accuracy and timestamp are universal across every real location source — W3C Geolocation, Google's server-side API, every tracker protocol. Altitude, speed, and heading are browser-only fields that would be null for every tracker provider, so I left them out.

The ref implementation returns a fixed position at Balham station with three seed tracked devices scattered around South London — an AirTag on keys nearby, a Tile in a backpack in central London, a SmartTag on a dog collar at Wandsworth Common. Enough to make "find restaurants near me" work without real GPS, and enough variety for meaningful test coverage.

## What This Unlocks

The ref provider is the only implementation for now. Device location is inherently client-side — there's no server-side Google API that returns "where is this phone right now?" Real providers will need a bridge: a browser extension pushing geolocation data, a native app relaying CoreLocation, or someone building an unofficial FindMy integration brave enough to depend on Apple's undocumented protocols.

The SPI is the foundation. When those bridges arrive, the capability checks, the MCP wiring, and the error handling are already in place. The LLM-facing tools — `device_position`, `tracked_devices` — just work, because the dispatch pattern is the same one PlaceSearch and Directions already use.
