# unreal demo for puerts

## 复现：增量生成的 ue_bp.d.ts 引用已不存在的蓝图类型

本分支用于复现 Puerts.Gen 增量生成蓝图声明时，缓存块引用了本次未生成的类型，导致 ue_bp.d.ts 无法通过 TS 编译的问题。

本分支相对 master 的改动：

* 删除了 `Content/StarterContent/MixinSuperTestBase.uasset`（模拟版本控制删除资源、内容插件被禁用等情况），`MixinSuperTestDerived`（父类是 MixinSuperTestBase）保持不变；
* `Typing/ue/ue_bp.d.ts` 是删除前用 submodule 中的 puerts 生成的结果，作为增量生成的缓存；
* `EngineAssociation` 为 5.8（复现时使用的引擎版本）。

步骤：

1. `git clone --recursive -b repro/bp-decl-stale-reference https://github.com/Civitasv/puerts_unreal_demo.git`，按下文“怎么跑这demo”安装 v8 并编译；
2. 打开编辑器（提示 MixinSuperTestDerived 父类丢失属预期，不要修复或保存它），在控制台执行 `Puerts.Gen`；
3. 在工程根目录执行：

   ~~~
   npx -p typescript@5 tsc --noEmit --experimentalDecorators --target esnext Typing/puerts/index.d.ts Typing/cpp/index.d.ts Typing/ffi/index.d.ts Typing/ue/index.d.ts
   ~~~

   期望没有错误，实际：

   ~~~
   Typing/ue/ue_bp.d.ts(11,70): error TS2339: Property 'MixinSuperTestBase' does not exist on type 'typeof StarterContent'.
   ~~~

   生成的 `MixinSuperTestDerived_C` 仍然 `extends UE.Game.StarterContent.MixinSuperTestBase.MixinSuperTestBase_C`，而 MixinSuperTestBase 的声明已经不存在。

## 怎么跑这demo？

* git clone --recursive https://github.com/chexiongsheng/puerts_unreal_demo.git 

* ue4.25及以上版本，需要根据[文档](https://puerts.github.io/docs/puerts/unreal/install)下载安装v8。Puerts插件以submodule的形式放在`Plugins/Puerts`，插件本体在`Plugins/Puerts/unreal/Puerts`，v8要解压到`Plugins/Puerts/unreal/Puerts/ThirdParty`下

* 由于插件多嵌套了两层，编译时JsEnv.Build.cs不会把插件的`Content`、`Typing`同步到本工程（会拷到`Plugins/Puerts`根目录）。本工程已提交与submodule版本对应的拷贝，更新submodule后需要手动同步：`cp -r Plugins/Puerts/unreal/Puerts/Content/. Content/ && cp -r Plugins/Puerts/unreal/Puerts/Typing/. Typing/`

* 生成vs工程（window下，在mac下生成xcode工程），然后进入工程编译

* 如果要打包运行，要到“项目设置/打包/Additional Not-Asset Directories to Package”，把Content下的“JavaScript”目录添加进去。

* 如果打开基础ue类功能，可能会报找不到react-reconciler的错误，需要到Content/JavaScript下执行`npm install .`安装依赖的库。

## Demo说明

* 本demo目录是一个Typescript工程，可以用vscode（建议安装vscode，nodejs，typescript）打开这个目录。

* [TsGameInstance.cpp](Source/puerts_unreal_demo/TsGameInstance.cpp)是程序逻辑的入口，里头关键语句是JsEnv->Start("QuickStart")，其执行了启动脚本QuickStart.js。

* QuickStart.js是由[TsProj](TsProj)下QuickStart.ts的编译结果，所以直接看QuickStart.ts即可，里头演示主要功能的用法。

* 编译，在vscode上“Terminal -> Run Build Task”选tsc watch，修改代码后会自动编译。

* PerfTest.ts编辑器下在老版本v8以及quickjs后端不可用

* 若采用较新版本的vscode进行调试，发现无法命中断点，则尝试将本demo目录根目录下".vscode"目录中的"launch.json"中的"remoteRoot"配置项的"${workspaceRoot}"修改为"${workspaceFolder}"，或者直接去除"remoteRoot"配置项。

* UE某5.3，ReactUMG插件会编译报错，疑似该版本ue的bug，如果碰到可以修改puerts_unreal_demo.uproject禁用该plugin。puerts_unreal_demo.uproject内的Plugins配置修改如下：

~~~json
"Plugins": [
		{
			"Name": "Puerts",
			"Enabled": true
		},
		{
			"Name": "ReactUMG",
			"Enabled": false
		}
	]
~~~