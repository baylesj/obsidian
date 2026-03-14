---
{"dg-publish":true,"permalink":"/garden/2026-03-14-open-screen-claude-generated-report/"}
---

# Open Screen Repository — Code Review Findings
*Generated: 2026-03-14. Focus: cast/, util/, platform/ directories.*

---

## Table of Contents

1. [Summary](#summary)
2. [Priority 1 — Correctness / Data Loss Risk](#priority-1--correctness--data-loss-risk)
3. [Priority 2 — Broken / Unimplemented Public API](#priority-2--broken--unimplemented-public-api)
4. [Priority 3 — Design Debt / FIXMEs](#priority-3--design-debt--fixmes)
5. [Priority 4 — Test Coverage Gaps](#priority-4--test-coverage-gaps)
6. [Priority 5 — Code Duplication and Minor Cleanup](#priority-5--code-duplication-and-minor-cleanup)
7. [Cross-Cutting: Exit-Time Destructors](#cross-cutting-exit-time-destructors)
8. [Appendix: What Is Actually Good](#appendix-what-is-actually-good)

---

## Summary

The repository is architecturally sound with clean layering, good use of `ErrorOr<T>`, `WeakPtr`, and the `Alarm`/`TaskRunner` patterns, and meaningful fuzzing coverage over network-facing parsers. However, there is a meaningful backlog of correctness gaps, unimplemented production code, and accumulated technical debt, concentrated in the TLS platform layer, the Cast sender API, and the statistics/streaming subsystems. The issues below are organized by severity.

---

## Priority 1 — Correctness / Data Loss Risk

### 1.1 TLS write flow control is unimplemented

**File:** `platform/impl/tls_connection_posix.cc`, line 32

```cpp
// TODO(jophba): implement write blocking/unblocking
TlsConnectionPosix::TlsConnectionPosix(IPEndpoint local_address,
                                       TaskRunner& task_runner)
    : task_runner_(task_runner),
      socket_(std::make_unique<StreamSocketPosix>(local_address)) {}
```

**Problem:** When the kernel write buffer is full, no back-pressure is applied. Writes can be silently dropped under sustained load. The entire write-blocking/unblocking path is absent from the class.

**Recommended fix direction:** Implement `OnWriteable()`/`OnUnwriteable()` state in `TlsConnectionPosix`. When a write returns `EAGAIN`/`EWOULDBLOCK`, buffer the pending data and register a write-ready callback on the socket. Drain the buffer when the socket becomes writable again, following the pattern used in `UdpSocketPosix`.

---

### 1.2 Android RTP payload hack is `true` by default

**File:** `cast/streaming/public/sender_session.h`, lines 123–125

```cpp
// Whether or not the android RTP value hack should be used (for legacy
// android devices). For more information, see https://crbug.com/631828.
bool use_android_rtp_hack = true;
```

**File:** `cast/streaming/impl/rtp_defines.h`, lines 110–116

```cpp
// Setting `use_android_rtp_hack` to true means that we match the legacy Chrome
// sender's behavior of always sending the audio and video hacks for AndroidTV,
// as some legacy android receivers require these.
// TODO(issuetracker.google.com/184438154): we need to figure out what receivers
// need this still, if any. The hack should be removed when possible.
RtpPayloadType GetPayloadType(AudioCodec codec, bool use_android_rtp_hack);
RtpPayloadType GetPayloadType(VideoCodec codec, bool use_android_rtp_hack);
```

**Problem:** The default `true` means all new senders use out-of-spec RTP payload types. The set of receivers that actually require this is unknown (per the TODO). This silently degrades protocol conformance for all senders that don't explicitly opt out.

**Recommended fix direction:**
1. Audit which receiver firmware versions require the hack (issue 184438154).
2. Switch the default to `false` if the affected population is small or known.
3. If the hack must remain, document which receiver versions require it in `rtp_defines.h` so future maintainers can make an informed removal decision.

---

### 1.3 `FlatMap::erase_key()` crashes if key is absent

**File:** `util/flat_map.h`, lines 51–60

```cpp
decltype(auto) erase_key(const Key& key) {
  auto it = find(key);
  // This will otherwise segfault on platforms, as it is undefined behavior.
  OSP_CHECK(it != this->end()) << "failed to erase: element not found";
  return static_cast<std::vector<std::pair<Key, Value>>*>(this)->erase(it);
}
```

**Problem:** Unlike `std::map::erase()` (which is a silent no-op for missing keys), `erase_key()` fatally crashes. The comment acknowledges the segfault risk. Callers that do a conditional erase must guard with a `find()` first, creating a redundant two-pass lookup.

**Recommended fix direction:** Add a separate `try_erase_key(const Key&)` that returns `bool` or an iterator and does not crash. Audit all callers of `erase_key()` to determine which require the crash (i.e., truly expect the key to exist) and which should use the safe variant.

---

## Priority 2 — Broken / Unimplemented Public API

### 2.1 `CastMediaSource::From()` is `OSP_UNIMPLEMENTED()` in production

**File:** `cast/sender/public/cast_media_source.cc`, lines 15–20

```cpp
// static
ErrorOr<CastMediaSource> CastMediaSource::From(const std::string& source) {
  // TODO(btolsch): Implement when we have URL parsing.
  OSP_UNIMPLEMENTED();
  return Error::Code::kUnknownError;
}
```

**Problem:** This is a **public factory method** on a public type. Any embedder calling `CastMediaSource::From(url)` receives a warning log and `Error::Code::kUnknownError`. There is no URL parsing in the repo to unblock this — it is simply not implemented.

**Recommended fix direction:**
- If `googleurl` (already in `third_party/`) provides adequate URL parsing, implement `From()` using it.
- At minimum, add a comment to the public header (`cast/sender/public/cast_media_source.h`) that this method is not yet implemented, and consider `= delete`-ing it or returning a more descriptive error code (`kNotImplemented` or similar) so callers can distinguish "bad input" from "not implemented".

---

### 2.2 `ScopedWakeLock` is a no-op on Linux

**File:** `util/scoped_wake_lock.h`, lines 25–27

```cpp
// TODO(issuetracker.google.com/288311411): Implement for Linux.

class ScopedWakeLock;
```

**Problem:** `ScopedWakeLock` has a macOS implementation (`scoped_wake_lock_mac.cc`) but no Linux implementation. On the most common headless embedded receiver platform, wake locks silently do nothing. Issue: 288311411.

**Recommended fix direction:** Implement a Linux `ScopedWakeLock` using the `/sys/power/wake_lock` / `/sys/power/wake_unlock` interface (for Android) or a `systemd-inhibit`/D-Bus inhibitor lock for desktop Linux. The implementation should be gated on a platform detection check in `BUILD.gn`.

---

### 2.3 TLS session resumption not implemented

**File:** `platform/impl/tls_connection_factory_posix.cc`, lines 69–75

```cpp
// TODO(issuetracker.google.com/281741213): Add support for resuming sessions.
// TODO(issuetracker.google.com/281741213): Integrate with Auth.
void TlsConnectionFactoryPosix::Connect(const IPEndpoint& remote_address,
                                        const TlsConnectOptions& options) {
  OSP_CHECK(task_runner_.IsRunningOnTaskRunner());
  TRACE_SCOPED1(TraceCategory::kSsl, "TlsConnectionFactoryPosix::Connect",
                "remote_address", remote_address.ToString());
```

**Problem:** Every TLS connection pays full handshake cost. No session tickets or PSK. Additionally, auth integration is entirely absent from the `Connect()` path. Issue: 281741213 (both items).

**Recommended fix direction:** Use BoringSSL's `SSL_CTX_set_session_cache_mode` and `SSL_set_session` APIs to enable client-side session caching. Auth integration will depend on the cast certificate chain being threaded through `TlsConnectOptions`.

---

### 2.4 Receiver-side session renegotiation not implemented

**File:** `cast/streaming/public/receiver_constraints.h`, lines 143–148

```cpp
// Returns true if all configurations supported by `other` are also
// supported by this instance.
//
// TODO(crbug.com/1356762): Implement receiver-side session renegotiation
// so we can eliminate complicated logic for constraints compatibility.
bool IsSupersetOf(const ReceiverConstraints& other) const;
```

**Problem:** The `IsSupersetOf()` method exists specifically as a workaround for the lack of receiver-side renegotiation. Without renegotiation, the receiver cannot dynamically adapt to changing network or hardware conditions mid-stream. Bug: crbug.com/1356762.

**Recommended fix direction:** Implement a renegotiation flow in `ReceiverSession` that sends a new OFFER when constraints change, following the existing OFFER/ANSWER state machine. Once implemented, the `IsSupersetOf()` constraint compatibility logic can be removed.

---

### 2.5 `TraceLoggingPlatform::LogFlow` is optional (non-pure-virtual)

**File:** `platform/api/trace_logging_platform.h`, lines 44–47

```cpp
// Log a flow event.
// TODO(crbug.com/479316209): fast-follow: make non-optional once implemented
// in Chromium.
virtual void LogFlow(TraceEvent event, FlowType type);
```

**Problem:** `LogFlow` has a default no-op implementation instead of being pure virtual. Embedders that implement `TraceLoggingPlatform` (including Chromium) will silently receive no flow events from newly Perfetto-instrumented code paths. Bug: crbug.com/479316209.

**Recommended fix direction:** Once Chromium's `TraceLoggingPlatform` override is updated to implement `LogFlow`, make this method pure virtual (`= 0`) in the base class. Any embedder that forgets to implement it will then get a compile error rather than a silent no-op.

---

### 2.6 `Error::Code` is nearly full

**File:** `platform/base/error.h`, lines 20–23

```cpp
// TODO(crbug.com/openscreen/65): Group/rename OSP-specific errors
// NOTE: new values should be added to the end of the of enum and existing
// values should not be changed.
enum class Code : int8_t {
```

**Current last value** (line 207):
```cpp
    kMessageTimeout = 110,
  };
```

**Problem:** `int8_t` allows values -128 to 127. The enum currently uses values -1 (`kAgain`) through 110 (`kMessageTimeout`), leaving only 17 positive slots before overflow. Changing the underlying type is an ABI break for embedders. Additionally, the enum is a flat mix of platform errors, CBOR errors, mDNS errors, TLS errors, Cast streaming errors, etc. — tracked for grouping at crbug.com/openscreen/65.

**Recommended fix direction:**
- Change `int8_t` to `int16_t` (coordinate with embedders — this is an ABI break).
- Or: reserve a range for each subsystem (e.g., 100–119: Cast streaming, 120–139: mDNS) to make grouping explicit without renaming.
- Separately, consider the grouping cleanup from crbug.com/openscreen/65.

---

## Priority 3 — Design Debt / FIXMEs

### 3.1 `PlatformClientPosix` has three acknowledged FIXMEs in a public header

**File:** `platform/impl/platform_client_posix.h`

**Line 35:**
```cpp
// FIXME: Remove Create and Shutdown and use the ctor/dtor directly.
class PlatformClientPosix {
```

**Lines 70–72:**
```cpp
// This method is thread-safe.
// FIXME: Rename to GetTlsDataRouter()
TlsDataRouterPosix* tls_data_router();
```

**Lines 74–76:**
```cpp
// This method is thread-safe.
// FIXME: Rename to GetUdpSocketReader()
UdpSocketReaderPosix* udp_socket_reader();
```

**Problem:** The `Create()`/`ShutDown()` static singleton API should be replaced by direct constructor/destructor use. The two method names (`tls_data_router()`, `udp_socket_reader()`) violate the `GetX()` accessor naming convention used elsewhere. These are in a **public header** used by embedders.

**Recommended fix direction:** Rename the methods first (low risk, find-and-replace in tree). Then refactor `Create`/`ShutDown` to ctor/dtor — this requires auditing all call sites that assume static lifetime.

---

### 3.2 `session_messenger()` public backdoor — marked for removal

**File:** `cast/streaming/public/sender_session.h`, lines 181–183

```cpp
// TODO(https://crbug.com/1357839): remove this as part of refactoring Chrome
// to rely on `rpc_messenger()` instead.
SenderSessionMessenger& session_messenger() { return messenger_; }
```

**Problem:** This method exposes internal `SenderSessionMessenger` state via a public accessor purely for Chrome's current dependency on it. It is explicitly marked for removal once Chrome is refactored to use `rpc_messenger()` instead. Bug: crbug.com/1357839.

**Recommended fix direction:** In Chromium, identify all callers of `sender_session->session_messenger()` and migrate them to `rpc_messenger()`. Once Chrome no longer depends on it, remove the method from `sender_session.h`.

---

### 3.3 `CreateRemotingCapabilityV2()` is pre-deprecated

**File:** `cast/streaming/public/receiver_session.h`, lines 207–209

```cpp
// Creates a ReceiverCapability version 2 object. This will be deprecated
// as part of issuetracker.google.com/184429130.
ReceiverCapability CreateRemotingCapabilityV2();
```

**Problem:** The method is documented as future-deprecated in-source. Issue 184429130 tracks the migration. No replacement is named in the file.

**Recommended fix direction:** Identify the replacement API from issue 184429130, add `[[deprecated("Use X instead")]]` to the declaration, and migrate callers in Chromium before removing.

---

### 3.4 `TlsConnectionFactoryPosix::Connect()` does not handle concurrent attempts

**File:** `platform/impl/tls_connection_factory_posix.h`, lines 37–42

```cpp
// TlsConnectionFactory overrides.
//
// TODO(jophba): Determine how to handle multiple connection attempts
// to the same remote_address, and how to distinguish errors.
void Connect(const IPEndpoint& remote_address,
             const TlsConnectOptions& options) override;
```

**Problem:** The behavior when `Connect()` is called multiple times concurrently (either to the same or different `remote_address`) is undefined. Error attribution when multiple attempts are in flight is unresolved.

**Recommended fix direction:** Add a map of in-flight connection attempts keyed by `remote_address`. On a second `Connect()` to the same address, either queue it, reject it with an error, or cancel the first — document which behavior is chosen and enforce it.

---

### 3.5 Device auth error reporting paths are missing

**File:** `cast/receiver/channel/device_auth_namespace_handler.cc`

**Line 70** — silent discard on proto parse failure:
```cpp
if (!device_auth_message.ParseFromString(payload)) {
  // TODO(btolsch): Consider all of these cases for future error reporting
  // mechanism.
  return;
}
```

**Lines 101–105** — credential failure not surfaced to application layer:
```cpp
if (tls_cert_der.empty() || device_creds.certs.empty() ||
    !device_creds.private_key) {
  // TODO(btolsch): Add this to future error reporting.
  router->Send(virtual_conn, GenerateErrorMessage(AuthError::INTERNAL_ERROR));
  return;
}
```

**Problem:** Proto parse failures are silently swallowed (no callback, no log at error level). Credential failures send an error message on the wire but are not surfaced to the application layer for handling.

**Recommended fix direction:** Route both failure cases through the `Client` interface or a dedicated error callback so that the application layer can react (e.g., close the connection, log a metric, trigger re-auth).

---

### 3.6 `span.h` rename is half-done

**File:** `platform/base/span.h`, lines 22–27

```cpp
// In Open Screen code, use these aliases for the most common types of Spans.
// TODO(crbug.com/364687926): rename to byte_view.h and remove Span alias.
using ByteView = std::span<const uint8_t>;
using ByteBuffer = std::span<uint8_t>;
template <typename T>
using Span = std::span<T>;
```

**Problem:** The primary types are now `ByteView` and `ByteBuffer`, but the file is still named `span.h` and retains the `Span<T>` template alias for compatibility. Bug: crbug.com/364687926.

**Recommended fix direction:**
1. Rename `span.h` → `byte_view.h`.
2. Add a deprecated forwarding `span.h` that `#include`s `byte_view.h` and `#pragma message`s a deprecation warning.
3. Mass-migrate `#include "platform/base/span.h"` → `byte_view.h` in tree, then remove `Span<T>` and the forwarding header.

---

### 3.7 Adaptive playout delay removed without resolution

**File:** `cast/streaming/public/receiver_session.cc`, lines 521–523

```cpp
    // TODO(crbug.com/40238532): re-add support for adaptive playout delay??
    .rtp_extensions = {}};
}
```

**Problem:** Adaptive playout delay support was previously present and was removed. The `??` in the TODO signals unresolved decision-making. Without it, the receiver cannot dynamically adapt buffer depth to network jitter. Bug: crbug.com/40238532.

**Recommended fix direction:** Resolve the open question in the bug: determine whether re-adding this feature is in scope, and if so, implement it in the RTP extensions negotiation path. If not in scope, close the bug and remove the TODO.

---

### 3.8 `ReceiverSocketFactory` lacks `TaskRunner` for sequence checking

**File:** `cast/receiver/public/receiver_socket_factory.h`, lines 31–33

```cpp
// `client` and `socket_client` must outlive `this`.
// TODO(btolsch): Add TaskRunner argument just for sequence checking.
ReceiverSocketFactory(Client& client, CastSocket::Client& socket_client);
```

**Problem:** Without a `TaskRunner` argument, there is no runtime enforcement that methods are called on the correct sequence. In debug builds this means sequence violations go undetected.

**Recommended fix direction:** Add `TaskRunner& task_runner` to the constructor and store a reference. Add `OSP_CHECK(task_runner_.IsRunningOnTaskRunner())` at the start of public methods, following the pattern used in `TlsConnectionPosix` and `CastSocket`.

---

### 3.9 `kNumFramesDroppedByEncoder` statistic silently unsupported

**File:** `cast/streaming/impl/statistics_analyzer.cc`, lines 388–391

```cpp
// TODO(b/298205111): Support kNumFramesDroppedByEncoder stat.
PopulateFrameCountStat(StatisticsEvent::Type::kFrameCaptureEnd,
                       StatisticType::kNumFramesCaptured, media_type,
                       stats_list);
```

**Problem:** The `kNumFramesDroppedByEncoder` statistic is silently omitted from the stats list. Consumers of the statistics API will see zero for this field regardless of actual encoder drops. Bug: b/298205111.

**Recommended fix direction:** Instrument the encoder path in `cast/streaming/impl/` to emit a `kFrameDroppedByEncoder` event, then wire it through to `PopulateFrameCountStat` for `kNumFramesDroppedByEncoder`.

---

## Priority 4 — Test Coverage Gaps

### 4.1 `SSL_shutdown` in `TlsConnectionPosix` destructor has no unit test

**File:** `platform/impl/tls_connection_posix.cc`, lines 47–55

```cpp
TlsConnectionPosix::~TlsConnectionPosix() {
  if (platform_client_) {
    platform_client_->tls_data_router()->DeregisterConnection(this);
  }
  // TODO(issuetracker.google.com/169966671): This is only tested by CastSocket
  // E2E tests at the moment.
  if (ssl_) {
    SSL_shutdown(ssl_.get());
  }
}
```

**Problem:** The destructor's `SSL_shutdown` path is exercised only through high-level `CastSocket` E2E tests, not through any unit test of `TlsConnectionPosix` itself. If the destructor path breaks, it will only be caught by an E2E test run. Issue: 169966671.

**Recommended fix direction:** Add a `TlsConnectionPosixTest` that creates a `TlsConnectionPosix` with a mock SSL context, verifies `SSL_shutdown` is called on destruction, and verifies `DeregisterConnection` is called on the data router.

---

### 4.2 `StatisticsAnalyzer` clock offset estimator is untested

**File:** `cast/streaming/impl/statistics_analyzer_unittest.cc`, lines 124–127

```cpp
void SetUp() {
  // In general, use an estimator that doesn't have an offset.
  // TODO(issuetracker.google.com/298085631): add test coverage for the
  // estimator usage in this class.
```

**Problem:** All unit tests for `StatisticsAnalyzer` use a zero-offset stub for the clock offset estimator. The logic that applies the estimator's output to reported statistics has no test coverage. Issue: 298085631.

**Recommended fix direction:** Add tests that inject a non-zero `FakeClockOffsetEstimator` and verify that the reported `packetLossRate`, frame timing stats, or end-to-end latency values are correctly adjusted.

---

### 4.3 `TlsConnectionFactoryPosix` concurrent `Connect()` behavior is untested

**File:** `platform/impl/tls_connection_factory_posix.h` (see issue 3.4 above)

**Problem:** Concurrent `Connect()` calls are not only unguarded (see 3.4) but are also completely untested. There is no test verifying the behavior when two `Connect()` calls race.

**Recommended fix direction:** Once the behavior is decided (see 3.4), add a unit test that calls `Connect()` twice concurrently and asserts the correct outcome (error returned, queued, or previous cancelled).

---

## Priority 5 — Code Duplication and Minor Cleanup

### 5.1 `AvErrorToString` is copy-pasted between standalone sender and receiver

**File:** `cast/standalone_receiver/decoder.cc`, lines 21–25

```cpp
// The av_err2str macro uses a compound literal, which is a C99-only feature.
// So instead, we roll our own here.
// TODO(issuetracker.google.com/224642520): dedup with standalone sender.
std::string AvErrorToString(int error_num) {
```

**File:** `cast/standalone_sender/ffmpeg_glue.h`, lines 68–72

```cpp
// The av_err2str macro uses a compound literal, which is a C99-only feature.
// So instead, we roll our own here.
// TODO(issuetracker.google.com/224642520): dedup with standalone receiver.
std::string AvErrorToString(int error_num);
```

**Problem:** `AvErrorToString()` is declared/defined twice with identical logic and comments pointing at each other. Any bug fixes or improvements must be made in both places. Issue: 224642520.

**Recommended fix direction:** Create `cast/standalone_common/BUILD.gn` and `cast/standalone_common/ffmpeg_util.{h,cc}` (or similar). Move `AvErrorToString` there. Update both standalone targets to depend on `//cast/standalone_common`.

---

### 5.2 `RemoveValueFromMap` is limited to raw-pointer maps

**File:** `util/std_util.h`, lines 32–41

```cpp
template <typename Key, typename Value>
void RemoveValueFromMap(std::map<Key, Value*>* map, Value* value) {
  for (auto it = map->begin(); it != map->end();) {
    if (it->second == value) {
      it = map->erase(it);
    } else {
      ++it;
    }
  }
}
```

**Problem:** The template hardcodes `Value*` as the mapped type, making it unusable with smart-pointer maps. This limits reuse and propagates raw-pointer map patterns.

**Recommended fix direction:** Templatize on the mapped type: `template <typename Key, typename MappedValue, typename Value>` where `Value` is the search type and `MappedValue` may be `Value*` or `std::unique_ptr<Value>`. Or, if callers are few, audit them and use `std::erase_if` (C++20) directly at the call sites.

---

### 5.3 `session_messenger()` accessor exposes internals (see also 3.2)

Already documented above as Priority 3 design debt.

---

## Cross-Cutting: Exit-Time Destructors

The Clang diagnostic `-Wexit-time-destructors` is suppressed in **six BUILD.gn files**, all sharing the same tracking bug: **crbug.com/40031409**.

Notably, in `cast/streaming/BUILD.gn`, the suppression appears in a **production (non-test) target** (`openscreen_source_set("sender")`), not just a test target — making it higher priority than the others.

| File | Target | Risk |
|---|---|---|
| `cast/streaming/BUILD.gn` | `sender` (production) | **High** |
| `cast/streaming/BUILD.gn` | `unittests` | Medium |
| `cast/common/BUILD.gn` | `unittests` | Medium |
| `cast/receiver/BUILD.gn` | `unittests` | Medium |
| `util/BUILD.gn` | `unittests` | Low |
| `platform/BUILD.gn` | `test` | Low |

**Problem:** Exit-time destructors run after `main()` returns, at which point the embedder's event loop and task runner are torn down. Non-trivial destructors in this phase can race with or access already-destroyed state, causing crashes or undefined behavior on shutdown — especially within Chromium's process model.

**Recommended fix direction:** Enable the diagnostic (remove the `configs` line) in each target, compile, and fix the resulting errors one by one:
- Replace function-local `static` instances with dynamically allocated singletons (using `new` and `std::lorgeron_ptr` or `base::NoDestructor<T>` in Chromium).
- Replace global `const std::string`/`std::vector` instances with `constexpr` arrays or raw string literals.
- For global `LogManager` or `TraceManager` singletons: convert to `base::NoDestructor<T>` wrappers.

Start with the production `sender` target in `cast/streaming/BUILD.gn`.

---

## Appendix: What Is Actually Good

The following should be preserved and used as models for future work:

- **Architecture layering**: The `base/ → api/ → impl/` dependency hierarchy enforced by GN `visibility` is clean and correct. Do not break it.
- **`ErrorOr<T>`**: Well-designed. Use this consistently — avoid adding raw `bool` + out-parameter patterns.
- **`WeakPtr` / `WeakPtrFactory`**: Correct and idiomatic. All async callbacks should use this pattern rather than raw `this` captures.
- **`Alarm`**: Clean RAII scheduled-task abstraction. Prefer it over posting raw delayed tasks.
- **Fuzzing**: Four parser fuzzers with seed corpora. Extend this when adding new network message parsers.
- **`TaskRunnerImpl`**: The A/B vector swap pattern (never hold the mutex while running tasks) is correct and should be preserved.
- **`cast/docs/architecture.md`**: High-quality architecture document. Keep it up to date as the streaming session model changes.
- **Test fakes**: `FakeClock`, `FakeTaskRunner`, `FakeUdpSocket`, `MockTlsConnection` in `platform/test/` are well-designed. Use them rather than introducing mocking at a lower level.
- **Style guide** (`docs/style_guide.md`): The no-Abseil-in-public-headers rule and the YAGNI guidance are worth enforcing.

---

*End of findings. 23 specific issues documented across `cast/`, `util/`, and `platform/`.*
