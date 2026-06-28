# edge-ai-research

This project is an iPhone app experiment for running a tiny local GGUF language model through `llama.cpp`.

The notes below explain the setup path used to make `llama.cpp` callable from a SwiftUI app. They are written as a practical tutorial for developers who are new to Swift, Xcode, and Apple app projects.

## What This Setup Does

The app uses three layers:

1. A downloaded `llama.cpp` `XCFramework` binary.
2. A small local Swift package that exposes that binary to Swift.
3. Swift app code that loads a bundled `.gguf` model, sends a prompt to `llama.cpp`, and displays the generated response in a tiny chat UI.

The important idea is that the SwiftUI app does not talk directly to a server. The model runs inside the app process through the `llama.cpp` C API.

## Requirements

Install these first:

1. Xcode from the Mac App Store or Apple Developer downloads.
2. Command Line Tools for Xcode.
3. A downloaded `llama.cpp` `XCFramework` release from:

   <https://github.com/ggml-org/llama.cpp/releases>

4. A small GGUF text model. For the demo, I used:

   `SmolLM2-135M-Instruct-Q2_K.gguf`

   from:

   <https://huggingface.co/bartowski/SmolLM2-135M-Instruct-GGUF>

For a first test, use the smallest quantized model you can tolerate. Small models are not very smart, but they are good for proving that the pipeline works.

## Vocabulary For New Xcode Users

`Xcode project`
: The `.xcodeproj` file. This is what Xcode opens to build and run the app.

`Target`
: A buildable thing inside Xcode. The iPhone app target produces the `.app`.

`Swift Package`
: A reusable Swift module. In this setup, the package wraps the `llama.cpp` binary.

`XCFramework`
: Apple's package format for compiled libraries that support multiple platforms, such as iOS device, iOS simulator, macOS, and visionOS.

`GGUF`
: The model file format used by modern `llama.cpp`.

`Bundle`
: The final app folder that gets installed on the simulator or device. Resources like `.gguf` files must be copied into the bundle before the app can load them.

## Part 1: Get llama.cpp Running As The Backend

These were the rough steps used to make `llama.cpp` available inside the app.

### 1. Download The llama.cpp XCFramework

From the `llama.cpp` GitHub releases page, download the Apple `XCFramework` binary release.

After unzipping it, you should have something like:

```text
llama.xcframework/
  Info.plist
  ios-arm64/
  ios-arm64_x86_64-simulator/
  macos-arm64_x86_64/
  ...
```

The exact platform folders can vary by release, but for iPhone simulator development you need an iOS simulator slice, usually named something like:

```text
ios-arm64_x86_64-simulator
```

For real iPhone hardware, you need:

```text
ios-arm64
```

### 2. Create A Local Swift Package Wrapper

At the repository root, create a local package folder:

```text
ModelBenchmarkCore/
  Package.swift
  Sources/
    LlamaCpp/
      LlamaCpp.swift
  Vendor/
    llama.xcframework/
```

The app target will depend on this package.

This keeps the raw downloaded framework separate from the app UI code.

### 3. Move The XCFramework Into The Package

Place the downloaded framework here:

```text
ModelBenchmarkCore/Vendor/llama.xcframework
```

This gives Swift Package Manager a stable local path to the binary.

### 4. Define The Package Manifest

The package manifest tells Swift Package Manager:

1. The package name.
2. The public product the app can import.
3. The binary target path for `llama.xcframework`.
4. The tiny Swift wrapper target that depends on the binary.

Conceptually, the manifest looks like this:

```swift
// swift-tools-version: 6.0

import PackageDescription

let package = Package(
    name: "ModelBenchmarkCore",
    platforms: [
        .iOS(.v18),
        .macOS(.v15)
    ],
    products: [
        .library(
            name: "LlamaCpp",
            targets: ["LlamaCpp"]
        )
    ],
    targets: [
        .target(
            name: "LlamaCpp",
            dependencies: ["llama"]
        ),
        .binaryTarget(
            name: "llama",
            path: "Vendor/llama.xcframework"
        )
    ]
)
```

The product name `LlamaCpp` is what the app imports later:

```swift
import LlamaCpp
```

