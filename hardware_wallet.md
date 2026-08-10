# Implementation Plan: Hardware-Secured Stellar Crypto Wallet in Flutter

This document outlines the step-by-step technical architecture, setup, and implementation plan for building a hardware-secured Stellar (XLM) crypto wallet in Flutter. The plan uses Android's Keystore system (`EncryptedSharedPreferences`), system authentication via `local_auth` (PIN, Pattern, Password, or Biometrics), and zero-lifetime key handling in Dart memory.

---

## 1. System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      User Action / UI                       │
└──────────────────────────────┬──────────────────────────────┘
                               │ 1. Initiate Send XLM
                               ▼
            ┌────────────────────────────────────┐
            │ local_auth (OS Auth / Biometrics)  │
            └──────────────────┬─────────────────┘
                               │ 2. Screen Lock / Fingerprint Valid
                               ▼
            ┌────────────────────────────────────┐
            │     flutter_secure_storage         │
            └──────────────────┬─────────────────┘
                               │ 3. Fetch Encrypted Seed Bytes
                               ▼
            ┌────────────────────────────────────┐
            │   Android Keystore / TEE Hardware  │
            └──────────────────┬─────────────────┘
                               │ 4. Decrypt to Uint8List
                               ▼
            ┌────────────────────────────────────┐
            │    SecureStellarSigner (Memory)    │
            │  - Build Tx                        │
            │  - Sign with KeyPair               │
            │  - wipeBytes(seed) in finally      │
            └──────────────────┬─────────────────┘
                               │ 5. Submit Envelope
                               ▼
┌─────────────────────────────────────────────────────────────┐
│               Stellar Horizon Network / Node                │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Phase-by-Phase Implementation Plan

### Phase 1: Environment & Native Configuration

Configure native Android properties to block screenshots, set up secure fragment activities, and declare core security dependencies.

#### 1. Dependencies (`pubspec.yaml`)

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_secure_storage: ^9.2.2
  local_auth: ^2.3.0
  stellar_flutter_sdk: ^0.8.6
  flutter_windowmanager: ^0.2.0
```

#### 2. Android Native Manifest (`android/app/src/main/AndroidManifest.xml`)

Disable automatic cloud backups to prevent unencrypted key synchronization and declare biometric hardware permissions:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.USE_BIOMETRIC"/>
    <application
        android:allowBackup="false"
        android:fullBackupOnly="false"
        android:label="Stellar Secure Wallet"
        android:icon="@mipmap/ic_launcher">
    </application>
</manifest>
```

#### 3. Activity Configuration (`MainActivity.kt`)

Extend `FlutterFragmentActivity` instead of standard `FlutterActivity` to host OS authentication overlays:

```kotlin
import io.flutter.embedding.android.FlutterFragmentActivity

class MainActivity: FlutterFragmentActivity()
```

---

### Phase 2: Hardware Secure Storage & Authentication Service

Implement the secure key storage layer using Android's Keystore system and enforce device lock verification prior to key access.

```dart
import 'dart:convert';
import 'dart:typed_data';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:local_auth/local_auth.dart';

class SecureVaultService {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );
  
  final LocalAuthentication _auth = LocalAuthentication();
  static const String _seedKey = 'stellar_secret_seed';

  /// Save raw seed bytes to hardware-backed storage
  Future<void> storeSeed(Uint8List seedBytes) async {
    final base64Seed = base64Encode(seedBytes);
    await _storage.write(key: _seedKey, value: base64Seed);
  }

  /// Prompt OS lock screen (PIN/Pattern/Biometrics) and retrieve seed bytes
  Future<Uint8List?> retrieveSeedWithAuth() async {
    bool authenticated = await _auth.authenticate(
      localizedReason: 'Authenticate to access your wallet keys',
      options: const AuthenticationOptions(
        stickyAuth: true,
        biometricOnly: false, // Allows fallback to device PIN / Password / Pattern
      ),
    );

    if (!authenticated) {
      throw Exception('Device authentication failed.');
    }

    final base64Seed = await _storage.read(key: _seedKey);
    if (base64Seed == null) return null;
    return base64Decode(base64Seed);
  }
}
```

---

### Example: Secure Signing Execution Layer

Combine explicit memory-wiping logic with the `stellar_flutter_sdk` inside a strict `try-finally` lifecycle to guarantee key deletion even if network or signing errors occur.

