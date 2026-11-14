# fslop.github.io — FSLOP (`MyVibeProject`)

> **Vibe Coder at FSLOP** — Trung tâm tài liệu, cổng thông tin và điều phối hệ sinh thái hệ điều hành bare-metal trên **Apple Silicon (Mac M1)** thuộc tổ chức [`MyVibeProject`](https://github.com/MyVibeProject).

---

## Giới thiệu

**FSLOP (`MyVibeProject`)** tập trung nghiên cứu, tùy biến và hợp nhất các hệ điều hành hiện đại chạy **bare-metal dual-boot** an toàn cùng **macOS** trên phần cứng **Apple Silicon Mac M1 (`t8103`)**.

Thay vì chạy máy ảo nặng nề làm giảm hiệu năng đồ họa hoặc can thiệp rủi ro vào bảng phân vùng GPT độc quyền của Apple, hệ sinh thái tại **FSLOP** kế thừa tầng phần cứng, bootloader và driver của **Asahi Linux** để xây dựng hai nhánh hệ điều hành chủ lực:

1. **[Chroma-Flex](https://github.com/MyVibeProject/Chroma-Flex)** (`ChromiumOS + Asahi Linux`): Chạy **ChromiumOS ARM64** theo mô hình **Brunch-style Hybrid** (boot qua loop image + custom `initramfs` trên phân vùng Asahi ext4).
2. **[Water-Vapor](https://github.com/MyVibeProject/Water-Vapor)** (`SteamOS ARM + Omarchy + Asahi Linux`): Hệ điều hành **Dual-Mode (Chơi game & Làm việc)** kết hợp **SteamOS 3 Game Mode** (`Gamescope` + `Steam GamepadUI` + `muvm` + `FEX-Emu`) và **Omarchy Desktop Mode** (`Arch Linux ARM` + `Hyprland` + `Quickshell`).

---

## Kiến trúc Hệ sinh thái FSLOP trên Apple Silicon

```text
Apple Silicon Mac M1 (t8103 — 8 GB Unified Memory / 256 GB NVMe SSD)
  └── Apple iBoot -> m1n1 (Stage 1/2 + DeviceTree) -> U-Boot (UEFI) -> GRUB2
        │
        ├── [macOS]         APFS Container nguyên bản (được bảo toàn qua Asahi Installer)
        │
        ├── [Chroma-Flex]   linux-asahi + initramfs-chroma.cpio.gz (Brunch-style Hybrid)
        │                     ├── Mount Asahi ext4 (/host) -> losetup chromiumos.img + chroma-patches.img
        │                     ├── Overlay kernel modules + vendorfw + khóa bảo vệ loa ngoài (chroma-speaker-lock)
        │                     └── switch_root -> ChromiumOS ARM64 (Upstart + Ash/Chrome)
        │
        └── [Water-Vapor]   linux-asahi (Asahi-ALARM Arch Linux ARM64, 16KB page size)
                              ├── Hạ tầng chung : PipeWire + asahi-audio + speakersafetyd + Mesa Honeykrisp (Vulkan)
                              ├── [Game Mode]    Gamescope (DRM/KMS) -> Steam GamepadUI (-gamepadui -steamos3)
                              │                    └── muvm (4KB KVM microVM + Venus) -> FEX-Emu / Proton
                              └── [Desktop Mode] Omarchy (Hyprland + Quickshell + Walker + Neovim + Ghostty)
```

---

## Các Dự án Chủ lực (Flagship Projects)

### 1. [Chroma-Flex](https://github.com/MyVibeProject/Chroma-Flex) — *ChromiumOS + Asahi Linux*

- **Mục tiêu**: Đưa trải nghiệm **ChromiumOS / FydeOS ARM64** lên Mac M1 mà không cần biên dịch toàn bộ cây nguồn `cros_sdk` (>250 GB ổ cứng) và không ghi đè layout 12 phân vùng GPT của ChromiumOS lên ổ NVMe nội bộ.
- **Giải pháp kỹ thuật chính**:
  - **Brunch-style Boot (`initramfs-chroma.cpio.gz`)**: Mount phân vùng `ext4` của Asahi Linux, ánh xạ `chromiumos.img` qua `losetup -P`, ghép lớp `overlayfs` từ `chroma-patches.img` + `/lib/modules` + `/lib/firmware/vendor` rồi `switch_root` sang ChromiumOS.
  - **Xử lý Page Size (4KB vs 16KB)**: Hỗ trợ kiểm tra ELF alignment (`01-check-elf-pagesize.sh`) với 2 lộ trình: **Track A** (kernel `linux-asahi` 4KB page size khởi động trực tiếp userspace 4KB) và **Track B** (kernel 16KB chuẩn kết hợp `patchelf --page-size 16384` + GPU AGX đầy đủ).
  - **An toàn phần cứng**: Tự động khóa/mute mạch khuếch đại loa trong `TAS2770` (`chroma-speaker-lock`) khi chưa có daemon `speakersafetyd` hoạt động, chỉ cho phép xuất âm thanh qua tai nghe 3.5mm / USB / Bluetooth.

### 2. [Water-Vapor](https://github.com/MyVibeProject/Water-Vapor) — *SteamOS ARM + Omarchy + Asahi Linux*

- **Mục tiêu**: Hợp nhất môi trường chơi game console **SteamOS 3 (Holo)** và môi trường lập trình viên hiện đại **Omarchy** trên cùng một nền tảng **Arch Linux ARM (`Asahi-ALARM`)** chạy bare-metal trên Mac M1.
- **Giải pháp kỹ thuật chính**:
  - **Dual-Mode Session Switcher (`greetd` / `uwsm` + `steamos-session-select`)**: Chuyển đổi tức thời giữa **Game Mode** (`Gamescope` DRM/KMS trực tiếp) và **Desktop Mode** (`Hyprland` Wayland compositor) mà không cần khởi động lại máy.
  - **Vượt rào cản 16KB Page Size & x86_64 Gaming**: Sử dụng **`muvm`** (`libkrun` 4KB KVM microVM) kết hợp Vulkan **`Venus`** virtio-gpu passthrough xuống driver **`Mesa Honeykrisp`** ở host 16KB, bên trong chạy **`FEX-Emu`** + **Steam / Proton** (cùng lộ trình **Native ARM64 / Proton-ARM64EC**).
  - **Tối ưu bộ nhớ 8 GB Unified Memory**: Kết hợp `zram` (`zstd`), giới hạn VRAM/guest RAM động cho `muvm`, và giải phóng hoàn toàn tiến trình Desktop khi ở Game Mode.

---

## Danh mục Repository trong Hệ sinh thái `MyVibeProject`

Mỗi thành phần upstream từ `AsahiLinux`, `omacom` và `fxgl` được quản lý theo cặp: một **Public Fork** (đồng bộ upstream) và một **Private Renamed Repo** (tùy biến nội bộ cho hệ sinh thái FSLOP).

### A. Cổng thông tin & Dự án Chủ lực

| Repository | Trạng thái | Mô tả |
| --- | --- | --- |
| [`MyVibeProject/fslop.github.io`](https://github.com/MyVibeProject/fslop.github.io) | Public | Trang chủ & tài liệu trung tâm của **FSLOP** (*Vibe Coder at FSLOP*) |
| [`MyVibeProject/Chroma-Flex`](https://github.com/MyVibeProject/Chroma-Flex) | Private | Hệ thống **Brunch-style Hybrid** kết hợp **ChromiumOS ARM64 + Asahi Linux** trên Mac M1 |
| [`MyVibeProject/Water-Vapor`](https://github.com/MyVibeProject/Water-Vapor) | Private | Hệ điều hành Dual-Mode **SteamOS ARM + Omarchy + Asahi Linux** trên Mac M1 |

### B. Nền tảng Desktop, Ứng dụng & Gaming (`Water-Vapor` Stack)

| Private Renamed Repo | Public Fork (`MyVibeProject`) | Vai trò |
| --- | --- | --- |
| [`water-vapor-desktop`](https://github.com/MyVibeProject/water-vapor-desktop) | [`omarchy`](https://github.com/MyVibeProject/omarchy) | Môi trường Desktop Mode (Omarchy Arch Linux + Hyprland tùy biến cho Apple Silicon) |
| [`water-vapor-mac`](https://github.com/MyVibeProject/water-vapor-mac) | [`omarchy-mac`](https://github.com/MyVibeProject/omarchy-mac) | Cấu hình phần cứng, phím tắt, HiDPI và tích hợp hệ thống Apple Silicon |
| [`water-vapor-mac-pkgs`](https://github.com/MyVibeProject/water-vapor-mac-pkgs) | [`omarchy-mac-pkgs`](https://github.com/MyVibeProject/omarchy-mac-pkgs) | Kho gói `aarch64` chuyên biệt cho Apple Silicon Mac |
| [`water-vapor-mac-installer`](https://github.com/MyVibeProject/water-vapor-mac-installer) | [`omarchy-mac-installer`](https://github.com/MyVibeProject/omarchy-mac-installer) | Bộ cài đặt tích hợp từ phía macOS cho dòng máy Apple Silicon |
| [`water-vapor-pkgs`](https://github.com/MyVibeProject/water-vapor-pkgs) | [`omarchy-pkgs`](https://github.com/MyVibeProject/omarchy-pkgs) | Tập hợp các gói Arch Linux lõi phục vụ môi trường Desktop & Gaming |
| [`try-water-vapor`](https://github.com/MyVibeProject/try-water-vapor) | [`try-omarchy`](https://github.com/MyVibeProject/try-omarchy) | Công cụ chạy thử nhanh môi trường Water-Vapor Desktop ngay trên macOS |
| [`water-vapor-homebrew`](https://github.com/MyVibeProject/water-vapor-homebrew) | [`homebrew-mac`](https://github.com/MyVibeProject/homebrew-mac) | Homebrew tap phân phối bộ công cụ hỗ trợ trên macOS |
| [`water-vapor-quickshell`](https://github.com/MyVibeProject/water-vapor-quickshell) | [`quickshell`](https://github.com/MyVibeProject/quickshell) | Toolkit QtQuick xây dựng thanh trạng thái & shell Wayland cho Desktop Mode |
| [`water-vapor-hyprland-titlebars`](https://github.com/MyVibeProject/water-vapor-hyprland-titlebars) | [`omarchy-hyprland-titlebars`](https://github.com/MyVibeProject/omarchy-hyprland-titlebars) | Plugin titlebar cho các workspace floating trên Hyprland |
| [`water-vapor-audio-tuner`](https://github.com/MyVibeProject/water-vapor-audio-tuner) | [`omarchy-audio-tuner`](https://github.com/MyVibeProject/omarchy-audio-tuner) | Công cụ tinh chỉnh profile DSP âm thanh loa laptop |
| [`water-vapor-aether`](https://github.com/MyVibeProject/water-vapor-aether) | [`aether`](https://github.com/MyVibeProject/aether) | Công cụ tùy biến theme đồng bộ giao diện Desktop |
| [`water-vapor-folio`](https://github.com/MyVibeProject/water-vapor-folio) | [`folio`](https://github.com/MyVibeProject/folio) | Trình quản lý tệp tối giản ưu tiên bàn phím cho Desktop Mode |
| [`water-vapor-snap`](https://github.com/MyVibeProject/water-vapor-snap) | [`omasnap`](https://github.com/MyVibeProject/omasnap) | Trình chụp ảnh màn hình & chú thích native Wayland cho Hyprland |
| [`water-vapor-reel`](https://github.com/MyVibeProject/water-vapor-reel) | [`omareel`](https://github.com/MyVibeProject/omareel) | Trình quay và chỉnh sửa video màn hình cho Desktop Mode |
| [`water-vapor-screensaver`](https://github.com/MyVibeProject/water-vapor-screensaver) | [`omarchy-screensaver`](https://github.com/MyVibeProject/omarchy-screensaver) | Màn hình chờ native Wayland cho Desktop Mode |
| [`water-vapor-ttfx`](https://github.com/MyVibeProject/water-vapor-ttfx) | [`ttfx`](https://github.com/MyVibeProject/ttfx) | Hiệu ứng chữ terminal Rust tĩnh cho trình bảo vệ màn hình |
| [`water-vapor-plain`](https://github.com/MyVibeProject/water-vapor-plain) | [`plain`](https://github.com/MyVibeProject/plain) | Trình soạn thảo văn bản tối giản cho file cấu hình |
| [`water-vapor-local-ai`](https://github.com/MyVibeProject/water-vapor-local-ai) | [`omarchy-local-ai`](https://github.com/MyVibeProject/omarchy-local-ai) | Danh mục & thiết lập mô hình AI chạy cục bộ |
| [`water-vapor-steamac`](https://github.com/MyVibeProject/water-vapor-steamac) | [`steamac`](https://github.com/MyVibeProject/steamac) | Kiến trúc tham chiếu SteamOS ARM64 trên Apple Silicon (`libkrun` + `Venus` + Gamepad/Audio) |
| [`water-vapor-muvm`](https://github.com/MyVibeProject/water-vapor-muvm) | [`muvm`](https://github.com/MyVibeProject/muvm) | MicroVM KVM 4KB (`muvm`) chạy ứng dụng & game x86_64/Steam trên host kernel 16KB |
| [`water-vapor-fex-rootfs`](https://github.com/MyVibeProject/water-vapor-fex-rootfs) | [`fex-emu-rootfs-generator`](https://github.com/MyVibeProject/fex-emu-rootfs-generator) | `systemd` generator tự động mount FEX-Emu x86_64 RootFS bên trong `muvm` |
| [`water-vapor-binfmt`](https://github.com/MyVibeProject/water-vapor-binfmt) | [`binfmt-dispatcher`](https://github.com/MyVibeProject/binfmt-dispatcher) | Cấu hình `binfmt_misc` chuyển hướng thực thi binary x86/x86_64 sang FEX-Emu |

### C. Hạ tầng Phần cứng, Bootloader, Video & Audio (`Asahi Linux` & `omacom` Stack dùng chung)

| Private Renamed Repo | Public Fork (`MyVibeProject`) | Vai trò đối với `Chroma-Flex` & `Water-Vapor` |
| --- | --- | --- |
| [`water-vapor-m1n1`](https://github.com/MyVibeProject/water-vapor-m1n1) | [`m1n1`](https://github.com/MyVibeProject/m1n1) | Bootloader Apple Silicon (`m1n1`) khởi tạo phần cứng và nạp DeviceTree |
| [`asahi-u-boot`](https://github.com/MyVibeProject/asahi-u-boot) | [`u-boot`](https://github.com/MyVibeProject/u-boot) | U-Boot cung cấp giao diện UEFI runtime (stage 2 payload sau `m1n1`) |
| [`fslop-asahi-installer`](https://github.com/MyVibeProject/fslop-asahi-installer) | [`asahi-installer`](https://github.com/MyVibeProject/asahi-installer) | Trình cài đặt thu gọn APFS an toàn trên macOS & trích xuất `vendorfw` |
| [`fslop-asahi-installer-data`](https://github.com/MyVibeProject/fslop-asahi-installer-data) | [`asahi-installer-data`](https://github.com/MyVibeProject/asahi-installer-data) | Danh mục OS (`installer_data.json`) phục vụ cài đặt qua `alx.sh` |
| [`fslop-asahi-scripts`](https://github.com/MyVibeProject/fslop-asahi-scripts) | [`asahi-scripts`](https://github.com/MyVibeProject/asahi-scripts) | Bộ script bảo trì hệ thống (`update-m1n1`, `asahi-fwextract`, initramfs hooks) |
| [`fslop-asahi-plymouth`](https://github.com/MyVibeProject/fslop-asahi-plymouth) | [`asahi-plymouth`](https://github.com/MyVibeProject/asahi-plymouth) | Giao diện Plymouth boot splash cho hệ thống chạy nhân Asahi Linux |
| [`water-vapor-libva`](https://github.com/MyVibeProject/water-vapor-libva) | [`libva-v4l2_request`](https://github.com/MyVibeProject/libva-v4l2_request) | Driver VA-API tăng tốc giải mã video phần cứng qua Apple Video Decoder (AVD) |
| [`asahi-avd-fw`](https://github.com/MyVibeProject/asahi-avd-fw) | [`avd-fw`](https://github.com/MyVibeProject/avd-fw) | Firmware mã nguồn mở thay thế cho Apple Video Decoder (AVD) trên Apple Silicon |
| [`asahi-speakersafetyd`](https://github.com/MyVibeProject/asahi-speakersafetyd) | [`speakersafetyd`](https://github.com/MyVibeProject/speakersafetyd) | Daemon giám sát nhiệt độ cuộn âm bảo vệ loa ngoài Mac M1 (`TAS2770`) |
| [`fslop-asahi-audio`](https://github.com/MyVibeProject/fslop-asahi-audio) | [`asahi-audio`](https://github.com/MyVibeProject/asahi-audio) | Cấu hình DSP âm thanh (PipeWire / Bankstown) cho loa trên Apple Silicon |
| [`fslop-alsa-ucm-asahi`](https://github.com/MyVibeProject/fslop-alsa-ucm-asahi) | [`alsa-ucm-conf-asahi`](https://github.com/MyVibeProject/alsa-ucm-conf-asahi) | ALSA Use Case Manager (UCM) profile dùng cho cả PipeWire và ChromiumOS `CRAS` |
| [`fslop-asahi-nvram`](https://github.com/MyVibeProject/fslop-asahi-nvram) | [`asahi-nvram`](https://github.com/MyVibeProject/asahi-nvram) | Công cụ đọc/ghi biến NVRAM Apple Silicon từ môi trường Linux |
| [`fslop-asahi-fix27`](https://github.com/MyVibeProject/fslop-asahi-fix27) | [`asahi-fix27`](https://github.com/MyVibeProject/asahi-fix27) | Tiện ích khắc phục cờ khởi động macOS 27 từ phía Linux |
| [`asahi-tiny-dfr`](https://github.com/MyVibeProject/asahi-tiny-dfr) | [`tiny-dfr`](https://github.com/MyVibeProject/tiny-dfr) | Daemon điều khiển dải cảm ứng Touch Bar (`tiny-dfr`) trên MacBook Pro M1 |
| [`asahi-widevine-installer`](https://github.com/MyVibeProject/asahi-widevine-installer) | [`widevine-installer`](https://github.com/MyVibeProject/widevine-installer) | Trình cài đặt Widevine CDM `aarch64` hỗ trợ phát nội dung số có DRM |

---

## Cấu trúc Workspace Phát triển (`Fslop.code-workspace`)

Không gian làm việc cục bộ kết nối trực tiếp 3 repository lõi:

```text
Desktop/
├── Fslop.code-workspace      # VS Code / Antigravity Multi-root Workspace
├── fslop.github.io/          # Cổng thông tin & tài liệu tổng quan FSLOP (repo này)
│   └── README.md
├── Chroma-Flex/              # Dự án ChromiumOS + Asahi Linux (Brunch-style Hybrid)
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── README.md
│   ├── docs/                 # ARCHITECTURE.md & ROADMAP.md
│   ├── initramfs/            # PID 1 init script cho loop-boot ChromiumOS
│   ├── overlay/              # Upstart job & khóa bảo vệ loa ngoài M1
│   └── scripts/              # Công cụ kiểm tra ELF page size, đóng gói initramfs & GRUB
└── Water-Vapor/              # Dự án SteamOS ARM + Omarchy + Asahi Linux (Dual-Mode OS)
    ├── README.md
    └── docs/                 # ARCHITECTURE.md & ROADMAP.md
```

---

## Nguyên tắc An toàn Phần cứng trên Apple Silicon Mac M1

1. **Bảo vệ tuyệt đối phân vùng hệ thống Apple NVMe**: Không bao giờ dùng `dd` hay ghi đè bảng phân vùng GPT trực tiếp lên các phân vùng `Apple_APFS_ISC`, `Apple_APFS` (macOS) hoặc `Apple_APFS_Recovery`. Mọi thao tác chia phân vùng phải thực hiện qua bộ cài đặt chính thức của Asahi Linux (`curl https://alx.sh | sh`).
2. **Bảo vệ loa vật lý (`TAS2770`)**: Tuyệt đối không mở khóa các control ALSA của loa trong trên Mac M1 nếu chưa có **`speakersafetyd`** đang chạy ở chế độ nền. Khi thử nghiệm môi trường mới (như ChromiumOS giai đoạn đầu), bắt buộc giữ cơ chế khóa `chroma-speaker-lock` và sử dụng tai nghe 3.5mm, màn hình HDMI/DisplayPort hoặc tai nghe USB/Bluetooth.
3. **Quản lý Firmware (`vendorfw`)**: Luôn sử dụng gói firmware Wi-Fi, Bluetooth, DCP và GPU (`all_firmware.tar.gz`) được `asahi-installer` trích xuất từ chính phân vùng macOS/Stub trên máy đích để đảm bảo tương thích tuyệt đối với phiên bản phần cứng.

