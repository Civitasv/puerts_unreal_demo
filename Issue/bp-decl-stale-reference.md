# [UE] Bug: Puerts.Gen 增量生成的 ue_bp.d.ts 会引用已不存在的蓝图类型，导致 TS 编译失败

- Puer 版本：master（c5cd096，即本工程 submodule `Plugins/Puerts` 固定的版本）
- UE 版本：5.8
- 平台：Editor(win)

## 错误信息

```
Typing/ue/ue_bp.d.ts(11,70): error TS2339: Property 'MixinSuperTestBase' does not exist on type 'typeof StarterContent'.
```

生成的 ue_bp.d.ts 片段：

```ts
// __TYPE_DECL_START: 11f8de2bb8b81440a5d432601273841800000000
    namespace Game.StarterContent.MixinSuperTestDerived {
        class MixinSuperTestDerived_C extends UE.Game.StarterContent.MixinSuperTestBase.MixinSuperTestBase_C {
```

`MixinSuperTestBase` 的声明已经不在 ue_bp.d.ts 里了，但 `MixinSuperTestDerived` 的声明块仍然引用它。

## 原因分析

蓝图声明是增量生成的（以下行号对应 c5cd096 的 `Source/DeclarationGenerator/Private/DeclarationGenerator.cpp`）：

1. `RestoreBlueprintTypeDeclInfos`（598 行）从上一次生成的 ue_bp.d.ts 中恢复每个包的声明块和版本号，
   版本号写在每个块的 `// __TYPE_DECL_START: <版本号>` 中。非 association 的块恢复后 `IsExist` 为 false（644 行）。
2. `LoadAllWidgetBlueprint` 扫描资源，对仍然存在的资源置 `IsExist = true`，
   并且只有版本号变化时才置 `Changed = true`（717、718 行）。版本号是 `GetPackageSavedHash()`（63 行），只取决于 uasset 内容。
3. 只有 `Changed` 的包会重新加载资源并生成声明（411 行），其它包沿用缓存块。
4. 输出时把所有 `IsExist` 的块原样写入 ue_bp.d.ts（464、468 行）。

在复现场景中：

- `MixinSuperTestBase` 已不在资源列表中，它的缓存块 `IsExist` 保持 false，不再输出；
- `MixinSuperTestDerived` 的资源没有变化，版本号相同，直接沿用缓存块，而缓存块里仍然 `extends MixinSuperTestBase_C`。

输出前没有检查缓存块引用的类型在本次生成结果中是否存在，于是 ue_bp.d.ts 自相矛盾。
只要 `MixinSuperTestDerived` 不被重新保存，之后每次增量生成都会保留这个错误。

实际项目里，以下情况都会触发：通过版本控制删除或移动了被其它蓝图引用的资源、禁用了包含被引用蓝图的内容插件、
被引用的资源加载失败等。

## 复现思路

要触发这个问题，需要同时满足两个条件：

1. 存在上一次生成的 ue_bp.d.ts，其中 A 的声明块引用了 B；
2. 本次生成时 B 不存在了，而 A 的资源没有变化（版本号相同，不会重新生成）。

本工程的 `repro/bp-decl-stale-reference` 分支直接提交了满足这两个条件的状态：

- `Typing/ue/ue_bp.d.ts`：在删除 B 之前，用 submodule 中的 puerts 执行 `Puerts.Gen` 得到的结果，作为增量生成的缓存。
  其中 `MixinSuperTestDerived_C`（A）继承自 `MixinSuperTestBase_C`（B）。
- 删除了 `Content/StarterContent/MixinSuperTestBase.uasset`（B），`MixinSuperTestDerived.uasset`（A）保持不变。

版本号只取决于 uasset 内容，所以 clone 下来后 A 的版本号与缓存中一致，只需执行一次 `Puerts.Gen` 就能复现。

## 复现步骤

1. clone 本分支并编译：

   ```
   git clone --recursive -b repro/bp-decl-stale-reference https://github.com/Civitasv/puerts_unreal_demo.git
   ```

   按 README 下载 v8 到 `Plugins/Puerts/unreal/Puerts/ThirdParty`，然后生成工程并编译。
2. 用 UE 5.8 打开编辑器，在控制台执行 `Puerts.Gen`，等待 "generate finish" 提示。
   编辑器提示 `MixinSuperTestDerived` 的父类丢失是预期行为，不要修复或保存它。
3. 在工程根目录执行类型检查：

   ```
   npx -p typescript@5 tsc --noEmit --experimentalDecorators --target esnext Typing/puerts/index.d.ts Typing/cpp/index.d.ts Typing/ffi/index.d.ts Typing/ue/index.d.ts
   ```

- 期望：没有错误。
- 实际：出现上面的 TS2339 错误。

在任意工程中也可以手动复现：先执行一次 `Puerts.Gen`，关闭编辑器后删除某个被其它蓝图引用（例如作为父类）的蓝图资源，
重新打开编辑器再执行 `Puerts.Gen`。