```dart
import 'dart:convert';
import 'dart:typed_data';
import 'package:stellar_flutter_sdk/stellar_flutter_sdk.dart';

class SecureStellarService {
  final StellarSDK _sdk = StellarSDK.PUBLIC; // Use StellarSDK.TESTNET for testing

  /// Overwrites all bytes in the buffer with zeroes in-place
  void _wipeBytes(Uint8List? buffer) {
    if (buffer == null) return;
    for (int i = 0; i < buffer.length; i++) {
      buffer[i] = 0;
    }
  }

  Future<SubmitTransactionResponse> executeSignedPayment({
    required Uint8List seedBytes,
    required String recipientAccountId,
    required String amountXlm,
  }) async {
    KeyPair? keyPair;

    try {
      // 1. Decode seed string from bytes
      final secretSeedStr = utf8.decode(seedBytes);
      keyPair = KeyPair.fromSecretSeed(secretSeedStr);

      // 2. Fetch sender account sequence number from Stellar Horizon
      AccountResponse account = await _sdk.accounts.account(keyPair.accountId);

      // 3. Build Payment Transaction
      Transaction tx = TransactionBuilder(account)
          .addOperation(
            PaymentOperationBuilder(
              recipientAccountId,
              Asset.NATIVE,
              amountXlm,
            ).build(),
          )
          .build();

      // 4. Sign transaction
      tx.sign(keyPair, Network.PUBLIC);

      // 5. Submit to Stellar Horizon network
      return await _sdk.submitTransaction(tx);
    } finally {
      // 6. OVERWRITE MEMORY: Zero out raw seed bytes immediately
      _wipeBytes(seedBytes);
      keyPair = null;
    }
  }
}
```

---

### Phase 4: UI & Screen Protection Integration

Incorporate `FLAG_SECURE` on critical screens to prevent screen scraping/capture and coordinate the end-to-end payment flow.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_windowmanager/flutter_windowmanager.dart';

class WalletScreen extends StatefulWidget {
  const WalletScreen({super.key});

  @override
  State<WalletScreen> createState() => _WalletScreenState();
}

class _WalletScreenState extends State<WalletScreen> {
  final _vault = SecureVaultService();
  final _stellar = SecureStellarService();

  @override
  void initState() {
    super.initState();
    _enableScreenProtection();
  }

  /// Prevent screenshots and Android task-switcher leaks
  Future<void> _enableScreenProtection() async {
    await FlutterWindowManager.addFlags(FlutterWindowManager.FLAG_SECURE);
  }

  Future<void> _handleSendPayment() async {
    try {
      // 1. Prompt OS Lock & fetch key into RAM
      final seedBytes = await _vault.retrieveSeedWithAuth();
      if (seedBytes == null) return;

      // 2. Execute transaction (seed memory wiped in 'finally' block)
      final response = await _stellar.executeSignedPayment(
        seedBytes: seedBytes,
        recipientAccountId: 'GBRPYHIL2CI3FNQ4BXLFMNDLFJUNPU2HY3ZMFXYA', // Recipient Address
        amountXlm: '5.0',
      );

      if (response.isSuccess) {
        if (!mounted) return;
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Payment Successful!')),
        );
      }
    } catch (e) {
      if (!mounted) return;
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Transaction Failed: $e')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Stellar Secure Wallet')),
      body: Center(
        child: ElevatedButton(
          onPressed: _handleSendPayment,
          child: const Text('Send 5 XLM'),
        ),
      ),
    );
  }
}
```

---

## 3. Verification & Testing Checklist

| Test Target | Validation Method | Expected Result |
| :--- | :--- | :--- |
| **Screenshot Prevention** | Attempt screenshot or screen recording while on wallet view | OS prevents capture or renders a black preview screen. |
| **Task Switcher Leakage** | Open Android recent apps view | App snapshot appears obscured or entirely blank. |
| **Authentication Fallback** | Test without registered fingerprint (PIN/Pattern fallback) | OS prompts device PIN/Pattern and successfully grants key access. |
| **Memory Cleanup Failure** | Trigger transaction network error during `submitTransaction` | `finally` block executes regardless; `seedBytes` are set to `0` across all indices. |
| **Cloud Backup Inspection** | Force Android cloud backup via ADB (`adb shell bu...`) | Key store files excluded due to `android:allowBackup="false"`. |