### 5. Add A Small Swift Wrapper Around The C Module

The binary framework exposes C functions from `llama.cpp`.

The Swift wrapper exists for two reasons:

1. It re-exports the generated C module so the app can use the C symbols.
2. It provides a small, friendly Swift entry point for backend startup and shutdown.

The wrapper shape is:

```swift
@_exported import llama

public enum LlamaCpp {
    public static var systemInfo: String {
        String(cString: llama_print_system_info())
    }

    public static func initializeBackend() {
        llama_backend_init()
    }

    public static func shutdownBackend() {
        llama_backend_free()
    }
}
```

The important line is:

```swift
@_exported import llama
```

That means any Swift file that imports `LlamaCpp` can also see the underlying `llama.cpp` C API types and functions.

### 6. Link The Package In Xcode

Open the app project in Xcode.

Then:

1. Select the project in the left sidebar.
2. Select the app target.
3. Open the `General` tab.
4. Find `Frameworks, Libraries, and Embedded Content`.
5. Add the local package product named `LlamaCpp`.

If Xcode has not seen the package yet:

1. Use `File > Add Package Dependencies`.
2. Choose `Add Local...`.
3. Select the `ModelBenchmarkCore` folder.
4. Add the `LlamaCpp` product to the app target.

After this, the app should be able to compile files that contain:

```swift
import LlamaCpp
```

### 7. Verify Backend Startup Before Loading A Model

Before trying inference, start with a minimal smoke test.

The app should be able to call:

```swift
LlamaCpp.initializeBackend()
let info = LlamaCpp.systemInfo
LlamaCpp.shutdownBackend()
```

This proves:

1. The package is linked.
2. The framework is embedded correctly.
3. Swift can call into the `llama.cpp` C API.

If this fails, fix package linking before moving on to model loading.

## Part 2: Bundle And Load A GGUF Model

### 1. Add The Model File To The App

Create a folder inside the app source tree:

```text
ModelBenchmark/ModelBenchmark/Models/
```

Put the downloaded GGUF file there:

```text
ModelBenchmark/ModelBenchmark/Models/SmolLM2-135M-Instruct-Q2_K.gguf
```

In Xcode, make sure the model is included in the app target. If the file is not part of the target, it will not be copied into the final app bundle.

### 2. Load The Model URL From The Bundle

At runtime, the app does not load the model from the source folder. It loads the copy inside the installed app bundle.

Use:

```swift
Bundle.main.url(
    forResource: "SmolLM2-135M-Instruct-Q2_K",
    withExtension: "gguf"
)
```

If that returns `nil`, the model was not copied into the app bundle.

Common causes:

1. The file is not included in the app target.
2. The filename does not match exactly.
3. The file is nested in a bundle subdirectory but the code is looking at the root.

### 3. Load The Model With llama.cpp

The backend loading sequence is:

1. Call `llama_backend_init()`.
2. Create default model parameters with `llama_model_default_params()`.
3. Load the `.gguf` file with `llama_model_load_from_file(...)`.
4. Create context parameters with `llama_context_default_params()`.
5. Create a context with `llama_init_from_model(...)`.
6. Create a sampler chain for generation.

The rough Swift flow is:

```swift
LlamaCpp.initializeBackend()

var modelParams = llama_model_default_params()
modelParams.n_gpu_layers = 0
modelParams.use_mmap = true

let model = llama_model_load_from_file(modelURL.path, modelParams)

var contextParams = llama_context_default_params()
contextParams.n_ctx = 512
contextParams.n_batch = 512
contextParams.n_ubatch = 512

let context = llama_init_from_model(model, contextParams)
```

For the tiny demo, GPU layers were set to `0` so the first version runs on CPU. That is slower, but simpler to debug.

## Part 3: Set Up The App API

The app-side API was set up in two layers:

1. `LlamaCpp`, the small package-level facade.
2. `LlamaChatEngine`, the app-level inference engine.

### Layer 1: The Package Facade

The package facade exposes three simple calls.

```swift
LlamaCpp.initializeBackend()
```

Starts the global `llama.cpp` backend. Call this before loading a model.

```swift
LlamaCpp.shutdownBackend()
```

