---
description: "Cap's instrumentation challenges run server-generated JS to check browser behavior alongside proof-of-work in the self-hosted, open-source CAPTCHA."
---

# Instrumentation challenges

Instrumentation challenges are Cap's second layer of verification, running silently alongside the core proof-of-work system and present on Cap Standalone.

They generate a unique JavaScript program on every request that is executed inside the visitor's browser. The output is checked server-side, allowing Cap to verify that the script ran against a browser-shaped environment before accepting a token. Note that this is a best-effort signal, not proof of a genuine browser: the computation is deterministic given the script, so a sufficiently complete fake DOM can in principle reproduce it outside a browser.

## How they work

When a challenge is issued, the server generates a self-contained JavaScript bundle that runs a few browser API probes and evaluates a main computation chain, where multiple integer variables are initialised with random seed values and then mutated through randomised operations, including bitwise AND/OR/XOR/NAND, prototype-chain tricks, and DOM-based arithmetic that appends a tree of elements to the page, walks back up it accumulating values, and then removes them.

The server tracks the expected result of every operation in parallel, so it knows what the final four values must be.

All of these checks run inside an iframe, which `postMessage`s the answers back to the parent.

## Why DOM operations

Pure arithmetic can be replicated in a non-browser environment by simply running the JavaScript. Simple `innerText` round-trips and element-tree walks can also be stubbed correctly with a small fake DOM. Layout-dependent reads via `offsetWidth`, `offsetHeight`, and `getBoundingClientRect()` reject shims that do not implement these APIs consistently, but fixed geometry checks can be reproduced without a layout engine. These probes are compatibility checks against expected browser behavior, not proof that a real browser is present or a guarantee of substantial replay cost.

The layout probe uses border-box dimensions and a small tolerance for subpixel rounding at different zoom levels. It briefly retries zero-sized measurements while the iframe becomes rendered. An iframe that remains `display: none` cannot provide layout measurements and will fail the probe. Integrations should keep the instrumentation iframe rendered; the built-in widget positions it offscreen with zero opacity.

Instrumentation challenges often also mix these with a preset list of checks.

## Automated browser detection

Instrumentation challenges can also optionally attempt to block automated webdrivers. While we do a very large amount of checks for these, they are not foolproof. Even commercial, closed-source CAPTCHAs, like Turnstile, can be bypassed by attackers by using patched stealth browsers.

## Relationship to proof-of-work

Proof-of-work requires the client to perform computation to find a valid solution. Instrumentation adds a best-effort check of browser behavior, which a sufficiently complete simulation can reproduce. Instrumentation should always be paired with proof-of-work; it does not replace that computational cost or guarantee that the client used a browser.

Instrumentation is not foolproof. While challenges like these are deployed at massive scale by platforms such as [YouTube](https://www.reddit.com/r/youtubedl/comments/1mkzmp3/what_is_a_po_token/) and [Twitter](https://x.com/i/js_inst), I do not recommend using them as a replacement for proof-of-work. Without PoW and with real browsers, attackers can cheaply mine these challenges.
