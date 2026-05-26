# network_integration.rs — Integration tests for the makepad-network crate

**File path**: `platform/network/tests/network_integration.rs` (365 lines)
**Core purpose**: Tests WebSocket transport (PlainTcp, Platform, Auto) roundtrip, HTTPS GET requests, HTTP POST body preservation, and plain TCP socket stream large-payload echo.

## Content
- **Helpers**: `find_free_port()` binds a random local port; `wait_for_event()` polls `NetworkRuntime::recv_timeout` with a deadline; `find_header_end()` / `parse_content_length()` parse raw HTTP responses
- **Test isolation**: `test_guard()` provides a global Mutex so tests run sequentially
- **WebSocket roundtrip** (`plain_websocket_roundtrip_via_http_server`, `platform_websocket_roundtrip_via_http_server`): Starts an HTTP server, opens a WebSocket, sends a 5-byte binary payload, receives the echo from the server, then closes
- **HTTPS path** (`https_google_request_exercises_https_path`): Issues a GET to `https://makepad.nl/`, verifies a status code in [100, 600) or an error that is not "unsupported"
- **HTTP POST** (`http_post_body_roundtrip_preserves_json_payload`): Sends a JSON POST body to a local TCP echo server, asserts Content-Type and body match exactly
- **TCP socket stream** (`socket_stream_plain_tcp_large_roundtrip`): Sends 256 KiB of patterned data over `SocketStream::connect`, verifies exact echo
- All tests are `#[cfg(not(target_arch = "wasm32"))]` (native only)
