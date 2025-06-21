# Plan: Building and Testing Tantivy for Android and iOS

## Overview
This plan outlines the steps to build Tantivy for Android and iOS platforms and create a separate project to run tests on these devices.

## Phase 1: Prepare the Build Environment

### 1.1 Set Up Rust Cross-Compilation Tools
```bash
# Install Rust targets for Android
rustup target add \
    aarch64-linux-android \
    armv7-linux-androideabi \
    i686-linux-android \
    x86_64-linux-android

# Install Rust targets for iOS
rustup target add \
    aarch64-apple-ios \
    x86_64-apple-ios \
    aarch64-apple-ios-sim
```

### 1.2 Install Platform-Specific Dependencies
- **Android**: Install Android NDK and set up environment variables
  ```bash
  # Example using cargo-ndk
  cargo install cargo-ndk
  export ANDROID_NDK_HOME=/path/to/android/ndk
  ```

- **iOS**: Install Xcode and iOS SDK (already available if on macOS)

## Phase 2: Configure Tantivy for Cross-Compilation

### 2.1 Create Cross-Compilation Configuration
Create `.cargo/config.toml` in the project root:

```toml
[target.aarch64-linux-android]
linker = "<NDK_PATH>/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android-clang"
ar = "<NDK_PATH>/toolchains/llvm/prebuilt/darwin-x86_64/bin/aarch64-linux-android-ar"

[target.armv7-linux-androideabi]
linker = "<NDK_PATH>/toolchains/llvm/prebuilt/darwin-x86_64/bin/armv7a-linux-androideabi-clang"
ar = "<NDK_PATH>/toolchains/llvm/prebuilt/darwin-x86_64/bin/armv7a-linux-androideabi-ar"

[target.aarch64-apple-ios]
linker = "cc"
rustflags = [
  "-C", "link-arg=-fuse-ld=lld",
  "-C", "link-arg=-mios-version-min=12.0",
]
```

### 2.2 Modify Cargo.toml for Platform-Specific Dependencies
Review `Cargo.toml` to handle platform-specific dependencies:

```toml
[target.'cfg(target_os = "android")'.dependencies]
# Android-specific dependencies

[target.'cfg(target_os = "ios")'.dependencies]
# iOS-specific dependencies
```

### 2.3 Handle Platform-Specific Code
Create conditional compilation blocks for platform-specific code:

```rust
#[cfg(target_os = "android")]
fn platform_specific_function() {
    // Android implementation
}

#[cfg(target_os = "ios")]
fn platform_specific_function() {
    // iOS implementation
}
```

## Phase 3: Build Tantivy Libraries

### 3.1 Build for Android
```bash
# For each target architecture
cargo ndk --target aarch64-linux-android --platform 21 build --release
cargo ndk --target armv7-linux-androideabi --platform 21 build --release
cargo ndk --target i686-linux-android --platform 21 build --release
cargo ndk --target x86_64-linux-android --platform 21 build --release
```

### 3.2 Build for iOS
```bash
# For device
cargo build --target aarch64-apple-ios --release

# For simulator
cargo build --target aarch64-apple-ios-sim --release
```

### 3.3 Create Universal Binary for iOS (if needed)
```bash
lipo -create \
  target/aarch64-apple-ios/release/libtantivy.a \
  target/x86_64-apple-ios/release/libtantivy.a \
  -output target/universal-ios/release/libtantivy.a
```

## Phase 4: Create Test Project

### 4.1 Structure for Test Project
```
tantivy-mobile-tests/
├── android/
│   ├── app/
│   │   ├── src/
│   │   │   └── main/
│   │   │       ├── java/
│   │   │       │   └── com/tantivy/tests/
│   │   │       │       └── MainActivity.kt
│   │   │       ├── jni/
│   │   │       │   └── tantivy_tests.rs
│   │   │       └── AndroidManifest.xml
│   │   └── build.gradle
│   └── build.gradle
├── ios/
│   ├── TantivyTests/
│   │   ├── AppDelegate.swift
│   │   ├── TestRunner.swift
│   │   └── Info.plist
│   └── TantivyTests.xcodeproj
└── rust/
    ├── src/
    │   ├── lib.rs
    │   └── tests/
    │       └── mobile_tests.rs
    └── Cargo.toml
```

