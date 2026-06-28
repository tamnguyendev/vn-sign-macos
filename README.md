# VN Sign macOS

macOS native applications for digital signing with USB tokens.

## Projects

| Folder | Description | Tech |
|--------|-------------|------|
| `usb-token-agent/` | Menu bar agent — PKCS#11 signing via HTTP API, MQTT, UDP discovery | Swift 5.9, macOS 14+ |
| `sign-app/` | Desktop signing UI (Avalonia) | .NET 8, C# |

---

## Build Locally

### usb-token-agent (Swift)

```bash
cd usb-token-agent
swift build -c release
```

Output binary: `.build/release/UsbTokenAgent`

### sign-app (.NET)

```bash
cd sign-app
dotnet build
```

---

## GitHub Actions — Build & Sign macOS PKG

The workflow `.github/workflows/build-pkg.yml` automates:

1. Build the Swift binary (Release, arm64)
2. Package as `.app` bundle with Info.plist
3. Code-sign with Developer ID Application certificate
4. Wrap in a `.pkg` installer, signed with Developer ID Installer
5. Notarize with Apple (no Gatekeeper warnings)
6. Create GitHub Release with the `.pkg` attached

### Trigger

- **Automatic**: Push a git tag matching `v*.*.*` (e.g. `v1.0.0`)
- **Manual**: Run from GitHub Actions UI → "Run workflow" → enter version

### Required GitHub Secrets

Set these in **Settings → Secrets and variables → Actions**:

| Secret | Description | How to get |
|--------|-------------|------------|
| `DEV_ID_APP_P12` | Base64-encoded Developer ID Application `.p12` | See below |
| `DEV_ID_APP_PASSWORD` | Password for the Application `.p12` | Set when exporting |
| `DEV_ID_INSTALLER_P12` | Base64-encoded Developer ID Installer `.p12` | See below |
| `DEV_ID_INSTALLER_PASSWORD` | Password for the Installer `.p12` | Set when exporting |
| `DEV_ID_APP_NAME` | Signing identity string | `"Developer ID Application: Your Name (TEAMID)"` |
| `DEV_ID_INSTALLER_NAME` | Installer signing identity string | `"Developer ID Installer: Your Name (TEAMID)"` |
| `KEYCHAIN_PASSWORD` | Any random string (for temp CI keychain) | Generate any passphrase |
| `APPLE_ID` | Apple ID email (for notarization) | Your Apple Developer account |
| `APPLE_ID_PASSWORD` | App-specific password | appleid.apple.com → Security → App Passwords |
| `APPLE_TEAM_ID` | 10-char Team ID | developer.apple.com → Membership |

### How to export certificates as Base64

```bash
# 1. Export from Keychain Access as .p12 (set a strong password)

# 2. Encode to Base64 for GitHub Secrets:
base64 -i DeveloperIdApplication.p12 | pbcopy
# Paste into GitHub secret DEV_ID_APP_P12

base64 -i DeveloperIdInstaller.p12 | pbcopy
# Paste into GitHub secret DEV_ID_INSTALLER_P12
```

### How to find your signing identity names

```bash
security find-identity -v -p codesigning
# Example output:
# "Developer ID Application: Tam Nguyen (ABC123XYZ)"

security find-identity -v -p basic
# Look for "Developer ID Installer: Tam Nguyen (ABC123XYZ)"
```

---

## Create a Release

```bash
# Tag and push
git tag v1.0.0
git push origin v1.0.0
```

GitHub Actions will automatically build, sign, notarize, and publish the `.pkg`.

---

## Install the PKG (End Users)

1. Download `UsbTokenAgent-mac-arm64-X.Y.Z.pkg` from [Releases](../../releases)
2. Double-click → Installer opens (no Gatekeeper warning since it's notarized)
3. Follow prompts — installs to `/Applications/UsbTokenAgent.app`
4. Launch from Applications — appears as menu bar icon (no Dock icon)

### Requirements

- macOS 14 (Sonoma) or later
- Apple Silicon (arm64)
- USB Token with PKCS#11 compatible driver

---

## Project Structure

```
vn-sign-macos/
├── .github/workflows/
│   └── build-pkg.yml          # CI: build + sign + notarize + release
├── usb-token-agent/
│   ├── Package.swift          # Swift package manifest
│   ├── Sources/               # Swift source files
│   ├── CPkcs11/               # C bridge for PKCS#11 headers
│   ├── Resources/             # AppIcon, appsettings, dylibs
│   ├── entitlements.plist     # Hardened runtime entitlements
│   └── dist/                  # (gitignored) local builds
├── sign-app/                  # .NET Avalonia desktop UI
├── .gitignore
└── README.md
```
