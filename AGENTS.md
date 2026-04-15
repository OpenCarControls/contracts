# Context: Open Car Contracts (Protobuf)

## 📌 Project Role
This repository is the **Single Source of Truth** for all data structures bridging the Rust firmware (ESP32) and the mobile companion app. It is consumed as a Git Submodule by both the firmware and the phone app repositories. 

## 🏗️ Architecture: The Envelope Pattern
We use an opaque payload architecture to keep the core firmware entirely agnostic to vehicle-specific features.
*   **`proto/core.proto` (The Envelope):** Handles routing, message IDs, and timestamps. It contains a `car_id` (e.g., "hmg") so the receiver knows how to decode the otherwise opaque `bytes payload`.
*   **`proto/cars/*.proto` (The Payload):** Vehicle-specific states and commands (e.g., `hmg.proto`, `tesla.proto`). These are compiled to bytes and stuffed into the core envelope's payload field.
*   **`opencar/core/v1/transport.toml` (Transport Metadata):** Generic cross-platform transport bindings shared by firmware and the app, such as MQTT topic templates and BLE GATT service/characteristic UUIDs. Treat these values as wire-level contract data just like the `.proto` schemas.

## 📜 Strict Protobuf Rules
*   **Single Global Version:** The entire repository is versioned as one (e.g., `v1.2.0`).
*   **Append-Only:** To maintain forward and backward compatibility, you must follow these rules:
    *   Never change a field number.
    *   Never change a field's data type.
    *   Never delete an existing field. Instead, mark it with `[deprecated = true]`.
    *   Always append new fields to the bottom of a message.
*   **Optional Fields:** Every state field must be marked `optional`. This enables "partial updates" (sending just a 3-byte speed update instead of the full state) which the phone app natively merges into its master state.
*   **Transport Agnostic:** These definitions are identical whether transmitted over BLE (local) or MQTT (remote over LTE).