### 4.2 Create Rust Test Library
`rust/Cargo.toml`:
```toml
[package]
name = "tantivy-mobile-tests"
version = "0.1.0"
edition = "2021"

[dependencies]
tantivy = { path = "../path/to/tantivy" }

[lib]
crate-type = ["staticlib", "cdylib"]
```

`rust/src/lib.rs`:
```rust
use tantivy;

#[no_mangle]
pub extern "C" fn run_tests() -> i32 {
    // Run tests and return success/failure code
    // ...
}
```

### 4.3 Android Integration

#### Create Kotlin Bindings
```kotlin
// MainActivity.kt
class MainActivity : AppCompatActivity() {
    companion object {
        init {
            System.loadLibrary("tantivy_tests")
        }
    }

    private external fun runTests(): Int

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        val result = runTests()
        val resultView: TextView = findViewById(R.id.test_results)
        resultView.text = "Test result: ${if (result == 0) "PASS" else "FAIL"}"
    }
}
```

#### Configure Gradle for JNI
```gradle
// app/build.gradle
android {
    // ...
    externalNativeBuild {
        cmake {
            path "src/main/jni/CMakeLists.txt"
        }
    }
}
```

### 4.4 iOS Integration

#### Create Swift Bindings
```swift
// TestRunner.swift
import Foundation

class TestRunner {
    func runTests() -> Int32 {
        return run_tests()
    }
}

// C function declaration
@_cdecl("run_tests")
func run_tests() -> Int32
```

## Phase 5: Implement Test Suite

### 5.1 Create Mobile-Specific Tests
```rust
// rust/src/tests/mobile_tests.rs
#[cfg(test)]
mod tests {
    use tantivy::*;
    
    #[test]
    fn test_basic_indexing() {
        // Test basic indexing functionality
    }
    
    #[test]
    fn test_search() {
        // Test search functionality
    }
    
    #[test]
    fn test_file_operations() {
        // Test file operations on mobile platforms
    }
}
```

### 5.2 Create Test Runner
```rust
// rust/src/lib.rs
#[no_mangle]
pub extern "C" fn run_tests() -> i32 {
    let mut test_results = Vec::new();
    
    // Run each test and collect results
    test_results.push(run_test_basic_indexing());
    test_results.push(run_test_search());
    test_results.push(run_test_file_operations());
    
    // Return 0 if all tests passed, non-zero otherwise
    if test_results.iter().all(|&r| r) {
        0
    } else {
        1
    }
}
```

## Phase 6: Continuous Integration Setup

### 6.1 GitHub Actions Workflow
Create `.github/workflows/mobile-tests.yml`:

```yaml
name: Mobile Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  android-tests:
    runs-on: ubuntu-latest
    steps:
      # Setup Android emulator and run tests
      
  ios-tests:
    runs-on: macos-latest
    steps:
      # Setup iOS simulator and run tests
```

## Phase 7: Documentation

### 7.1 Create Documentation
- Setup instructions
- Test execution instructions
- Troubleshooting guide
- Platform-specific considerations

## Timeline Estimate
- Phase 1-2: 1 week
- Phase 3: 1 week
- Phase 4-5: 2 weeks
- Phase 6-7: 1 week

## Potential Challenges
1. **File system differences**: Mobile platforms have different file system permissions and structures
2. **Memory constraints**: Mobile devices have limited memory compared to desktop environments
3. **Platform-specific bugs**: Some functionality may behave differently on mobile platforms
4. **Dependencies**: Some dependencies may not be compatible with mobile platforms
5. **Testing environment**: Setting up automated testing on physical devices
