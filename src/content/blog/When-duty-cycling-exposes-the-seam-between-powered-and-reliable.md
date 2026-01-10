---
title: "When duty cycling exposes the seam between powered and reliable"
description: "Low power modes surface readiness assumptions that steady state systems quietly satisfy."
pubDate: 2026-01-XX
heroImage: "../../assets/modem-power-seam.png"
---

## When duty cycling exposes the seam between powered and reliable

In many systems, readiness is treated as a binary property. A component is powered, it responds to commands, and the system proceeds.

That assumption holds in steady state. It becomes fragile once power is no longer continuous.

Duty cycling does not simply reduce energy consumption. It removes the temporal slack that steady state systems rely on without ever modeling. When that slack disappears, the distinction between “powered” and “reliable” becomes operationally relevant.

## Steady state and the illusion of readiness

The device in question periodically publishes telemetry over a cellular connection. In its original configuration, the modem remained powered continuously. Between transmissions, the rest of the system slept.

In this mode, publishes were reliable. No special handling was required. Once the modem reported connectivity, the system assumed it was ready to use.

That assumption held because the modem spent most of its lifetime powered, idle, and physically stable.

## What changed under duty cycling

To reduce power consumption, the modem was power gated and enabled only when data needed to be transmitted.

Nothing else changed. UART configuration remained the same. The modem firmware was unchanged. The cloud side was untouched.

After power-up, the modem responded correctly to AT commands. Network registration completed successfully. The MQTT session was reported as connected.

At that point, the system attempted to publish data.

## Where the failure appeared

The failure occurred during the MQTT publish transition.

After issuing the publish command, the system waited for the modem’s prompt before sending the payload. Instead of the expected character, the modem returned corrupted data. Sometimes a different printable character. Sometimes non-ASCII noise.

Without a valid prompt, the payload was never sent and the publish operation stalled.

No other part of the interaction failed. Earlier commands succeeded. Connection state was reported as valid. Only the publish transition broke.

## Why it failed there and not earlier

The difference was not protocol state, but framing tolerance.

Most earlier interactions returned structured, multi-byte responses with redundancy and clear delimiters. Minor framing errors did not invalidate the exchange.

The publish transition depended on a single byte arriving correctly. There was no redundancy and no recovery path. Either the prompt arrived intact or the operation could not proceed.

Under steady state operation, this distinction was invisible. The modem had been powered long enough that physical conditions had stabilized before any interaction occurred.

Under duty cycling, the publish path was the first interaction that required precise framing shortly after power-up.

## What steady state had been doing implicitly

Continuous power had been providing a property the architecture never acknowledged.

Voltage rails had settled. Oscillators had locked. Internal clocks were stable. Thermal equilibrium had been reached. UART timing behaved predictably.

None of this was modeled explicitly. Readiness was inferred from responsiveness.

As long as the system remained in steady state, that inference held.

Duty cycling removed the padding that made it true.

## Why local fixes were insufficient

Once framed as a communication problem, several obvious responses presented themselves.

Retries reduced failure rates but did not eliminate them. Lenient parsing accepted corrupted input and made correctness conditional. Adding arbitrary delays improved behavior without guaranteeing it.

All of these approaches treated the symptom. They assumed the system could interact with hardware before it was reliable and attempted to compensate afterward.

The steady state configuration already demonstrated the correct behavior. Reliability emerged naturally once the modem had been powered long enough.

The missing element was not logic. It was time.

## Making the seam explicit

The correction was to treat readiness as something that emerges, not something implied.

Instead of powering the modem immediately before transmission, the system powered it earlier and allowed it to stabilize while continuing normal operation. Sensor sampling and other tasks proceeded as usual.

By the time transmission was required, the modem behaved identically to its steady state counterpart.

No protocol changes were required. No special cases were introduced. The system simply stopped relying on an implicit property of continuous power.

## The broader pattern

This failure is not specific to cellular modems, UARTs, or MQTT.

Low power modes force systems to confront assumptions that steady state hides. They introduce phases. They make time part of correctness. They turn previously invisible boundaries into explicit seams of responsibility.

When those seams are not modeled, systems compensate indirectly. Reliability depends on tolerance rather than structure.

When they are modeled, the fixes are often simple.

## Where this leaves system design

Power gating is not just an optimization. It changes the shape of the system.

Once a system leaves steady state, “on” and “ready” are no longer interchangeable. Treating them as such works only as long as the system never needs to observe the difference.

When you duty cycle a modem, the seam between powered and reliable becomes a design responsibility.

Steady state systems can afford to ignore that distinction.  
Low power systems cannot.
