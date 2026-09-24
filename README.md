# Skinner47 · dya 分支

这个分支把 Skinner47 接到 **DYA Studio**（cormoran 的 ZMK Studio 增强版）上，
轨迹球驱动和 keyball `dya-nv` 分支用的是同一套：cormoran 的 PMW3610 驱动
（devicetree 兼容名 `cormoran,pmw3610`）+ DYA 的 custom Studio RPC 模块。

`main` 分支保持原样，本分支为新增的 `dya` 分支。

## 与 main 分支的差别

| 项目 | main | dya（本分支） |
| --- | --- | --- |
| ZMK | `zmkfirmware/zmk@main` | `cormoran/zmk@main+dya` |
| Zephyr | 随 ZMK 决定 | `cormoran/zephyr@v4.1.0+zmk-fixes+nrf-half-duplex-uart` |
| 轨迹球驱动 | badjeff `pixart,pmw3610` | cormoran `cormoran,pmw3610`（与 keyball dya-nv 相同） |
| 跨半输入 | badjeff split relay 模块 | 不需要：轨迹球所在的右半就是中央 |
| Studio | 官方 ZMK Studio（键位编辑） | 官方功能 + DYA Studio（轨迹球、连接、设置、诊断） |
| 板级定义 | `boards/arm/...`（HWMv1） | `boards/yangxing/...` + `board.yml`（Zephyr HWMv2） |

### 已启用的 DYA Studio 功能

* **Keymap**：官方 ZMK Studio 键位/层编辑，布局预览里会画出轨迹球位置；
  **Macro** 子页可以在网页上创建/改名/删除运行时宏（cormoran/zmk-feature-runtime-macro）；
  **Combo** 子页可以编辑运行时组合键（cormoran/zmk-feature-runtime-combo）
* **Trackball**：CPI、轴方向、smart algorithm、downshift/sample 等参数在线调整；
  运行时可调的输入处理器（速度、旋转、轴吸附、自动鼠标层）
* **Connection**：BLE profile 管理、OS 自动识别、按连接/OS 切换默认层
* **Settings**：休眠/空闲超时等设置、通用 custom settings、电池历史
* **Troubleshooting**：device info、watchdog 重启原因、KSCAN 诊断

## 构建

本地（需要 `west`、Zephyr SDK、`protoc`）：

```sh
make init-standalone   # 下载依赖到 ./dependencies
make build-all         # 输出到 ./build/<artifact>/zephyr/zmk.uf2
```

也可以直接用 GitHub Actions 的 `Build ZMK firmware` 工作流，产物同上四个。

## 烧录

| 文件 | 用途 |
| --- | --- |
| `skinner47_right.uf2` | 右半 = **主手（中央）**，接 USB / 连蓝牙的那一半，轨迹球也在这一半 |
| `skinner47_left.uf2` | 左半 = 副手（外设） |
| `skinner47_left_reset.uf2` / `skinner47_right_reset.uf2` | 清空已保存的设置（键位、custom settings、电池历史、BLE 配对），从固件默认值重新开始 |

刷完固件后建议先刷一次 `*_reset.uf2`（两边都要），再刷正式固件，避免旧的
设置分区内容影响新配置。

## 使用 DYA Studio

1. 用 USB 连接右半（主手）
2. 打开 <https://studio.dya.cormoran.works/>
3. 「Connect via USB」连接键盘

本分支关闭了 Studio 自动锁定（`CONFIG_ZMK_STUDIO_LOCKING=n`），所以默认可以直接
读写。键位里也保留了 `&studio_unlock`：**按住 SPACE（MOUSE 层）+ 左下角那颗键**
即可在需要时解锁 Studio。

## 键位上的几处新增行为

* **自动鼠标层**：转动轨迹球 200ms 后自动激活第 4 层（MOUSE），停手 400ms 后自动
  退出，因此不用先按层键就能点击/滚轮；这两项都可以在 DYA Studio 里改。
* **滚轮层 / snipe 层**：第 5 层（SCROLL）轨迹球变成滚轮，第 6 层（SNIPE）变成
  1/3 速度慢速移动，和 main 分支的行为一致，参数同样可以在 Studio 里调。
* **运行时宏**：DYA Studio 的 Macro 页里新建宏后，会分配到一个槽位号
  （0 ~ 7，见网页里的宏列表）；键位上用 `&rmacro <槽位号>` 播放。当前固件在
  **按住 SPACE + 左下角第二颗键** 上绑了 `&rmacro 0` 作示例，空槽位按下去没有
  任何动作；要绑别的键，直接在该层改成 `&rmacro N`，或干脆用 DYA Studio 的
  Keymap 页在线改。
  宏的数量/大小上限：8 个宏、每个最大 256 字节、名字最长 24 字节（共享 1KB
  内存池），需要的话在 `skinner47_right_defconfig` 里调
  `ZMK_RUNTIME_MACRO_COUNT` / `_MAX_BYTES` / `_POOL_BYTES` / `_NAME_MAX_LEN`。
* **运行时组合键**：DYA Studio 的 Combo 页里按槽位编辑「哪几个键位同时按下 →
  触发什么行为」，也可以给槽位起名字。固件里**没有预置任何组合键**，所以在网页上
  添加之前键盘行为不变；全局的 timeout / slow-release / require-prior-idle 也在
  该页设置。默认 8 个槽位、每个最多 16 个键位，需要更多就改
  `ZMK_RUNTIME_COMBO_MAX_COMBOS` / `_MAX_POSITIONS_PER_COMBO`（每个槽位约占 64B
  RAM）。如果想预置默认组合键，可以在 keymap 里加一个
  `cormoran,runtime-combo-defaults` 节点，网页上还能「Reset to Default」恢复。

## 注意事项

* 板级定义已迁移到 Zephyr **HWMv2**。ZMK `main` 从 Zephyr 4.1 开始要求这一点，
  旧写法（`boards/arm/...` + `Kconfig.board`）在当前 ZMK 上无法构建。
* 主手是**右半**（`CONFIG_ZMK_SPLIT_ROLE_CENTRAL` 在 `skinner47_right` 上）：
  USB、蓝牙、ZMK Studio / DYA Studio 的 RPC 全部在右半，插哪边都一样，插右半即可。
* 轨迹球也挂在右半，刚好和主手同一半，所以是本地直连、不再需要跨半转发；
  传感器设置对外暴露的键名前缀是 `ball`（例如 `cpi@ball`）。
* 左半（副手）只跑键盘矩阵和屏幕：不启用 Studio、不带轨迹球驱动，只保留与主手
  同步设置所需的 relay。若以后想把主手换回左边，改动集中在
  `Kconfig.defconfig`（`ZMK_SPLIT_ROLE_CENTRAL`）、两个 `*_defconfig` 和
  `build.yaml` 里的 `studio-rpc-usb-uart` 归属。
* `CONFIG_ZMK_BATTERY_HISTORY=y` 会周期性写入 flash（每小时左右一次）。如果不看
  电池历史，可以在
  `config/boards/yangxing/skinner47/skinner47_left_defconfig` 里关掉这两个开关，
  以减少 flash 写入。
* `&bootloader` 依赖 ZMK 的 boot retention，本分支已经按 nRF52840 UF2 引导程序
  配好（`nrf52840_uf2_boot_mode.dtsi` + `RETAINED_MEM`/`RETENTION`）。
