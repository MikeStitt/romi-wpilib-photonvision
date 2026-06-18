# ADR-001: Base OS Selection for Romi + PhotonVision Images

## Status
Accepted

## Context
We need to select a base OS for building bootable Romi images that run WPILib 2026 + PhotonVision 2026 on Raspberry Pi 4/5 and Orange Pi 5. The image builder will use a photon-image-modifier style script pipeline (modify existing OS image → output bootable image).

## Decision
**Raspberry Pi 4/5: Raspberry Pi OS Trixie (64-bit, Debian 13)**
**Orange Pi 5: Ubuntu 24.04 LTS (Noble) with Rockchip kernel (Joshua Riek's ubuntu-rockchip images)**

**QEMU Development: Ubuntu 24.04 LTS cloud image (works in QEMU virt machine)**

## Rationale

### PhotonVision's Actual Base OS Per Platform (from CI/CD)

| Platform | Base Image | Source |
|----------|------------|--------|
| **Raspberry Pi** | Raspberry Pi OS Trixie (2025-10-01-raspios-trixie-arm64-lite.img.xz) | Official Raspberry Pi OS downloads |
| **Orange Pi 5 / 5B / 5Plus / 5Pro / 5Max / Rock 5C** | Ubuntu 24.04 preinstalled server (Joshua Riek ubuntu-rockchip) | github.com/Joshua-Riek/ubuntu-rockchip |
| **Rubik Pi 3** | Ubuntu 24.04 server (Qualcomm IoT) | Canonical |
| **Limelight / Luma P1 / Snakeyes** | Raspberry Pi OS Trixie (same as RPi) | Official Raspberry Pi OS downloads |

**Key insight**: PhotonVision uses **platform-native OS images**, not a single unified OS:
- RPi → Raspberry Pi OS (Debian-based, Pi-specific bootloader/kernel)
- Rockchip boards → Ubuntu 24.04 with Rockchip kernel/firmware
- Qualcomm boards → Ubuntu 24.04 with Qualcomm kernel/firmware

### Why Not Single Ubuntu 24.04 for RPi?

| Factor | Raspberry Pi OS Trixie | Ubuntu 24.04 on RPi |
|--------|------------------------|---------------------|
| **Bootloader** | Native Pi firmware (bootcode.bin, start4.elf, fixup.dat) | Requires U-Boot + UEFI (not native) |
| **Kernel** | Raspberry Pi kernel (vc4, v3d, bcm2835 drivers) | Generic/mainline kernel (limited VPU/ISP support) |
| **GPU/Video** | Full VC4/V3D + MMAL + libcamera support | Partial support, no hardware video encode |
| **Camera (libcamera)** | Native libcamera RPi pipeline (ISP tuning) | Basic libcamera, no RPi ISP tuning |
| **WPILib Pi support** | Native (WPILibPi historically uses RPi OS) | Untested, may lack Pi-specific HAL |
| **PhotonVision CI** | ✅ Official build target | ❌ Not a build target |
| **QEMU emulation** | ❌ `raspi4b` machine incomplete | ✅ `virt` machine works |

### Driver Support Comparison (RPi 4/5)

| Subsystem | Raspberry Pi OS Trixie | Ubuntu 24.04 (mainline) |
|-----------|------------------------|-------------------------|
| **VC4/V3D GPU** | Full OpenGL ES, Vulkan (experimental) | Mesa VC4/V3D (no VPU/ISP) |
| **Camera (CSI)** | libcamera RPi IPA (ISP tuning, AWB/AE) | libcamera basic (no ISP tuning) |
| **Hardware video encode** | H.264/H.265 via V4L2 (v4l2-request) | Not available |
| **GPIO/I2C/SPI** | Native kernel drivers | Generic gpiolib (works) |
| **PWM/Fan control** | Native (rpi-firmware, rpi-eeprom) | Limited |
| **Bootloader EEPROM updates** | `rpi-eeprom` tool | Manual/unsupported |
| **WiFi/BT (Pi 4/5)** | brcmfmac + firmware (native) | brcmfmac (works) |

**Conclusion**: For camera-heavy workloads (PhotonVision's core use case), **Raspberry Pi OS is superior** due to ISP tuning, hardware video encode, and libcamera RPi pipeline.

## QEMU Development Strategy

Since Raspberry Pi OS Trixie **does not boot reliably in QEMU** (Pi bootloader not fully emulated), we use a **dual-track approach**:

### Track 1: Ubuntu 24.04 Cloud Image (QEMU Development)
- Base: `ubuntu-24.04-server-cloudimg-arm64.img`
- Machine: `virt` + UEFI + cloud-init
- Use for: PhotonVision JAR testing, dashboard development, NT4 bridge, image builder pipeline logic
- **Limitation**: No Pi-specific hardware (camera, GPU, GPIO)

### Track 2: Raspberry Pi OS Trixie (Hardware Validation)
- Base: `2025-10-01-raspios-trixie-arm64-lite.img.xz`
- Test on: Real RPi 4/5 hardware
- Use for: Camera integration, 32U4 flashing, GPIO/I2C, final image validation

### Track 3: Orange Pi 5 Ubuntu (Hardware Validation)
- Base: `ubuntu-24.04-preinstalled-server-arm64-orangepi-5.img.xz` (Joshua Riek)
- Test on: Real Orange Pi 5 hardware
- Use for: Rockchip-specific camera/GPU, final image validation

## Key Technical Findings from Spike (2026-06-17)

1. **PhotonVision 2026.3.4 runs on Java 25** - Despite building with Java 17 (source/target compatibility), the install script pulls `openjdk-25-jre-headless` which is available in Ubuntu 24.04 and Raspberry Pi OS Trixie and works correctly.

2. **Ubuntu 24.04 cloud image boots in QEMU** - Using `virt` machine + UEFI + cloud-init, reaches login in ~60s. PhotonVision install.sh `--test` mode shows all dependencies resolvable.

3. **Raspberry Pi OS Trixie does not boot in QEMU** - The Pi image format requires Pi-specific firmware (boot partition with bootcode.bin, start4.elf) that QEMU's `raspi4b` machine doesn't fully emulate. The `virt` machine can't read the Pi's MBR partition layout.

4. **PhotonVision Orange Pi 5 pre-built image** - Uses GPT + EFI system partition but doesn't boot in QEMU `virt` machine (UEFI doesn't find bootloader). Likely requires U-Boot on real hardware.

5. **PhotonVision install.sh test mode on Ubuntu 24.04** - All packages resolve: `avahi-daemon`, `libatomic1`, `v4l-utils`, `sqlite3`, `openjdk-25-jre-headless`, `usbtop`, `network-manager`, `net-tools`, `netplan.io`.

6. **PhotonVision CI builds RPi images on Raspberry Pi OS Trixie** - The official PhotonVision Raspberry Pi image IS based on Raspberry Pi OS Trixie, not Ubuntu.

## Consequences

### Positive
- **Platform-native OS** = best driver/hardware support for each board
- PhotonVision install.sh works on both Debian (RPi OS) and Ubuntu (Orange Pi) without modification
- QEMU development possible via Ubuntu 24.04 cloud image
- Matches PhotonVision's own CI/CD strategy

### Negative
- **Two different base images** to maintain (RPi OS Trixie + Ubuntu 24.04 Rockchip)
- Build pipeline needs platform-specific stages (boot config, kernel, firmware)
- QEMU cannot fully validate Pi-specific features (camera, GPU, 32U4 I2C)

### Mitigation
- Abstract platform differences in image builder (common stages + platform-specific overlays)
- Hardware CI: test on real RPi 4/5 and Orange Pi 5
- Document Pi-specific packages needed on Ubuntu if we ever unify (not recommended)

## Implementation Notes

### Image Builder Pipeline (Platform-Aware)
```bash
# Platform detection
PLATFORM="${1:-rpi}"  # rpi | opi5

case "$PLATFORM" in
  rpi)
    BASE_IMG="2025-10-01-raspios-trixie-arm64-lite.img.xz"
    BOOT_PARTITION="partition=1"
    ROOT_PARTITION="partition=2"
    KERNEL_PACKAGE="raspberrypi-kernel"
    FIRMWARE_PACKAGES_PACKAGES="raspberrypi-bootloader raspberrypi-firmware rpi-eeprom"
    ;;
  opi5)
    BASE_IMG="ubuntu-24.04-preinstalled-server-arm64-orangepi-5.img.xz"
    BOOT_PARTITION="partition=1"  # EFI system partition
    ROOT_PARTITION="partition=2"
    KERNEL_PACKAGE="linux-image-rockchip"
    FIRMWARE_PACKAGES="linux-firmware rockchip-firmware u-boot-tools"
    ;;
esac

# Common stages (01-base, 03-photonvision, 05-dashboard, 06-integration, 07-finalize)
# Platform-specific stages (02-network, 04-romi-core, 08-platform-specific)
```

### QEMU Test Command (Ubuntu 24.04 for development)
```bash
qemu-system-aarch64 \
  -M virt -cpu cortex-a72 -m 2G -smp 4 \
  -drive file=ubuntu-24.04.qcow2,format=qcow2,if=virtio \
  -netdev user,id=net0,hostfwd=tcp::5800-:5800,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  -display none -serial file:serial.log \
  -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd
```

### Hardware Test Commands
```bash
# Flash RPi image to SD card
sudo dd if=romi-rpi.img of=/dev/sdX bs=4M status=progress

# Flash Orange Pi 5 image
sudo dd if=romi-opi5.img of=/dev/sdX bs=4M status=progress
```

## Validation
- [x] PhotonVision JAR starts on Ubuntu 24.04 arm64 (Java 25)
- [x] install.sh --test mode resolves all dependencies on Ubuntu 24.04
- [x] Ubuntu 24.04 cloud image boots in QEMU virt machine
- [x] NetworkManager + netplan configuration works on Ubuntu
- [x] **PhotonVision CI uses Raspberry Pi OS Trixie for RPi builds**
- [x] **PhotonVision CI uses Ubuntu 24.04 Rockchip for Orange Pi 5 builds**
- [ ] PhotonVision install.sh on Raspberry Pi OS Trixie (needs hardware test)
- [ ] Camera/libcamera pipeline on RPi OS Trixie (hardware test)
- [ ] 32U4 flashing via avrdude on RPi hardware (hardware test)

## References
- PhotonVision CI workflow: https://github.com/PhotonVision/photon-image-modifier/blob/main/.github/workflows/main.yml
- PhotonVision install.sh: https://github.com/PhotonVision/photon-image-modifier/blob/main/install.sh
- PhotonVision 2026.3.4 release: https://github.com/PhotonVision/photonvision/releases/tag/v2026.3.4
- Raspberry Pi OS Trixie: https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2025-10-02/
- Ubuntu Rockchip (Joshua Riek): https://github.com/Joshua-Riek/ubuntu-rockchip
- WPILib 2026.2.1: https://github.com/wpilibsuite/allwpilib/releases/tag/v2026.2.1

## Date
2026-06-18 (updated from 2026-06-17)

## Authors
Spike implementation by agent