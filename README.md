# Repainting text with many emoji is slow in Safari

**Test page: https://leipert.github.io/safari-emoji-rendering-perf/**

When Safari repaints a block of text that contains many Apple Color Emoji, each emoji costs about 0.2 ms in Safari's
GPU process. Editing one line of a block with 841 emoji (a QR code drawn with ⬛️ and ⬜️, as people paste into chat and
comment fields) drops Safari to about 6 frames per second, and each key press takes about 250 ms to appear. Chromium on
the same machine keeps up, and so does a minimal app that embeds the same WebKit in a WKWebView.

## Run it

Open the [test page](https://leipert.github.io/safari-emoji-rendering-perf/) in Safari and wait 4 seconds. The page
changes the text of the middle line of a 29×29 emoji grid on every animation frame, as typing does, and then shows how
many frames rendered. A browser that keeps up renders about 240 frames, one every 16.7 ms.

| Variant                                  | Link                                                                                                                                                                                                                                         |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 841 emoji in one block                   | [default](https://leipert.github.io/safari-emoji-rendering-perf/)                                                                                                                                                                            |
| ASCII instead of emoji                   | [`?emoji=0`](https://leipert.github.io/safari-emoji-rendering-perf/?emoji=0)                                                                                                                                                                 |
| Each line in its own block element       | [`?lines=block`](https://leipert.github.io/safari-emoji-rendering-perf/?lines=block)                                                                                                                                                         |
| Type in the middle line yourself         | [`?type=1`](https://leipert.github.io/safari-emoji-rendering-perf/?type=1)                                                                                                                                                                   |
| Typing, each line in its own block       | [`?type=1&lines=block`](https://leipert.github.io/safari-emoji-rendering-perf/?type=1&lines=block)                                                                                                                                           |
| Smaller grids                            | [`?size=10`](https://leipert.github.io/safari-emoji-rendering-perf/?size=10), [`?size=15`](https://leipert.github.io/safari-emoji-rendering-perf/?size=15), [`?size=20`](https://leipert.github.io/safari-emoji-rendering-perf/?size=20) |

## Results

Measured on October 15, 2026 with Safari 27.0.1 (22625.1.29.11.28) on macOS 27.0.1 (26A434), on a MacBook Pro with an
Apple M4 Pro and 48 GB. It reproduces on the built-in Retina display and on an external ASUS ProArt PA27JCV, in normal
and private windows.

| Browser                                           | 841 emoji, one block  | 841 ASCII characters  | 841 emoji, one block per line |
| ------------------------------------------------- | --------------------- | --------------------- | ----------------------------- |
| Safari 27.0.1                                     | 25 frames, p50 166 ms | 241 frames, p50 17 ms | 223 frames, p50 17 ms         |
| Minimal WKWebView app (same WebKit, same machine) | 241 frames, p50 17 ms | 242 frames, p50 17 ms | 240 frames, p50 17 ms         |
| Chromium 145 (120 Hz display)                     | 423 frames, p50 8 ms  | 481 frames, p50 8 ms  | 482 frames, p50 8 ms          |

Frames rendered in 4 seconds, with the median frame time.

When typing in the middle line (`?type=1`), Safari takes a median of 244 ms from key press to frame (worst 449 ms). With
one block per line it takes 26 ms.

## Observations

- The cost grows with the number of emoji in the repainted block: a median frame time of 22 ms for 100 emoji, 48 ms for
  225, 77 ms for 400, and about 165 ms for 841, or about 0.2 ms per emoji per repaint.
- Changing text outside the emoji block, directly below it, stays at 60 fps. Only repaints that include the emoji are
  slow. WebKit repaints a whole block when any line of its text changes, so splitting the lines into separate block
  elements limits the repaint to one line and avoids the slowdown.
- None of these changed the result: `will-change: transform`, `will-change: filter`, `filter: saturate(1)`,
  `opacity: 0.99`, `contain: strict`, `text-rendering: optimizeSpeed`.
- The WebKit feature flags in Safari's Feature Flags window match the defaults of the WKWebView app, except for the
  Digital Credentials API.
- Safari's GPU process had been running for about 10 days when we measured. We have not checked whether a freshly
  launched Safari reproduces this.

## Profile

[`safari-gpu-process-sample.txt`](safari-gpu-process-sample.txt) is a `sample` of Safari's GPU process (3 seconds at
1 ms intervals) while the default page ran. Its `CA::CG::Queue` thread spends nearly all its time drawing the emoji
glyph images, and most of that creating a new color converter for each image:

```
CA::CG::DrawImage::draw_image                                    99%
  CA::OGL::Mosaic::upload_image                                  99%
    CGContextDrawImage                                           97%
      img_image                                                  90%
        CGDataProviderCreateForDestinationWithImage              90%
          icc_converter_info                                     75%
            CGCMSConverterCreateCachedCGvImageConverter          75%
              vImageConverter_CreateWithCGColorConversionInfo    64%
                PiecewiseGamma_ComputeTable16Q12                 59%
```

The `RemoteRenderingBackend` work queue waits on that queue in `RemoteImageBufferSet::endPrepareForDisplay` →
`CA::CG::ContextDelegate::operation_` → `_dispatch_sync_f_slow`.

In the WKWebView app, the same image draws take `CGSImageDataLock` → `img_data_lock` → `convert_using_vImageConverter`,
reusing an existing converter, and the CG queue is mostly idle.
