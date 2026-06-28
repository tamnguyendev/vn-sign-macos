# VN Sign macOS

Bộ ứng dụng ký số trên macOS dành cho USB Token (PKCS#11).

## Tổng quan

| Thư mục | Mô tả | Công nghệ |
|---------|-------|-----------|
| `usb-token-agent/` | Agent chạy nền (menu bar) — cung cấp API ký số qua HTTP, MQTT, UDP discovery | Swift 5.9, macOS 14+ |
| `sign-app/` | Ứng dụng ký số desktop (giao diện đồ họa) | .NET 8, Avalonia, C# |

**Khi cài đặt `sign-app`**, bộ cài `.pkg` sẽ tự động cài cả `UsbTokenAgent` kèm theo.  
**`usb-token-agent`** có thể cài riêng (standalone) nếu chỉ cần agent.

---

## Yêu cầu hệ thống

- macOS 14 (Sonoma) trở lên
- Apple Silicon (arm64 / M1, M2, M3, M4)
- USB Token có driver PKCS#11 tương thích (bit4id, SafeNet, ePass...)
- Kết nối internet (cho lần cài đặt đầu tiên và notarization)

---

## Cài đặt cho người dùng cuối

### Cách 1: Tải từ GitHub Releases

1. Truy cập [Releases](https://github.com/tamnguyendev/vn-sign-macos/releases)
2. Tải file `.pkg` mới nhất:
   - **VimesSign** (bao gồm cả UsbTokenAgent): `VimesSign-mac-arm64-X.Y.Z.pkg`
   - **UsbTokenAgent riêng** (chỉ agent): `UsbTokenAgent-mac-arm64-X.Y.Z.pkg`
3. Nhấp đúp vào file `.pkg` → Trình cài đặt mở (không có cảnh báo Gatekeeper vì đã được Apple notarize)
4. Làm theo hướng dẫn → Ứng dụng được cài vào `/Applications/`

### Cách 2: Cài từ Terminal

```bash
# Tải về
curl -LO https://github.com/tamnguyendev/vn-sign-macos/releases/latest/download/VimesSign-mac-arm64-1.0.0.pkg

# Cài đặt
sudo installer -pkg VimesSign-mac-arm64-1.0.0.pkg -target /
```

---

## Cấu hình lần đầu

Khi khởi chạy lần đầu, ứng dụng tự động tạo file cấu hình tại:

```
~/.config/vimes-sign/appsettings.json
```

Bạn cần chỉnh sửa file này với thông tin merchant của mình:

```bash
# Mở file cấu hình
open ~/.config/vimes-sign/appsettings.json
```

### Các cấu hình quan trọng

| Mục | Mô tả |
|-----|-------|
| `UsbSetting.UsbAgentIp` | IP của UsbTokenAgent (mặc định: `127.0.0.1`) |
| `UsbSetting.UsbAgentPort` | Port (mặc định: `9999`) |
| `UsbSetting.UsbTokenPin` | Mã PIN USB Token |
| `MySignSetting` | Thông tin kết nối Viettel MySign |
| `SmartCASetting` | Thông tin kết nối VNPT SmartCA |
| `TerminalSetting` | Thông tin MPKI (CA Gov) |

---

## Hướng dẫn sử dụng

### Ký bằng USB Token

1. Cắm USB Token vào máy
2. Mở **VimesSign** từ Applications
3. Chọn Merchant: **USB**
4. Nhập **Mã PIN** của token
5. Nhấn **Đăng nhập** → Chứng thư số sẽ hiện trong dropdown
6. Chọn file PDF cần ký → Nhấn **Ký**

> **Lưu ý:** Khi chọn merchant USB, trường "Tên đăng nhập" sẽ tự động bị vô hiệu hóa (không cần nhập).  
> Nút **Tải Chứng Thư Số** cho phép tải lại danh sách chứng thư từ token mà không cần đăng nhập lại.

### UsbTokenAgent (chạy nền)

- Agent tự khởi động khi VimesSign cần ký USB
- Hiển thị biểu tượng trên thanh menu (không có Dock icon)
- Cung cấp HTTP API tại `localhost:9999`
- Nếu cần khởi động riêng: mở `/Applications/UsbTokenAgent.app`

---

## Build từ mã nguồn (Dành cho lập trình viên)

### usb-token-agent (Swift)

```bash
cd usb-token-agent
swift build -c release
```

Output: `.build/release/UsbTokenAgent`

### sign-app (.NET 8)

```bash
cd sign-app
dotnet restore
dotnet build
```

Chạy thử:
```bash
dotnet run --project sign-app/VimesSignSample.csproj
```

---

## CI/CD — GitHub Actions

### Workflows

| File | Trigger | Mô tả |
|------|---------|--------|
| `build-usb-token-agent.yml` | Tag `usb-token-agent-v*.*.*` | Build + ký + notarize UsbTokenAgent (standalone) |
| `build-sign-app.yml` | Tag `sign-app-v*.*.*` | Build cả VimesSign + UsbTokenAgent → `.pkg` kết hợp |

Cả hai workflow đều chạy **build-only** trên push `main` (không ký, không notarize) để kiểm tra compilation.

### Tạo bản phát hành

```bash
# Phát hành UsbTokenAgent riêng
./scripts/release-usb-token-agent.sh 1.0.0

# Phát hành VimesSign (bao gồm UsbTokenAgent)
./scripts/release-sign-app.sh 1.0.0
```

---

## Cấu trúc dự án

```
vn-sign-macos/
├── .github/workflows/
│   ├── build-usb-token-agent.yml  # CI: UsbTokenAgent standalone
│   └── build-sign-app.yml        # CI: VimesSign + UsbTokenAgent kết hợp
├── usb-token-agent/
│   ├── Package.swift              # Swift package manifest
│   ├── Sources/                   # Mã nguồn Swift
│   ├── CPkcs11/                   # C bridge cho PKCS#11 headers
│   ├── Resources/                 # AppIcon, appsettings, dylibs
│   └── entitlements.plist         # Quyền hardened runtime
├── sign-app/
│   ├── VimesSignSample.csproj     # .NET 8 Avalonia project
│   ├── Program.cs                 # Entry point + DI setup
│   ├── MainWindow.axaml           # Giao diện chính
│   ├── MainWindow.axaml.cs        # Logic xử lý
│   ├── Resources/                 # Icon ứng dụng
│   ├── entitlements.plist         # Quyền cho .NET CoreCLR JIT
│   ├── appsettings.json           # Cấu hình (placeholder)
│   └── appsettings.example.json   # Template cấu hình
├── nuget.config                   # NuGet source (nuget.org)
├── .gitignore
└── README.md
```

---

## Ghi chú kỹ thuật

### Tại sao cần entitlements?

Ứng dụng .NET self-contained trên macOS cần các entitlement sau để chạy với hardened runtime:

- `com.apple.security.cs.allow-jit` — Cho phép CoreCLR JIT compile
- `com.apple.security.cs.allow-unsigned-executable-memory` — Cần cho .NET runtime
- `com.apple.security.cs.disable-library-validation` — Cho phép load dylib bên ngoài

### Dữ liệu người dùng

Tất cả dữ liệu runtime được lưu tại `~/.config/vimes-sign/`:

```
~/.config/vimes-sign/
├── appsettings.json    # Cấu hình người dùng
├── wwwroot/            # Chứng thư số đã tải
│   └── certs/          # File .pfx
└── logs/               # Log ứng dụng
```

### Phiên bản SDK

Ứng dụng sử dụng [Vimes SignSDK](https://www.nuget.org/packages/Vimes.SignSDK/) từ NuGet.org.  
Phiên bản hiện tại: `1.0.23`

---

## Hỗ trợ

- GitHub Issues: [tamnguyendev/vn-sign-macos/issues](https://github.com/tamnguyendev/vn-sign-macos/issues)
- Email: tamnt@vimes.vn
