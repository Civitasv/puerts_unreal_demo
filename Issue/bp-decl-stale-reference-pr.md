# PR：修复 #2373

- 源分支：`Civitasv/puerts:fix/unreal-blueprint-decl-consistency`
- 目标分支：`Tencent/puerts:master`

## 标题

```
修复 Puerts.Gen 增量生成后 ue_bp.d.ts 无法通过 TS 编译的问题
```

## 描述

```markdown
Fixes #2373

## 问题

蓝图声明是增量生成的：`RestoreBlueprintTypeDeclInfos` 从上一次的 ue_bp.d.ts 恢复每个包的声明块，只有资源版本号变化的包才会重新生成。
当某个缓存块引用的类型在本次没有生成时（被引用的资源被删除、内容插件被禁用、资源加载失败等），这个块仍然被原样输出，ue_bp.d.ts 自相矛盾，TS 编译报 TS2339 / TS2694。

## 修改

只在 `DeclarationGenerator`（编辑器模块）中新增了 `SanitizeBlueprintDeclarations`，在写 ue_bp.d.ts 之前调用，原有逻辑不变（+88 行）：

- 收集本次的已知类型：ue.d.ts 中的原生类型，以及所有会输出的蓝图声明块中声明的类型；
- 把块中引用的未知 `UE.xxx` 类型替换为 `any`，作为父类（`extends`）时替换为 `UE.Object`；
- 对发生过替换的包清空版本号，下次生成时会重新生成，被引用的类型恢复后声明也随之恢复，`any` 不会一直留在缓存里；
- 类型名用 `\w` 匹配，兼容 `FilenameToTypeScriptVariableName` 保留下来的中文等非 ASCII 名字。

已知的边界情况：如果蓝图 tooltip 注释里出现了未知的 `UE.xxx`，或者块引用了 `ShouldSkipBlueprintDeclarationAsset` 跳过的包中的类型，
该包每次生成都会被替换并重新生成（会打印一条 warning）。修改前这两种情况生成的声明本身就无法通过编译。

## 验证

复现工程：https://github.com/Civitasv/puerts_unreal_demo （UE 5.8，Puerts 以 submodule 形式放在 `Plugins/Puerts`）

- `repro/bp-decl-stale-reference`：submodule 为 c5cd096（master），复现步骤见该分支的 `Issue/bp-decl-stale-reference.md`；
- `verify/bp-decl-fix`：与复现分支相同，只是 submodule 切换到了本 PR 的提交。

两个分支都执行一次 `Puerts.Gen`，再执行：

    npx -p typescript@5 tsc --noEmit --experimentalDecorators --target esnext Typing/puerts/index.d.ts Typing/cpp/index.d.ts Typing/ffi/index.d.ts Typing/ue/index.d.ts

| | 修复前（c5cd096） | 修复后（本 PR） |
|---|---|---|
| tsc | `ue_bp.d.ts(11,70): error TS2339: Property 'MixinSuperTestBase' does not exist on type 'typeof StarterContent'.` | 无错误 |
| `MixinSuperTestDerived_C` | `extends UE.Game.StarterContent.MixinSuperTestBase.MixinSuperTestBase_C` | `extends UE.Object` |
| 该块的版本号 | 资源版本号 | 空（下次生成时重新生成） |

修复后的编辑器日志：

    LogTemp: Warning: invalid blueprint: /Game/StarterContent/MixinSuperTestDerived
    LogTemp: Warning: replace unknown type Game.StarterContent.MixinSuperTestBase.MixinSuperTestBase_C in blueprint declaration MixinSuperTestDerived_C of /Game/StarterContent/MixinSuperTestDerived
```

## 验证分支（`verify/bp-decl-fix`）说明

本分支在 `repro/bp-decl-stale-reference` 的基础上只把 submodule `Plugins/Puerts` 切换到了修复提交 b9ff3fc，
按 `Issue/bp-decl-stale-reference.md` 中的复现步骤执行，tsc 不再报错，结果见上表。
