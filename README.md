# Repainting text with many emoji is slow in Safari

**Test page: https://leipert.github.io/safari-emoji-rendering-perf/**

When Safari repaints a block of text, it draws every emoji in the block again, and each emoji costs time in Safari's GPU
process. WebKit repaints a whole block when any line of its text changes, so typing in a block full of emoji redraws
all of them on every key press. A 100×100 grid of ⬛️ and ⬜️ (10,000 emoji, like a large QR code drawn with emoji)
renders at about 10 frames per second in Safari while one line changes, and each key press takes about 150 ms to
appear. Chromium on the same machine renders the same page at 60 frames per second.

After Safari had been running for about 10 days, each emoji cost about 15 times more, and a 29×29 grid (841 emoji)
was already this slow. Restarting Safari restored the speed below.

## Run it

Open the [test page](https://leipert.github.io/safari-emoji-rendering-perf/) in Safari and wait 4 seconds. The page
changes the text of the middle line of a 100×100 emoji grid on every animation frame, as typing does, and then shows
how many frames rendered. A browser that keeps up renders about 240 frames, one every 16.7 ms.

| Variant                            | Link                                                                                                                                                                                                                                         |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 10,000 emoji in one block          | [default](https://leipert.github.io/safari-emoji-rendering-perf/)                                                                                                                                                                            |
| ASCII instead of emoji             | [`?emoji=0`](https://leipert.github.io/safari-emoji-rendering-perf/?emoji=0)                                                                                                                                                                 |
| Each line in its own block element | [`?lines=block`](https://leipert.github.io/safari-emoji-rendering-perf/?lines=block)                                                                                                                                                         |
| Type in the middle line yourself   | [`?type=1`](https://leipert.github.io/safari-emoji-rendering-perf/?type=1)                                                                                                                                                                   |
| Typing, each line in its own block | [`?type=1&lines=block`](https://leipert.github.io/safari-emoji-rendering-perf/?type=1&lines=block)                                                                                                                                           |
| Smaller grids                      | [`?size=29`](https://leipert.github.io/safari-emoji-rendering-perf/?size=29), [`?size=50`](https://leipert.github.io/safari-emoji-rendering-perf/?size=50), [`?size=75`](https://leipert.github.io/safari-emoji-rendering-perf/?size=75) |

The grid is wider than most windows, so the result line also shows the viewport size. The edited line is 10 emoji
shorter than the others, so typing never changes the page width.

## Results

Measured on October 15, 2026 with:

- Safari `Version 27.0.1 (22625.1.29.11.28)`
- macOS 27.0.1 (26A434)
- MacBook Pro with an Apple M4 Pro and 48 GB

Each cell shows the frames rendered in 4 seconds and the median frame time, in a 1028×1322 viewport.

| Variant                     | Safari 27.0.1, restarted | Minimal WKWebView app (same WebKit) | Chromium 145         |
| --------------------------- | ------------------------ | ----------------------------------- | -------------------- |
| 10,000 emoji, one block     | 41 frames, 131 ms        | 40 frames, 131 ms                   | 235 frames, 17 ms    |
| 10,000 ASCII characters     | 241 frames, 17 ms        | 242 frames, 17 ms                   | 242 frames, 17 ms    |
| 10,000 emoji, block per line | 225 frames, 17 ms      | 223 frames, 17 ms                   | 242 frames, 17 ms    |
| 5,625 emoji (`?size=75`)    | 58 frames, 76 ms         | 58 frames, 76 ms                    | 241 frames, 17 ms    |
| 2,500 emoji (`?size=50`)    | 125 frames, 32 ms        | 124 frames, 32 ms                   | 241 frames, 17 ms    |
| 841 emoji (`?size=29`)      | 240 frames, 17 ms        | 240 frames, 17 ms                   | 241 frames, 17 ms    |

When typing in the middle line (`?type=1`), Safari takes a median of 152 ms from key press to frame (worst 231 ms).
With one block per line it takes 19 ms.

### Before restarting Safari

Earlier the same day, Safari's GPU process had been running for about 10 days. With an earlier version of this page
that drew a 29×29 grid, Safari rendered 25 frames in 4 seconds (median 166 ms) with 841 emoji, against 241 frames with
ASCII and 223 frames with one block per line. Typing took a median of 244 ms per key press, and 26 ms with one block
per line. The median frame time grew with the grid: 22 ms for 100 emoji, 48 ms for 225, 77 ms for 400. The minimal
WKWebView app and Chromium kept up with the same page. This reproduced on the built-in Retina
display and on the external display, in normal and private windows.

## Observations

- In a restarted Safari, each emoji in the repainted block costs about 0.013 ms. After about 10 days of uptime, it cost
  about 0.2 ms.
- Only repaints that include the emoji are slow. Changing text in a separate element directly below the grid stays at
  60 frames per second, and so does putting each line in its own block element, because only the changed line is
  repainted.
- None of these changed the result in the long-running Safari: `will-change: transform`, `will-change: filter`,
  `filter: saturate(1)`, `opacity: 0.99`, `contain: strict`, `text-rendering: optimizeSpeed`.
- The WebKit feature flags in Safari's Feature Flags window match the defaults of the WKWebView app, except for the
  Digital Credentials API.

## Profiles

Both files are `sample` output for Safari's GPU process (3 seconds at 1 ms intervals) while the default page ran.

[`safari-gpu-process-sample-fresh.txt`](safari-gpu-process-sample-fresh.txt) is from the restarted Safari and the
100×100 grid. The `CA::CG::Queue` thread spends nearly all its time uploading the emoji glyph images into a texture
mosaic again, and converting their colors on each upload:

```
CA::CG::DrawImage::draw_image                    99%
  CA::OGL::Mosaic::upload_image                  96%
    CGContextDrawImage                           78%
      img_data_lock                              68%
        convert_using_vImageConverter            48%
          vImageConvert_AnyToAny                 41%
```

[`safari-gpu-process-sample.txt`](safari-gpu-process-sample.txt) is from the Safari that had been running for about
10 days, with the 29×29 grid. The same uploads also create a new color converter for every image:

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

In that process, the `RemoteRenderingBackend` work queue waits on the CG queue in
`RemoteImageBufferSet::endPrepareForDisplay` → `CA::CG::ContextDelegate::operation_` → `_dispatch_sync_f_slow`.