Shuts down the global backend. Call this when the engine is deallocated or the app is done with local inference.

```swift
LlamaCpp.systemInfo
```

Returns a string describing the compiled backend and hardware features. This is useful for debugging whether the framework is actually running.

### Layer 2: The Chat Engine

`LlamaChatEngine` is an `actor`.

In Swift, an `actor` protects mutable state from being accessed by multiple tasks at the same time. That matters here because the model, context, sampler, and token buffers should not be mutated by two generations at once.

The app calls one main method:

```swift
let reply = try await engine.generateReply(for: messages)
```

That method hides all of the lower-level `llama.cpp` steps:

1. Load the model if it is not loaded yet.
2. Convert chat messages into a prompt string.
3. Tokenize the prompt.
4. Decode the prompt into the context.
5. Sample output tokens one at a time.
6. Convert output tokens back into text.
7. Stop at an end token or response limit.
8. Return cleaned text to the UI.

## Part 4: Function Breakdown

This section explains the important app-side pieces.

### `ChatRole`

```swift
enum ChatRole: String, Sendable {
    case user
    case assistant
}
```

Represents who wrote a message.

The raw string values are used when building the prompt:

```text
user
assistant
```

### `ChatMessage`

```swift
struct ChatMessage: Identifiable, Equatable, Sendable
```

Represents one visible chat message.

Important fields:

```swift
let id = UUID()
```

Gives SwiftUI a stable identity for each message row.

```swift
let role: ChatRole
```

Stores whether the message came from the user or assistant.

```swift
let text: String
```

Stores the message content.

```swift
let includeInPrompt: Bool
```

Controls whether a message should be sent back into the model as conversation history.

This is useful because some assistant messages are UI-only, such as:

```text
Bundled GGUF model was not found.
```

Those should be shown to the user, but not included in the next model prompt.

### `ChatViewModel`

```swift
@MainActor
final class ChatViewModel: ObservableObject
```

Owns the state for the chat screen.

`@MainActor` means its UI state is updated on the main thread. SwiftUI expects that.

`ObservableObject` means SwiftUI can watch it for changes and redraw the screen.

#### `messages`

```swift
@Published var messages: [ChatMessage]
```

The list of messages shown in the chat.

`@Published` tells SwiftUI to refresh when the list changes.

#### `draft`

```swift
@Published var draft = ""
```

The text currently typed into the message field.

#### `status`

```swift
@Published var status = "Model loads on first send."
```

A short status line shown above the input box.

Example values:

```text
Model loads on first send.
Thinking...
Ready
Failed
```

#### `isGenerating`

```swift
@Published var isGenerating = false
```

Prevents sending multiple prompts at once and lets the UI disable the send button while the model is working.

#### `engine`

```swift
private let engine: LlamaChatEngine?
```

Holds the local inference engine.

It is optional because model lookup can fail. If the app cannot find the GGUF in the bundle, `engine` is `nil`.

#### `init()`

```swift
init()
```

Runs when the chat screen model is created.

It looks for the bundled GGUF model:

```swift
Bundle.main.url(
    forResource: "SmolLM2-135M-Instruct-Q2_K",
    withExtension: "gguf"
)
```

If found, it creates:

```swift
LlamaChatEngine(modelURL: modelURL)
```

If not found, it updates the UI status so the developer knows the app bundle is missing the model.

#### `send()`

```swift
func send()
```

Runs when the user taps the send button or presses return.

It does the UI-side workflow:

1. Trim empty whitespace.
2. Ignore the send if the draft is empty.
3. Ignore the send if a generation is already running.
4. Show an error if the model engine is missing.
5. Append the user message to the chat.
6. Create a transcript of messages that should be included in the prompt.
7. Start an async task.
8. Await `engine.generateReply(for:)`.
9. Append the assistant response.
10. Update the status line.
11. Re-enable the input controls.

This method is intentionally small enough that UI code does not need to know about tokenization, contexts, samplers, or C pointers.

### `ContentView`

```swift
struct ContentView: View
```

The main SwiftUI screen.

It creates the view model:

```swift
@StateObject private var chat = ChatViewModel()
```

`@StateObject` means SwiftUI owns this object for the lifetime of the screen.

The screen contains:

