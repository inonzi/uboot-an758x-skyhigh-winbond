# U-Boot for AN758x (Skyhigh/Winbond 8-bit ECC Fix)

## 修改说明

本分支在原版（仅支持 Winbond W29N02KVSIAF）的基础上，仅增加了对 **烽火 HG5585F / HG5382A (天海/Skyhigh S34ML02G300TFI00 颗粒)** 的 8-bit ECC 临时迁移支持。

- **核心改动**：在 `airoha_en7581_nand.c` 中加入了天海的 5 字节 ID (`0x01 0xda 0x00 0x95 0x46`) 匹配逻辑，使其能够触发 8-bit ECC 桥接。
- **配置改动**：为烽火的三款板级 defconfig 开启了 `CONFIG_AIROHA_NAND_ECC_BRIDGE=y`。
- **网页提示**：更新了 Web 恢复界面的中英文提示文案，标识其同时支持 Skyhigh 闪存。

> ⚠️ **注意**：这依然是一个“8-bit 读 / 4-bit 写”的临时迁移固件。主要用于拯救因原厂 8-bit 布局导致变砖的机器，刷入后请通过网页重建 UBI 并迁移至官方正式固件。

## 免责声明 (Disclaimer)

*   本仓库的修改是由 AI（DeepSeek）提供全程技术指导，并在其指示下完成代码定位与修改的。
*   维护者本人并非底层驱动开发专家，无法为底层代码的绝对安全性作担保。
*   请在刷机前务必备份重要数据。若发生不可逆的变砖或硬件损坏，需自行承担风险（不过本固件保留了 `nandecc 4` 的串口切换后门，理论上可以救回）。
