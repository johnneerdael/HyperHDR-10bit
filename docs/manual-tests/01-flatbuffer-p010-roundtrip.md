# Manual test: P010 flatbuffer round-trip

**Prereqs:** Built HyperHDR binary from this branch (`p010-wire-format`); a configured LED instance with a visible LED layout (e.g. four LEDs in a row); a client that can send a P010 frame over the flatbuffer port (default 19400). The simplest such client is the Android `HyperHDR-android` app once its v2 P010 client lands; until then, a small standalone test client works.

## Regression check (NV12 still works)

1. Run HyperHDR: `./build/bin/hyperhdr` (or your platform's launch path).
2. From an existing client (e.g. `HyperHDR-android` v1.0.0), connect and send NV12 frames.
3. Confirm LEDs follow as before.

If NV12 stops working: the regression is in this branch. Roll back and inspect.

## P010 path

The P010 path requires a client that sends `P010Image` flatbuffer frames. Until the Android client supports this, use a small Qt test program that fakes one:

```cpp
// scratch/p010_test_client.cpp — sketch only, build separately
#include <QTcpSocket>
#include <flatbuffers/flatbuffers.h>
#include "hyperhdr_request_generated.h"

int main() {
    QTcpSocket s;
    s.connectToHost("127.0.0.1", 19400);
    s.waitForConnected();

    // Build a 4x2 synthetic P010 frame: solid white-ish.
    // P010 luma: full intensity in the high 10 bits (= 0xFFC0 in a 16-bit
    // little-endian word). For a smoke test the actual values don't matter —
    // we just want to see the server log "Received first P010 frame" and not
    // error out on the size/stride checks.
    flatbuffers::FlatBufferBuilder b(1024);
    std::vector<uint8_t> y(4 * 2 * 2, 0xC0);   // 4*2 samples * 2 bytes/sample = 16 bytes
    std::vector<uint8_t> uv(4 * 1 * 2, 0x80);  // 4*1 samples * 2 bytes/sample = 8 bytes
    auto yvec = b.CreateVector(y);
    auto uvvec = b.CreateVector(uv);
    auto p010 = hyperhdrnet::CreateP010Image(b, yvec, uvvec, 4, 2, 8, 8);
    auto image = hyperhdrnet::CreateImage(b, hyperhdrnet::ImageType_P010Image, p010.Union(), -1);

    // Register first
    auto origin = b.CreateString("p010-smoke-test");
    auto reg = hyperhdrnet::CreateRegister(b, origin, 100);
    auto regReq = hyperhdrnet::CreateRequest(b, hyperhdrnet::Command_Register, reg.Union());
    b.Finish(regReq);

    auto sendFrame = [&s](flatbuffers::FlatBufferBuilder& builder) {
        uint32_t size = htonl(builder.GetSize());
        s.write(reinterpret_cast<const char*>(&size), 4);
        s.write(reinterpret_cast<const char*>(builder.GetBufferPointer()), builder.GetSize());
        s.flush();
        s.waitForBytesWritten();
    };
    sendFrame(b);

    // Then the P010 frame
    flatbuffers::FlatBufferBuilder b2(1024);
    auto yvec2 = b2.CreateVector(y);
    auto uvvec2 = b2.CreateVector(uv);
    auto p010_2 = hyperhdrnet::CreateP010Image(b2, yvec2, uvvec2, 4, 2, 8, 8);
    auto image_2 = hyperhdrnet::CreateImage(b2, hyperhdrnet::ImageType_P010Image, p010_2.Union(), -1);
    auto imgReq = hyperhdrnet::CreateRequest(b2, hyperhdrnet::Command_Image, image_2.Union());
    b2.Finish(imgReq);
    sendFrame(b2);

    s.close();
}
```

(This is a sketch; build it against the same flatbuffers + qt libs HyperHDR uses.)

4. Run the test client.
5. Watch HyperHDR's stdout/log. Expected output (first time):

```
[FlatBufferServer] Debug: Received first P010 frame. First plane size: 16 (stride: 8). Second plane size: 8 (stride: 8). Image size: 24 (4 x 2)
```

6. Subsequent frames just route through the LUT pipeline silently.

## Pass criteria

- HyperHDR logs the "Received first P010 frame" line at info/debug level.
- HyperHDR does NOT log "The P010 image data size … does not match" — that means our size math in `FlatBuffersServer.cpp` is wrong.
- HyperHDR does NOT log "Unsupported flatbuffers image format" — that means the dispatch never matched our new branch.
- LEDs change colour (or blink, depending on the test frame's content). With a synthetic 4×2 solid frame the visible result is "all four LEDs the same colour" — not exciting, but proves the pipeline ran.

## Fail modes

- **"Unsupported flatbuffers image format"**: the parser branch isn't matching. Verify `FlatBuffersParser.cpp` sets `image.format = FLATBUFFERS_IMAGE_FORMAT::P010` correctly.
- **"P010 image data size does not match"**: size validation in the server is mismatched against the client's frame. Recompute the expected size — for `width × height` P010, total bytes = `width * height * 3`.
- **"P010 image data contains incorrect stride"**: the test client sent strides ≠ `width * 2`. Either fix the test client or relax the validation if you have a real-world client that legitimately sends different strides.
- **HyperHDR crashes**: `FrameDecoder` may not actually be P010-ready in this build path even though the Linux V4L2 grabber feeds it. Capture the stack trace and treat as a separate bug.