1. A navigation title.
2. A scrollable message list.
3. A `Thinking...` indicator.
4. A status line.
5. A text input.
6. A send button.

### `MessageBubble`

```swift
private struct MessageBubble: View
```

Draws one message bubble.

User messages are aligned to the trailing side and use the accent color.

Assistant messages are aligned to the leading side and use a secondary background color.

This view is display-only. It does not call the model.

### `LlamaChatError`

```swift
enum LlamaChatError: LocalizedError
```

Defines readable errors for common failure points.

Cases include:

```swift
modelLoadFailed(String)
```

The `.gguf` file path was found, but `llama.cpp` could not load it.

```swift
contextCreateFailed
```

The model loaded, but the runtime context could not be created.

```swift
samplerCreateFailed
```

The sampler chain could not be created.

```swift
tokenizationFailed
```

The prompt could not be converted into model tokens.

```swift
promptTooLong(Int)
```

The prompt is larger than the tiny demo context allows.

```swift
decodeFailed(Int32)
```

`llama_decode` returned an error code.

### `LlamaChatEngine`

```swift
actor LlamaChatEngine
```

Owns the low-level `llama.cpp` state.

Important stored properties:

```swift
private let modelURL: URL
```

The app-bundle URL for the `.gguf` file.

```swift
private var model: OpaquePointer?
private var context: OpaquePointer?
private var vocab: OpaquePointer?
```

Pointers to `llama.cpp` runtime objects.

```swift
private var sampler: UnsafeMutablePointer<llama_sampler>?
```

The sampler chain used to choose the next token.

```swift
private var didInitializeBackend = false
```

Tracks whether this engine started the backend, so it knows whether to shut it down later.

```swift
private let contextSize: Int32 = 512
private let maxPromptTokens: Int32 = 384
private let maxResponseTokens: Int32 = 96
```

Small limits for a tiny local demo.

#### `init(modelURL:)`

```swift
init(modelURL: URL)
```

Stores the model path.

It does not immediately load the model. The first send loads the model lazily.

Lazy loading keeps app startup fast and makes model-loading failures show up when the user first tries inference.

#### `deinit`

```swift
deinit
```

Frees all `llama.cpp` resources when the engine is destroyed:

1. Free the sampler.
2. Free the context.
3. Free the model.
4. Shut down the backend if this engine initialized it.

This matters because model files and contexts can use a lot of memory.

#### `generateReply(for:)`

```swift
func generateReply(for messages: [ChatMessage]) throws -> String
```

The main inference method.

It performs the full generation loop:

1. Ensure the model, context, vocab, and sampler are loaded.
2. Clear previous context memory.
3. Reset the sampler.
4. Build a prompt from recent chat messages.
5. Tokenize the prompt.
6. Reject the prompt if it is too long.
7. Decode the prompt tokens.
8. Repeatedly sample one token.
9. Stop if the token is an end-of-generation token.
10. Convert each token piece into text.
11. Decode each sampled token back into the context.
12. Clean stop markers from the final text.

This is the method the UI calls.

#### `ensureLoaded()`

```swift
private func ensureLoaded() throws
```

Initializes the backend and loads the model if needed.

It is safe to call before every generation because it returns early after the model has already loaded.

The setup sequence is:

1. `LlamaCpp.initializeBackend()`
2. `llama_model_default_params()`
3. `llama_model_load_from_file(...)`
4. `llama_context_default_params()`
5. `llama_init_from_model(...)`
6. `llama_sampler_chain_default_params()`
7. `llama_sampler_chain_init(...)`
8. Add top-k, top-p, temperature, and distribution samplers.

The demo sampler settings were:

```text
top_k = 40
top_p = 0.9
temperature = 0.7
```

These are simple general-purpose settings, not tuned for quality.

#### `makePrompt(from:)`

```swift
private func makePrompt(from messages: [ChatMessage]) -> String
```

Turns chat messages into one prompt string.

For SmolLM2 Instruct, the prompt uses this style:

```text
<|im_start|>system
You are a concise helpful assistant.<|im_end|>

<|im_start|>user
Hello<|im_end|>
<|im_start|>assistant
```

The final assistant tag tells the model to continue as the assistant.

