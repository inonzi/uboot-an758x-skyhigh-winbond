# U-Boot for AN758x (Skyhigh/Winbond 8-bit ECC 临时迁移固件)

> **这不是长期使用的正式固件。** 它是一次性的「8-bit 读 / 4-bit 写」救砖镜像：读出原厂写入的 UBI，重建后写回 4-bit，随后必须刷回官方 4-bit preloader 与 FIP。

## 修改说明

本分支在基准分支（仅支持 Winbond W29N02KVSIAF）的基础上，增加了对 **烽火 HG5585F / HG5382A** 上 SkyHigh（原 Spansion/Cypress）S34ML02 系列颗粒的 8-bit ECC 临时迁移支持。

- **核心改动**：在 `drivers/mtd/nand/raw/airoha_en7581_nand.c` 与 TF-A 的 `parallel_nand_flash_table.c` 中加入 3 组 5 字节 ID 匹配：
  | 颗粒                      | 5 字节 ID          | ext_id     |
  | ----------------------- | ---------------- | ---------- |
  | S34ML02G300TFI00        | `01 DA 00 95 46` | `0x469500` |
  | S34ML02G2（部分工具显示为 H200） | `01 DA 90 95 46` | `0x469590` |
  | S34ML02G2 反序 ID         | `01 DA 46 95 90` | `0x909546` |
  命中后把控制器 ECC 强度设为 **8-bit/512B** 以读出原厂 UBI；网页「重建 UBI」时再切回 **4-bit/512B** 写入。64B OOB 的 G1（`01 DA 90 95 44`）不在支持列表内。
- **网页提示**：更新了 Web 恢复界面的中英文提示文案。
- **型号说明**：`S34ML02H200TFI000` 不是 SkyHigh 官方料号（官方为 `S34ML02G200TFI000`），仅见于编程器 / 私有器件库的显示名；源码表中登记的是 `S34ML02G2`。两者对应同一组 ID。

## 🛠️ 构建迁移固件（极其重要）

**本仓库默认编译出的是正式 4-bit 栈，不是救砖用的迁移版。** 三个 `defconfig` 已不再带 `CONFIG_AIROHA_NAND_ECC_BRIDGE=y`，所以常规 `make xxx_defconfig && make` 得到的固件不会触发 8-bit 读取。

迁移版**只能通过构建脚本并带 `BRIDGE_ECC=1` 生成**：它会同时打开 U-Boot 侧 `--enable AIROHA_NAND_ECC_BRIDGE` 和 TF-A 侧 `-DAIROHA_NAND_ECC_BRIDGE`。两侧必须同步，只改其中一侧会得到 BL2 与 U-Boot 不一致的混合固件。

### 必需环境变量（缺任意一个脚本会直接退出）

| 变量                    | 说明                      | CI 同款取值                  |
| --------------------- | ----------------------- | ------------------------ |
| `CROSS_COMPILE`       | AArch64 工具链前缀           | `aarch64-linux-gnu-`     |
| `ARM32_CROSS_COMPILE` | BL2 用的 Arm 裸机工具链前缀      | `arm-none-eabi-`         |
| `MBEDTLS_DIR`         | Mbed TLS **3.4.1** 源码目录 | `/path/to/mbedtls-3.4.1` |

`BRIDGE_TBB=0` 会关闭镜像认证，因此**不需要签名密钥**（`sign-an758x-fip.sh` 检测不到密钥时自动跳过签名）。不带它构建的镜像会要求签名链，可能起不来，所以迁移版固定带上。

### 命令

```sh
export CROSS_COMPILE=aarch64-linux-gnu-
export ARM32_CROSS_COMPILE=arm-none-eabi-
export MBEDTLS_DIR=/path/to/mbedtls-3.4.1

BRIDGE_ECC=1 BRIDGE_TBB=0 ./scripts/build-an758x.sh hg5585f-cu
```

机型参数用**短名**，不是 defconfig 文件名：

| 机型                | 脚本参数         | 对应 defconfig                            |
| ----------------- | ------------ | --------------------------------------- |
| 烽火 HG5585F-CU（联通） | `hg5585f-cu` | `an7581_fiberhome_hg5585f-cu_defconfig` |
| 烽火 HG5585F-CT（电信） | `hg5585f-ct` | `an7581_fiberhome_hg5585f-ct_defconfig` |
| 烽火 HG5382A        | `hg5382a`    | `an7581_fiberhome_hg5382a_defconfig`    |

> ⚠️ 写成 `an7581_fiberhome_hg5585f-cu` 会直接打印 usage 并退出——脚本的 `case` 只匹配短名。

产物在 `output/<机型>-tmp-8bit-read-4bit-write/`，例如 `output/hg5585f-cu-tmp-8bit-read-4bit-write/`。CI 的 `build.yml` 已固定 `BRIDGE_ECC=1`，只有本地手动构建才需要注意这一步。

## ⚠️ 刷机指南（请勿跳过）

1. **第一次刷机必须重建 UBI**：进入 U-Boot 网页后点「重建 UBI」，会清空整个 UBI 分区（含 fip 卷）。
2. **按顺序重刷**：重建后依次刷入 ① 官方 preloader → ② 官方 `bl31-u-boot.fip` → ③ 板级数据 / 系统镜像，然后才能重启。
3. **串口诊断不是万能后门**：`nandecc 4` 是 U-Boot 命令行指令，只有 U-Boot 已经跑起来才能敲。若卡在 **BL2 读 FIP** 阶段，根本进不了命令行，`nandecc 4` 无效；此时只能接 TTL 串口重刷。真正救砖靠的是第 1、2 步。

## 上机前必须验证的假设

bridge 的前提是「原厂按 **8-bit BCH** 写整片」。但 SkyHigh 官方规格里第 5 字节 `0x46` 表示 **4-bit/512B**（002-00499 Table 16），所以这是对本机原厂写法的**假设**，不是厂商规格。

刷入后先看串口识别行是否为 `ECC8/512`、UBI 能否挂载。UBI 层有 ECC8→ECC4 自动回退，**BL2 读 FIP 没有回退**：假设错了会卡在 BL2，只能 TTL 重刷。

## 免责声明 (Disclaimer)

- 本仓库的修改由 AI 辅助完成，维护者并非底层驱动开发专家，刷机前请务必备份重要数据。
- 若发生不可逆变砖或硬件损坏，需自行承担风险。本固件提供网页「重建 UBI」与 `nandecc 4` 串口诊断路径，但 BL2 阶段卡死仍需 TTL 重刷，不存在万能后门。
