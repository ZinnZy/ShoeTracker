# MediaPipe Tasks Vision (vendored)

Used only for cutting a shoe out of a photo, on the device. Loaded the first time a photo is cut out, never at app start.

- `vision_bundle.mjs`, `wasm/`: `@mediapipe/tasks-vision` 0.10.14 from npm. Apache License 2.0, https://github.com/google-ai-edge/mediapipe
- `magic_touch.tflite`: MediaPipe Interactive Segmenter model ("magic touch", float32, version 1), from
  https://storage.googleapis.com/mediapipe-models/interactive_segmenter/magic_touch/float32/1/magic_touch.tflite
  Apache License 2.0, https://ai.google.dev/edge/mediapipe/solutions/vision/interactive_segmenter
