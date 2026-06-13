# Open Car Contracts 🏎️📡

This repository is the **Single Source of Truth** for all data communication in the Open Car Controller project. It contains the Protocol Buffer (`.proto`) definitions that dictate exactly how the vehicle hardware and the mobile companion app talk to each other over BLE and MQTT, plus transport-level metadata (for example MQTT topic templates and BLE GATT UUIDs in `opencar/core/v1/transport.toml`).

By separating these contracts into their own repository, we ensure that both the firmware and the mobile app are mathematically guaranteed to stay in sync.

## 🏗️ Architecture: The Envelope Pattern

This project uses an **Envelope Pattern** to route messages.

1. **`proto/core.proto` (The Envelope):** Handles routing, timestamps, message IDs, and system health. It contains a `car_id` to identify the vehicle type (e.g., "hmg", "tesla"), but is otherwise agnostic to the specific commands or states within the opaque `bytes payload`.
2. **`proto/cars/*.proto` (The Payload):** Contains the actual, highly-specific state and commands for a given vehicle. These are compiled into raw bytes and placed inside the core envelope.
3. **`opencar/core/v1/transport.toml` (Transport Bindings):** Defines cross-repo transport details that are not protobuf messages themselves but still must stay in sync, such as MQTT topic templates and BLE GATT service/characteristic UUIDs.

This allows the core firmware to route messages without needing to understand the contents of the payload, keeping the codebase modular. The `car_id` allows the final destination (like the mobile app) to know which schema to use for decoding.

## 🛠️ How to Use This Repository

Do not copy-paste these files into your project. This repository is designed to be consumed as a **Git Submodule**. Refer to the firmware and mobile app to see how to consume these files.

## 🚨 Contributing Rules (CRITICAL)

Because these contracts dictate communication between physical hardware and remote apps, breaking changes can result in bricked controllers or malfunctioning apps.

If you are submitting a Pull Request to add a new car feature, you **MUST** follow the append-only rule:

1. **NEVER change a field number.** (e.g., `int32 speed = 1;` cannot become `int32 speed = 2;`).

2. **NEVER change a field's data type.**

3. **NEVER delete an existing field.** If a feature is deprecated, leave the field defined and add the `[deprecated = true]` annotation.

4. **ALWAYS append new fields to the bottom of the message.**

5. **USE `optional` for all state fields** to support partial, high-frequency updates without sending unnecessary bytes.

Protobuf is inherently forwards and backwards compatible only if you follow these rules. Older apps will safely ignore new fields, and newer apps will safely handle missing fields from older firmware.

## 🧹 Linting and Formatting

We use [`buf`](https://buf.build/) to manage our Protobuf files. It enforces styling, formatting, and prevents breaking changes.

**Recommended Setup:** This repository includes a Devcontainer configuration. If you open this project in VS Code with the Dev Containers extension (or GitHub Codespaces), the `buf` CLI will be pre-installed and ready to use.

If you prefer to work locally without the devcontainer, you will need to install the `buf` CLI manually.

### Native Windows Setup (Optional)
While the dev container is the primary supported environment, you can set up a local Windows environment by installing the necessary tools directly on your host machine.

1. **Install `buf` CLI:** We recommend using the Windows Package Manager (`winget`):
   ```powershell
   winget install Buf.Buf
   ```
2. **Install `protoc` (Protocol Buffers Compiler):** 
   ```powershell
   winget install -e --id Google.Protobuf
   ```

*(Alternatively, you can download the binaries manually from their respective GitHub release pages and add them to your system `PATH`.)*

Before submitting a PR, ensure your changes pass these checks by running:

*   **Format your files:** `buf format -w`
*   **Lint your files:** `buf lint`
*   **Check for breaking changes:** `buf breaking --against '.git#branch=main'`

## 🏷️ Versioning

This repository uses a **Single Global Version** (e.g., `v1.2.0`).

When a new feature is added to any car, the entire repository's version is bumped.
