# hotbot's WebDriverAgent: a flick without the hold

Branch `hotbot/no-completion-wait`, forked from `appium/WebDriverAgent` at `3d6a4e0`. One change to the agent's behaviour, opt-in per request.

## What

`POST /session/:id/wda/pressAndDragWithVelocity` takes `waitForCompletion` (default `true`, the stock behaviour). With `false`:

- `FBXCTestDaemonsProxy synthesizeEventWithRecord:waitForCompletion:error:` hands the event record to XCTest's synthesizer and does not spin the run loop until XCTest's completion fires;
- the record is kept alive in a set until that completion (released at the handler's return, the gesture stopped after the first touch);
- the hand-over stays on the agent's main thread (dispatched to a background queue, XCTest drops the event);
- the run loop is turned for one second after the hand-over so XCTest streams the finger's part of the flick (a quarter second delivered the touch but not the lift);
- the stability wait after the drag is skipped;
- the response carries the handler's stage times: `activeApplicationS`, `coordinatesS`, `dragCallS`, `waitedForCompletion`.

## Why

XCTest waits for the target app to go idle after every synthesised gesture. On an app that never goes idle (a dating card with video and badges) that is its timeout of about fifteen seconds, while the flick itself moves the screen within 0.5 to 1.4 s.

## Measured (2026-10-26 01:05, iPad, iPadOS 26.5.2, Bumble, the same build, from the card's top)

| | reply | screen moved | distinct frames at 30 fps | first photo's y after |
|---|---|---|---|---|
| awaited (stock behaviour) | 13.99 s | 0.72 to 1.70 s | 15 | -2242 |
| unawaited | 1.33 s | 0.53 to 1.44 s | 17 | -2223 |

## Build and run

```
xcodebuild -project WebDriverAgent.xcodeproj -scheme WebDriverAgentRunner \
  -destination 'id=<udid>' DEVELOPMENT_TEAM=<team> CODE_SIGN_STYLE=Automatic \
  PRODUCT_BUNDLE_IDENTIFIER=<bundle id> build-for-testing
ios install --path <DerivedData>/Build/Products/Debug-iphoneos/WebDriverAgentRunner-Runner.app --udid <udid>
ios runwda --bundleid <bundle id>.xctrunner --testrunnerbundleid <bundle id>.xctrunner --xctestconfig WebDriverAgentRunner.xctest --udid <udid>
```

## Open

Whether the one-second run-loop turn can shrink toward 0.6 s and still deliver the lift. A test whose card sits at the bottom measures nothing: bring it to the top first (an awaited upward velocity flick does; plain drags do not move this list).
