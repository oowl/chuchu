# Bitwarden Password Manager SSH key integration

## Scope

Load SSH key items from a user's Bitwarden Password Manager vault when Chuchu starts. This is distinct from Bitwarden Secrets Manager and from the desktop SSH Agent.

## Findings (2026-11-07)

- Bitwarden's Android app does not expose its vault session or SSH private keys to another Android app. Chuchu must authenticate and synchronize independently.
- The [password manager SDK](https://github.com/bitwarden/sdk-internal) has Kotlin bindings through UniFFI and can decrypt `SshKeyView.private_key`, but Bitwarden calls it an unsupported, unstable internal interface.
- The SDK's [Android library](https://github.com/bitwarden/sdk-internal/blob/main/crates/bitwarden-uniffi/kotlin/sdk/build.gradle) declares `minSdk = 28`; Chuchu's Android floor is raised to 28 on this branch.
- The open source SDK is [GPL-3.0-only](https://github.com/bitwarden/sdk-internal/blob/main/crates/bitwarden-uniffi/Cargo.toml). Chuchu is MIT licensed. Distributing an APK linked with this SDK needs a GPL-compliant combined distribution, including corresponding source and notices. See the [GNU GPL FAQ](https://www.gnu.org/licenses/gpl-faq.html#WhatDoesCompatMean).
- Bitwarden publishes the Android artifact through GitHub Packages. Its [Android build instructions](https://github.com/bitwarden/android#setup) require a GitHub personal access token with `read:packages`, or the SDK can be compiled and published to Maven local from source.
- The SDK's [sample Android app](https://github.com/bitwarden/sdk-internal/blob/main/crates/bitwarden-uniffi/kotlin/app/src/main/java/com/bitwarden/myapplication/MainActivity.kt) explicitly says it is not production ready. It demonstrates separate login, crypto initialization, and vault sync; it does not provide a reusable secure session implementation.
- The current [new password-login implementation](https://github.com/bitwarden/sdk-internal/blob/main/crates/bitwarden-auth/src/login/login_via_password/mod.rs) documents that two-factor authentication is not yet supported. Chuchu must implement the complete mobile login flow around the SDK for accounts with 2FA, rather than use that entry point alone.
- The UniFFI `Client` uses client-managed tokens and does not expose the SDK's Rust `new_with_sync` constructor. The Android host must own token refresh, synchronization, and storage of encrypted vault items.
- Chuchu currently persists private key PEM in Room and server profiles refer to a local key ID. Directly copying all Bitwarden private keys into that table would create a second persistent plaintext copy, so the integration needs a key-source model and an explicit cache policy.

## Proposed implementation

1. Raise the supported Android floor to 28 and plan a GPL-compliant combined distribution.
2. Pin an exact SDK commit/artifact version. Build or resolve its Kotlin AAR and native libraries for every APK ABI; include license notices and source/build instructions.
3. Add a Bitwarden account setup screen for server region or self-hosted URL, email, password, and two-factor challenge. Never reuse the installed Bitwarden app's session or ask users to paste vault exports.
4. Store refresh credentials only with Android Keystore protection, handle token expiry and logout, and keep unlocked vault keys in memory. Do not log tokens, private keys, or decrypted vault responses.
5. On app start, refresh the session and sync vault items off the UI thread. Decrypt only SSH key items; surface authentication, network, and decryption errors without blocking local keys or app launch.
6. Represent Bitwarden keys by vault item ID, with per-host selection and in-memory private key resolution at connect time. Update the key list UI to show source and sync state. Do not silently copy Bitwarden private keys into Chuchu's Room key table or backup export.
7. Test password and two-factor login, fresh and expired sessions, offline startup, key rotation/deletion, self-hosted TLS, each supported ABI, and Android 28. Verify that private keys do not enter logs, backups, or unencrypted preferences.

## Current gate (2026-11-07)

The Android version floor and GPL distribution policy were accepted by the user. The user's fork is `oowl/chuchu`, configured as the local `fork` remote on `feat/bitwarden-ssh-keys`. The branch is fast-forwarded to the fork's `main` at `00f9fd2` before feature edits.

The Android minimum version has been raised to 28. No SDK dependency or application code has been added yet. This workspace lacks an Android SDK and a Bitwarden AAR. The AAR is published through GitHub Packages and returns HTTP 401 without package credentials; it can alternatively be compiled from source with the Android SDK/NDK. GitHub Actions exposes native `.so` artifacts, but these do not include the Kotlin AAR. An APK build cannot be verified until the Android toolchain and AAR are available. The connected GitHub account has read-only access to `oowl/chuchu`, so pushing this branch also requires a credential with write access.
