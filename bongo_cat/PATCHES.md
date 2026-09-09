# BongoCat 宝箱补丁说明

针对 Unity 游戏 **Bongo Cat**（Steam AppID 3419430，Unity 6000.2.8f1）的 `Assembly-CSharp.dll` 重编译补丁。

原始反编译源码位于 `bongo_cat/Assembly-CSharp/`，构建源码树位于 `bongo_cat/Patch/build/src/`（已在反编译源码基础上清理了全部编译阻断产物并应用补丁）。原版 DLL（`bongo_cat/Assembly-CSharp.dll`）未被修改。

## 版本总览

| 版本 | 路径 | MD5 | 宝箱冷却 |
|------|------|-----|---------|
| 原版 | `bongo_cat/Assembly-CSharp.dll` | `25976CE6A3E82CF19228029C79A752DF` | 30 分钟 |
| Patch | `bongo_cat/Patch/Assembly-CSharp.dll` | `764004FB8387AC7157E514B73FD0C2F1` | 30 分钟（保持原版） |
| Patch2 | `bongo_cat/Patch2/Assembly-CSharp.dll` | `6BE1DA8D48A493EDA46E14B6F63AFDDB` | **1 分钟** |

Patch2 = Patch 的全部改动 + 冷却缩短。两者均包含自动领取功能。

## Patch / Patch2 共同包含的改动

### 1. 宝箱自动领取（核心功能）

- **作用**：装扮宝箱（Skin 店）就绪后自动购买领取，无需手动点击。
- **实现**：
  - `Shop.OnChestReady()` / `Shop.TimerUpdate()` 调用新增的 `Shop.AutoClaimChest()`：非 Emote 店、宝箱就绪、余额足够时直接 `_shopItem.Buy()`（与手动点击同一领取链路）。
  - `TimerUpdate` 每秒重试自动领取——宠物币余额由 Steam 统计异步加载，启动瞬间 `CanBuy()` 可能为 false，一次性触发会永久错过。
- **表情宝箱绝不自动领取**（`_isEmoteShop` 排除）。

### 2. 奖励类型过滤：只保留 Skin

- **作用**：开箱结果中仅 `itemslot == "skin"` 的物品才弹窗 / 自动佩戴 / 触发 UI 刷新；帽子等其他物品正常入库存但静默处理。
- **实现**：`ShopItem.Callback()`（实际领取落点）中以 `SteamItemIdReference.Inst[itemId].IsSkin` 判断，非 Skin 跳过弹窗逻辑；宝箱消耗、扣费、倒计时重置照常执行。就绪状态重置抽取为 `ShopItem.ResetChestReady()`。

### 3. 计时器冻结修复（原版潜在 bug 的放大修复）

- **背景**：原版在窗口失焦 + 无输入时把 `OnDemandRendering.renderFrameInterval` 设为 `100000`，冻结整个 player loop（协程/Steam 回调停摆）。打字或鼠标划过猫才恢复——表现为"键盘不动倒计时就停"。自动开箱恰好发生在用户不在时，会卡死宝箱 UI 状态机。
- **修复三处**：
  - `OnDemandRenderHelper.PauseIfIdle()`：`100000 → 60`（空闲约 1 FPS，计时照常推进）。
  - `Shop.TimerUpdate()`：改为基于 `Time.realtimeSinceStartup` 的真实时间戳追赶式计时，冻结恢复后按真实流逝时间补齐。
  - `ShopItem.StartBuy()`：真正发起购买时 `ResumeRenderingForDuration(10f)`，保证开箱状态机（loading → success → 0.8s → 倒计时）完整走完。

## Patch2 独有改动

### 4. 宝箱冷却缩短：30 分钟 → 1 分钟

- **改动点**：`Shop._stockRefreshTime`（硬编码私有字段，无 SerializeField/配置文件）`1800 → 60`。
- **效果**：开箱后 1 分钟即可再开。Steam 库存攒了多个宝箱令牌（上限 10）时，会以每分钟一个的速度连续自动开箱消耗存量。
- **限制**：令牌（`Chest_Token`, item def 98）由 Steam 服务器按游玩时长规则投放，客户端无法加速。改短冷却只是加速消耗存量 + 清空库存槽位让掉落请求持续有效，长期平均产出率不变。
- **回滚**：把 `Patch/Assembly-CSharp.dll` 复制回游戏目录即恢复 30 分钟冷却。

## 部署

