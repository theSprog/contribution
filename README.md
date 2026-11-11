# 开源贡献记录

## 状态说明

| 状态 | 含义 | 凭据 |
|---|---|---|
| 已提交 | 补丁已发到项目邮件列表，正在等待评审 | 补丁邮件存档 |
| 已获接受 | 维护者给出 Reviewed-by，或明确回复会合入，但代码还没有出现在公开仓库 | 维护者回复存档 |
| 已合入 | 代码已进入项目的公开 Git 仓库 | commit 链接 |
| 已被取代 | 同一补丁发了新版本，以新版本为准 | 新版本链接 |



## 补丁列表

| 项目 | 补丁 | 状态 | 链接 |
|---|---|---|---|
| ROCm/KCL | drm/amdkcl: use DMA iterator in PRIME address fallback | 已获接受 | https://lists.freedesktop.org/archives/amd-gfx/2026-November/156842.html |
| ROCm/DKMS | drm/amdkcl: run the remaining foreground probes in background | 已获接受 | https://lists.freedesktop.org/archives/amd-gfx/2026-November/157014.html |
| Linux/amdgpu | [PATCH v2 1/3] drm/amdgpu: only unregister the ATPX handler if we registered it | 已获接受 | https://lists.freedesktop.org/archives/amd-gfx/2026-November/157097.html |
| Linux/amdgpu | [PATCH v2 2/3] drm/amdgpu: undo module init in reverse order in module exit | 已获接受 | https://lists.freedesktop.org/archives/amd-gfx/2026-November/157098.html |
| Linux/amdgpu | [PATCH v2 3/3] drm/amdgpu: unwind module init on failure | 已获接受 | https://lists.freedesktop.org/archives/amd-gfx/2026-November/157099.html |
| Linux/radeon | drm/radeon: only unregister the ATPX handler if we registered it | 已获接受 | https://lists.freedesktop.org/archives/amd-gfx/2026-November/157101.html |
| Linux/amdgpu | [PATCH] drm/amdgpu: unwind module init on failure | v1，由上面的 v2 系列取代 | https://lists.freedesktop.org/archives/amd-gfx/2026-November/157013.html |
| Linux/e1000 | [PATCH iwl-net] e1000: report RX checksum offload as on when it is enabled | 已提交 | https://patchwork.kernel.org/project/netdevbpf/patch/OS3PR01MB10251F4C0DDF4A1AA90095494E7702@OS3PR01MB10251.jpnprd01.prod.outlook.com/ |



## 具体补丁

### drm/amdkcl: use DMA iterator in PRIME address fallback

**一句话解释**：`drm_prime_sg_to_dma_addr_array` 和 `drm_prime_sg_to_page_addr_arrays` 在 5.4.18 上语义不等价，但是 AMD 的 KCL 却错误的认为他们在 5.4.18 上是等价的

**问题**：amdgpu 导入其他设备导出的 DMA-BUF 时，需要把 scatterlist 展开成逐页的 DMA 地址数组，交给 GPU 页表使用。内核没有 `drm_prime_sg_to_dma_addr_array()` 时（例如 5.4），KCL 改用 `drm_prime_sg_to_page_addr_arrays()`。后者按 CPU 侧的 `sg->length` 遍历每个条目，而 IOMMU 映射时会把多个条目合并成一个更长的 DMA 段，DMA 段的长度和条目数都与 CPU 侧不一致。

**修复**：改用 `for_each_sg_dma_page()`，按 DMA 映射后的段和长度逐页取地址。主线的修复 `c0f83d164fb8` 没有回合到 5.4.y，所以只升级 5.4 的 stable 版本解决不了这个问题，只能在 KCL 里修。

**状态**：AMD 维护者回复会合入（“Thanks for the patch and the fix. We'll apply your fix.”）。

**链接**：

- patch：https://lists.freedesktop.org/archives/amd-gfx/2026-November/156842.html
- reviewed-by：https://lists.freedesktop.org/archives/amd-gfx/2026-November/156932.html
- lore：https://lore.kernel.org/r/OS3PR01MB102518A543E274E19D5FC0000E7772@OS3PR01MB10251.jpnprd01.prod.outlook.com



### drm/amdgpu: only unregister the ATPX handler if we registered it

**一句话解释**：`init` 路径和 `exit` 路径不对称，且 `init` 初始化失败时并没有回滚所有资源

- **ATPX handler 被误注销**（1/3 和 radeon 补丁）：ATPX 是双显卡笔记本上切换独显的 ACPI 接口，驱动通过 `vga_switcheroo` 注册它的 handler。系统里同一时间只能有一个 handler：amdgpu 没有检测到 ATPX 时不注册，已有其他 handler（例如 Apple 机型的 apple-gmux）时注册返回 `-EINVAL` 也被忽略，但卸载时却无条件调用 `vga_switcheroo_unregister_handler()`，把别的驱动的 handler 清掉。之后 `vga_switcheroo` 的 runtime PM 回调访问 handler 时没有判空，会解引用空指针。修复是记下注册是否成功，只注销自己注册的那一个。radeon 的这段代码相同，单独提交了同样的修复。

