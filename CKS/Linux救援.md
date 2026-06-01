是把硬碟掛載到live usb然後chroot嗎? 哪些目錄至關重要呢?

下午4:02





2 / 2

## Claude responded: 是的，完全正確！流程核心就是：

是的，完全正確！流程核心就是：

**Live USB 開機 → 掛載硬碟上的系統 → chroot 進去 → 操作就像在原系統內一樣**

------

### chroot 前必須掛載的目錄

#### 絕對必要（缺少會出錯）

| 目錄    | 來源           | 用途                                        |
| ------- | -------------- | ------------------------------------------- |
| `/`     | 硬碟分割區     | 根目錄，一切的基礎                          |
| `/dev`  | `--bind /dev`  | 裝置檔案，passwd 等指令需要存取 `/dev/null` |
| `/proc` | `--bind /proc` | 核心行程資訊，很多工具會讀取                |
| `/sys`  | `--bind /sys`  | 硬體與核心介面，grub-install 需要           |

bash

```bash
mount /dev/sda2 /mnt/sysroot
mount --bind /dev  /mnt/sysroot/dev
mount --bind /proc /mnt/sysroot/proc
mount --bind /sys  /mnt/sysroot/sys
```

#### 視情況需要

| 目錄        | 何時需要                                        |
| ----------- | ----------------------------------------------- |
| `/boot`     | 獨立 boot 分割區時（`lsblk` 看到有單獨的 boot） |
| `/boot/efi` | UEFI 系統執行 grub-install 時                   |
| `/run`      | systemd 相關操作，有時需要                      |

------

### 為什麼這四個缺一不可？

```
chroot 進去的程式以為自己在真實系統內
但它們還是需要跟 Linux kernel 溝通

/dev  ← 程式要讀寫裝置（包含 /dev/null、/dev/random）
/proc ← 程式要查詢 PID、系統狀態
/sys  ← 程式要存取硬體資訊
```

少掛 `/dev` → `passwd` 執行失敗
 少掛 `/proc` → `grub-install` 報錯
 少掛 `/sys` → 無法偵測 EFI 系統

------

### 完整的標準流程

bash

```bash
# 1. 確認分割區
lsblk -f

# 2. 掛載根目錄
mount /dev/sda2 /mnt/sysroot

# 3. 綁定虛擬檔案系統
mount --bind /dev  /mnt/sysroot/dev
mount --bind /proc /mnt/sysroot/proc
mount --bind /sys  /mnt/sysroot/sys

# 4. 進入 chroot
chroot /mnt/sysroot /bin/bash

# === 現在你就在原本系統內了 ===
passwd root
grub-install /dev/sda
update-grub

# 5. 離開並清理（順序很重要）
exit
umount /mnt/sysroot/{sys,proc,dev}
umount /mnt/sysroot
reboot
```

一個直覺理解的比喻：**chroot 像是借殼，但殼內的程式還是需要透過 `/dev`、`/proc`、`/sys` 這三條管道和真正的 kernel 通訊**，所以要把 Live 系統的這三個目錄「橋接」進去。