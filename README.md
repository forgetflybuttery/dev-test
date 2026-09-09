请继续重构当前 common/create-pipeline-context/src/main.ts。

目的：
为了后续使用 Jest 对 create-pipeline-context 的内部处理进行单体测试，
进一步减少 run() 中直接实现的业务逻辑，将 Context 生成相关处理 method 化。

当前已经拆分出的函数（例如 createCiBuildFlags、createCiIntegrationFlags、
createCiTestFlags、createCdDeployFlags、validateContext、outputContext、
outputTestParallelKeys）请尽量保持现有实现和行为不变。

请重点重构目前仍然直接写在 run() 中的 Context 生成处理。

要求：

1. 将 run() 中以下类型的处理按照职责拆分为独立 method：
   - GitHub Actions inputs 的读取
   - targetConfigPath 的判断和基础 config 的读取
   - REPO_TYPE 对应的 WORK_REPO / APP_REPO / MANIFEST_REPO 设置
   - input 参数向 config.sys 的设置
   - config_files 的读取、glob、YAML load、deep merge
   - production / non-production 环境相关参数的设置
   - container 的默认值及 image path 设置
   - Cloud Run Service / Cloud Run Job / Cloud Run Function 的 Context 设置
   - PRODUCT / MICROSERVICE / SERVICE 等 Context 设置
   - MODE=test 时的 override
   - TEST_PARALLEL_KEYS 的生成/读取处理

2. 不要为了拆分而过度细分。
   请按照“一个明确职责 = 一个 method”的原则进行拆分。

3. run() 最终主要只负责整体流程控制（orchestration），
   尽量形成类似下面的结构：

   getInputs()
       ↓
   loadConfig()
       ↓
   createPipelineContext()
       ↓
   createCiBuildFlags()
   createCiIntegrationFlags()
   createCiTestFlags()
   createCdDeployFlags()
       ↓
   validateContext()
       ↓
   outputContext()
   outputTestParallelKeys()

4. 拆出的业务逻辑 method 应尽量设计成容易进行 Jest 单体测试的形式。

5. 对于 Context 生成/转换类 method：
   - 尽量通过参数接收 input/config
   - 尽量通过 return 返回处理后的结果
   - 避免在 method 内直接调用 core.getInput()
   - 避免不必要地依赖全局状态

   例如优先考虑：
   function applyEnvironmentConfig(config: Config, inputs: Inputs): Config

   而不是：
   function applyEnvironmentConfig(): void

6. 不要求所有 method 的输入输出都转换成 JSON 字符串。
   TypeScript object / Config / Input object 可以直接作为参数和返回值。
   重点是 method 的输入和输出边界明确、容易进行单体测试。

7. GitHub Actions 特有的 I/O 操作，例如：
   - core.getInput()
   - core.setOutput()
   - core.exportVariable()
   - core.summary
   - fs.writeFileSync()
   尽量保留在 run() 或专门的 I/O method 中，
   不要混入纯粹的 Context 生成业务逻辑。

8. run() 最外层的 try/catch 和 core.setFailed() 错误处理暂时保留，
   不需要为了 method 化而单独拆成 error handler。

9. 必须保持现有功能、判断条件、默认值、错误信息和输出结果完全一致。
   本次只做 refactoring，不改变业务逻辑。

10. 不要删除现有已经拆分完成的 method。
    如果需要调整参数或返回值以提高 testability，可以进行最小限度修改。

11. 对需要直接进行 Jest 单体测试的 method，请考虑 export，
    但不要无意义地 export 所有内部 helper。

请先分析当前 run() 中的处理职责，再进行重构。
重构完成后，请说明：
- 新增/调整了哪些 method
- 每个 method 负责什么
- run() 最终负责什么
- 哪些 method 适合直接进行 Jest 单体测试
- 是否存在为了保持现有行为而没有拆分的处理
