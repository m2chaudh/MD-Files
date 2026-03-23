# Development Tools Report & Migration Guide (March 23, 2026)

This report provides a snapshot of the current development environment and a version-specific guide for replicating this setup on a new macOS machine.

---

## 1. Core Infrastructure & Package Managers
| Tool | Version | Purpose |
| :--- | :--- | :--- |
| **Homebrew** | 5.1.0 | Primary package manager for macOS. |
| **npm** | 11.12.0 | Node.js package manager (global tools). |
| **Git** | 2.50.1 | Source control (with `git-lfs` support). |
| **Node.js** | 25.8.1 | JavaScript runtime for backend/tools. |
| **Python** | 3.13.2 | Scripting and general development. |
| **Ruby** | 3.4.1 | Required for CocoaPods and iOS dependency management. |

## 2. Mobile & App Development (Flutter)
| Tool | Version | Status |
| :--- | :--- | :--- |
| **Flutter SDK** | 3.41.2 | Stable channel; fully configured. |
| **Dart SDK** | 3.11.0 | Included with Flutter. |
| **Xcode** | 26.3 | Required for iOS/macOS builds. |
| **Android Studio** | Latest (Cask) | IDE for Android development. |
| **CocoaPods** | 1.16.2 | iOS dependency manager. |
| **Firebase CLI** | 15.11.0 | Firebase project management. |

## 3. Game Development & IDEs
| Tool | Category | Status |
| :--- | :--- | :--- |
| **Unity Hub** | Game Engine | Installed via Homebrew Cask. |
| **JetBrains Rider** | IDE | Preferred for C# and Unity. |
| **.NET SDK** | Runtime | Required for Rider/Unity development. |
| **Docker Desktop** | Virtualization | Containerized development. |

## 4. AI & Specialized Utilities
| Tool | Purpose |
| :--- | :--- |
| **gemini-cli** | Interactive CLI agent. |
| **claude-code** | Claude CLI for codebase interactions. |
| **codex** | Specialized AI development tool. |
| **gh (GitHub CLI)** | GitHub repository management from CLI. |
| **ripgrep (rg)** | Ultra-fast search utility. |

---

## Migration Script (New macOS Setup)

### Step 1: Bootstrap Homebrew
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Step 2: Install Formulae with Version Pins
```bash
# Language Runtimes (Matches current running versions)
brew install node@25 python@3.13 ruby@3.4

# Link them to your PATH
brew link --overwrite node@25 python@3.13 ruby@3.4

# Development Utilities
brew install git git-lfs ripgrep sqlite gh cocoapods
```

### Step 3: Install Specific Global npm Tools
```bash
# Firebase Tools pinned to current version
npm install -g firebase-tools@15.11.0
```

### Step 4: Install Flutter 3.41.2 (Exact Version)
```bash
# Using FVM (Flutter Version Manager) to pin the version
brew install fvm
fvm install 3.41.2
fvm global 3.41.2
```

### Step 5: Install GUI Applications (Casks)
```bash
brew install --cask android-studio unity-hub rider dotnet-sdk docker-desktop gcloud-cli claude-code codex gemini-cli
```

### Step 6: Finalization
```bash
# Accept Xcode licenses
sudo xcodebuild -license accept

# Verify environment
node -v && python3 --version && ruby -v && flutter --version && firebase --version
```