```powershell
# 备份（首次）
Copy-Item "E:\SteamLibrary\steamapps\common\BongoCat\BongoCat_Data\Managed\Assembly-CSharp.dll" `
          "E:\SteamLibrary\steamapps\common\BongoCat\BongoCat_Data\Managed\Assembly-CSharp.dll.bak"

# 部署 Patch2（或换成 Patch 路径）
Copy-Item "E:\code\vibecoding\bongo_cat\bongo_cat\Patch2\Assembly-CSharp.dll" `
          "E:\SteamLibrary\steamapps\common\BongoCat\BongoCat_Data\Managed\Assembly-CSharp.dll" -Force
```

注意事项：

- Steam "验证文件完整性" 会还原为官方版本，需重新复制。
- 游戏日志：`C:\Users\<用户>\AppData\LocalLow\Irox Games\BongoCat\Player.log`（`Player-prev.log` 为上一次运行）。
- 无需清理注册表倒计时残留：启动时 `Mathf.Min(存档值, _stockRefreshTime)` 自动钳制。

## 重新构建方法

源码树：`Patch/build/src/`（313 个 .cs）。修改后用 Roslyn csc 编译，要点：

1. **输出文件名必须为 `Assembly-CSharp.dll`** —— csc 以输出文件名作为程序集内部名，场景中 MonoBehaviour 绑定嵌入程序集名，名称不符会导致全部脚本绑定失败（黑屏、missing script 警告）。
2. 必须使用 **.NET Framework 4.8 引用程序集**（`C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.8\`）编译 mscorlib/System 等——游戏 Managed 目录下的 mscorlib 是缺 `MinValue/MaxValue` 等常量的运行时门面，不能用于编译。
3. 其余引用取自游戏 `BongoCat_Data\Managed\`（Assembly-CSharp-firstpass、Heathen.Steamworks、DOTween、UnityEngine.* 等 22 个）。

构建命令模板（PowerShell，工作目录为项目根）：

```powershell
$REF="C:\Program Files (x86)\Reference Assemblies\Microsoft\Framework\.NETFramework\v4.8"
$M="E:\SteamLibrary\steamapps\common\BongoCat\BongoCat_Data\Managed"
$csc="C:\Program Files\dotnet\sdk\8.0.200\Roslyn\bincore\csc.dll"
$refs=@("$REF\mscorlib.dll","$REF\System.dll","$REF\System.Core.dll","$REF\System.Numerics.dll",
  "$M\Assembly-CSharp-firstpass.dll","$M\com.rlabrecque.steamworks.net.dll","$M\Discord.Sdk.dll",
  "$M\DOTween.dll","$M\Heathen.Steamworks.dll","$M\Helper.Runtime.dll","$M\Newtonsoft.Json.dll",
  "$M\SDL2-CS.dll","$M\Unity.Localization.dll","$M\Unity.TextMeshPro.dll",
  "$M\UnityEngine.AudioModule.dll","$M\UnityEngine.CoreModule.dll",
  "$M\UnityEngine.ImageConversionModule.dll","$M\UnityEngine.IMGUIModule.dll",
  "$M\UnityEngine.InputLegacyModule.dll","$M\UnityEngine.JSONSerializeModule.dll",
  "$M\UnityEngine.UI.dll","$M\UnityEngine.UIModule.dll","$M\UnityEngine.UIElementsModule.dll"
) | ForEach-Object { "-r:$_" }
$sources=(Get-ChildItem "Patch\build\src" -Recurse -Filter *.cs | ForEach-Object { $_.FullName })
& "C:\Program Files\dotnet\dotnet.exe" exec $csc -nologo -target:library -nostdlib `
  -nowarn:0169,0649,0436,0162,1090,0219,0414,0252,0253 `
  -out:Patch2\Assembly-CSharp.dll @refs @sources
```

## 修改过的源码文件（相对原反编译源码）

| 文件 | 改动 |
|------|------|
| `BongoCat/Shop.cs` | `AutoClaimChest` 新增；`TimerUpdate` 追赶式计时 + 每秒重试；`_stockRefreshTime` 1800→60（仅 Patch2） |
| `BongoCat/ShopItem.cs` | `Callback` 按 `IsSkin` 过滤；`ResetChestReady` 抽取；`StartBuy` 渲染保活 |
| `BongoCat/OnDemandRenderHelper.cs` | `PauseIfIdle` 空闲帧间隔 100000→60 |

另有 6 个文件的反编译产物清理（`<>9__N` 委托重命名、闭包类重构、`<PrivateImplementationDetails>` 替换、`BongoCat.UnityEngine` 命名空间遮蔽加 `global::`、netstandard2.1 API 降级重写等），均为编译必需的等价重写，不改变行为。