- **初始化失败时回滚不完整**（3/3）：`amdgpu_init()` 中 KFD 初始化失败时跳过了 ACPI 和 ATPX 的释放，`pci_register_driver()` 失败时直接返回，前面初始化的资源都没有释放，模块加载失败后会留下已注册的 handler。修复后按初始化的逆序逐级回滚。

- **卸载顺序**（2/3）：把 `amdgpu_exit()` 中 ATPX 和 ACPI 的释放改成严格的初始化逆序，使它与 3/3 的错误路径一致。KFD 和 PCI 驱动的注销保持原有顺序，并在代码里写明原因：KFD 进程的上下文可能在 `/dev/kfd` 关闭后仍然存在，必须在设备还在时由 KFD 清理，先注销 PCI 驱动会让设备移除路径等待这些进程。

**状态**：amdgpu/radeon 维护者 Alex Deucher 对 v2 系列和 radeon 补丁都回复了“Reviewed and applied”，但公开的 `amd-staging-drm-next` 分支还没有出现对应 commit，所以记为已获接受，commit 公开后改为已合入。v1 只包含 3/3 的修改，之后发现了 ATPX 归属问题，扩展成 v2 系列，并在 v1 的邮件下回复说明已被取代。

**链接**：

- cover-letter：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157096.html
- 1/3 patch：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157097.html
- 2/3 patch：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157098.html
- 3/3 patch：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157099.html
- radeon：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157101.html
- applied（v2 系列）：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157302.html
- applied（radeon）：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157303.html
- lore：https://lore.kernel.org/r/OS3PR01MB1025124AAC2F9492FCB206AC6E7722@OS3PR01MB10251.jpnprd01.prod.outlook.com



### drm/amdkcl: run the remaining foreground probes in background

**一句话解释**：某些m4探测片段没有在自己独立目录中执行导致影响后续内核特征判断

**背景**：DKMS 安装时，configure 通过编译一小段测试代码来判断目标内核有哪些 API。其他探测都在各自的临时目录里并行运行，只有 HDMI/DP colorspace 属性和 legacy gamma 这三个探测在共享目录里串行运行，能看到之前的探测留下的 `conftest.o`。在 ROCm 6.0.0 和 v5.17 内核上，探测命令末尾串着 `objtool`，编译失败后 `objtool` 处理旧的 `conftest.o` 成功，探测被误判为通过，把只有 1 个参数的函数当成 2 个参数，导致编译失败。这个问题我在 [issue #228](https://github.com/ROCm/amdgpu/issues/228) 中报告过，master 上已由 `a4e55d0e6d9a` 修复。

**修改**：让这三个探测和其他探测一样在独立目录中后台运行，消除共享状态。每个探测单独包装，因为 colorspace 组里已有一个后台探测，从后台任务里再启动的任务不会被顶层的等待逻辑等待。

**状态**：AMD 维护者给出 Reviewed-by，并回复会合入。

**链接**：

- patch：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157014.html
- reviewed-by：https://lists.freedesktop.org/archives/amd-gfx/2026-November/157020.html，https://lists.freedesktop.org/archives/amd-gfx/2026-November/157023.html
- lore：https://lore.kernel.org/r/OS3PR01MB102510E3381EAC477967C91D2E7722@OS3PR01MB10251.jpnprd01.prod.outlook.com



### e1000: report RX checksum offload as on when it is enabled

**一句话解释**：e1000 硬件的 RX 校验和卸载处于开启状态，但 `features` 中却忘记加入 `NETIF_F_RXCSUM`

**问题**：e1000 驱动加载时，接收校验和卸载（RX checksum offload）实际是开着的，`ethtool -k` 却显示 `rx-checksumming: off`。`e1000_probe()` 先把 `hw_features` 复制进 `features`，之后才往 `hw_features` 里加 `NETIF_F_RXCSUM`，所以这个特性默认显示为关闭；而模块参数 `XsumRX` 默认为 1，驱动照样在 RXCSUM 寄存器里打开 TCP/UDP 校验，协议栈也直接信任硬件的结果。后果是 `ethtool -K ethX rx off` 不起作用：请求的特性和当前特性相同，`e1000_set_features()` 不会被调用，只能先 `rx on` 再 `rx off` 才能真正关掉。这个问题由 `e97d3207c5e6`（"e1000: convert to ndo_fix_features"）引入，在那之前 ethtool 读的是 `adapter->rx_csum`。

**修复**：在 `e1000_check_options()` 之后，如果 RX 校验和卸载处于开启状态，就把 `NETIF_F_RXCSUM` 加进 `features`。`XsumRX=0` 时行为不变。

**状态**：已提交，等待 Intel 有线网卡驱动维护者评审。补丁按 Intel 驱动的流程发往 intel-wired-lan 和 netdev，netdev patchwork 的状态是 `awaiting-upstream`，表示由 Intel 的 `net-queue` 树收取后再提交给 netdev。

**链接**：

- patchwork：https://patchwork.kernel.org/project/netdevbpf/patch/OS3PR01MB10251F4C0DDF4A1AA90095494E7702@OS3PR01MB10251.jpnprd01.prod.outlook.com/
- lore：https://lore.kernel.org/r/OS3PR01MB10251F4C0DDF4A1AA90095494E7702@OS3PR01MB10251.jpnprd01.prod.outlook.com