The demo only keeps the most recent messages so the prompt stays within the small context window.

#### `tokenize(_:)`

```swift
private func tokenize(_ text: String) throws -> [llama_token]
```

Converts the prompt string into token IDs that the model understands.

It calls `llama_tokenize` twice:

1. First call asks how many tokens are needed.
2. Second call writes the tokens into a Swift array.

This two-step pattern is common when calling C APIs from Swift.

#### `decode(_:context:position:)`

```swift
private func decode(
    _ tokens: [llama_token],
    context: OpaquePointer,
    position: inout Int32
) throws
```

Feeds tokens into the model context.

It creates a `llama_batch`, fills in token IDs and positions, then calls:

```swift
llama_decode(context, batch)
```

The `position` value tracks where the next token belongs in the sequence.

#### `piece(for:)`

```swift
private func piece(for token: llama_token) -> String
```

Converts one generated token back into readable text.

It calls:

```swift
llama_token_to_piece(...)
```

Some tokens are whole words. Some are partial words. Some are spaces or punctuation. The final reply is built by appending each piece.

#### `clean(_:)`

```swift
private func clean(_ text: String) -> String
```

Removes stop markers that may appear in the generated text.

Examples:

```text
<|im_end|>
<|im_start|>
</s>
```

It also trims leading and trailing whitespace before returning the reply to the UI.

## Part 5: First Response Checklist

Use this checklist when bringing the project up from zero.

1. Download the `llama.cpp` `XCFramework`.
2. Create the `ModelBenchmarkCore` local Swift package.
3. Put `llama.xcframework` under `ModelBenchmarkCore/Vendor/`.
4. Add the binary target to `Package.swift`.
5. Add the `LlamaCpp` wrapper source file.
6. Add the local package to the Xcode app target.
7. Confirm `import LlamaCpp` builds.
8. Call `LlamaCpp.initializeBackend()` and `LlamaCpp.shutdownBackend()` in a smoke test.
9. Download a tiny `.gguf` model.
10. Add the `.gguf` file to the app target.
11. Confirm `Bundle.main.url(...)` finds the model.
12. Load the model with `llama_model_load_from_file`.
13. Create a context with `llama_init_from_model`.
14. Build a prompt string.
15. Tokenize the prompt.
16. Decode prompt tokens.
17. Sample output tokens.
18. Convert tokens back into text.
19. Show the text in the SwiftUI chat view.

## Troubleshooting

### `No such module 'LlamaCpp'`

The app target is not linked to the local Swift package product.

Fix this in Xcode by adding the `LlamaCpp` package product to the app target.

### `No such module 'llama'`

The Swift package can see `LlamaCpp`, but the binary target is not exposing the underlying C module.

Check:

1. The `llama.xcframework` path is correct.
2. The framework contains headers.
3. The framework contains a module map.
4. The binary target name in `Package.swift` matches the dependency name.

### The App Builds But Cannot Find The Model

`Bundle.main.url(...)` is returning `nil`.

Check:

1. The `.gguf` filename.
2. The app target membership for the model file.
3. Whether Xcode copied the file to the bundle root or into a subdirectory.

### The Model Loads But Responses Are Empty

Possible causes:

1. The prompt format does not match the model.
2. The model produced an end token immediately.
3. The max response token limit is too low.
4. The context was not decoded correctly before sampling.

### The Prompt Is Too Long

The demo context is intentionally tiny.

Fix options:

1. Keep fewer chat messages.
2. Increase `contextSize`.
3. Use a smaller system prompt.
4. Use a model with a larger supported context.

### Simulator Build Works But Device Build Fails

The `XCFramework` may contain the simulator slice but not the real iOS device slice.

Check that the framework contains:

```text
ios-arm64
```

for real iPhone builds.

## Practical Notes

The tiny model is for proving the path, not for high-quality answers.

Bundling GGUF files increases app size quickly. If you commit models to git, consider Git LFS.

The lowest-risk development order is:

1. Link `llama.cpp`.
2. Start and stop the backend.
3. Load the model.
4. Tokenize one prompt.
5. Decode one prompt.
6. Generate one short response.
7. Build the chat UI around the working engine.
