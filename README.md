请重构当前 `common/create-pipeline-context/src/main.ts`，目标是提高可测试性，并为后续 Jest 单体测试做准备。

当前问题：
- `run()` 从读取 GitHub Actions input、读取/merge config、生成 pipeline context、计算 CI/CD flags、validation、output 全部集中在一个很大的 method 中。
- 目前 `run()` 过长，单体测试 scope 不清晰。
- 希望保持现有业务逻辑、输出结果、错误处理行为完全不变，仅进行结构性 refactoring。

请按以下原则修改：

1. 保留 `run()` 作为 GitHub Actions 的入口方法，但让 `run()` 只负责 orchestration，不再包含大量业务逻辑。
2. 将 `run()` 内部处理按职责拆成独立、可测试的 functions。
3. 优先拆分以下部分：
   - GitHub Actions inputs 读取
   - target config path 决定逻辑
   - base config load / config files merge
   - repository / APP_REPO / MANIFEST_REPO 设置
   - environment（prod / nprd）相关 context 设置
   - container / Cloud Run / Cloud Run Job / Cloud Run Function 相关设置
   - CI build flag 生成
   - CI integration flag 生成
   - CI test flag 生成
   - CD deploy flag 生成
   - validation
   - pipeline context / GITHUB_ENV / GITHUB_OUTPUT / Job Summary 输出
4. 已经存在的简单 utility functions，例如：
   - `convertFlatConfig`
   - `convertEnvConfig`
   - `replaceEnv`
   - `nvl`
   - `existsFile`
   不要无意义地继续拆分。
5. 对有业务判断逻辑的 function 使用 `export`，以便 Jest 可以直接 import 并进行 unit test。
6. 不要把所有 function 强制设计成 JSON string 输入 / JSON string 输出。
   - production code 应继续使用合适的 TypeScript object / Config 类型。
   - JSON 仅作为 Jest test fixture 的保存形式。
7. 尽量把纯业务逻辑和以下 I/O 分离：
   - `core.getInput`
   - `core.setOutput`
   - `core.exportVariable`
   - `fs.readFileSync`
   - `fs.writeFileSync`
   - `glob`
   - `execSync`
8. 如果某段逻辑依赖外部 I/O，请尽量通过参数传入必要数据，使核心判断逻辑可以直接 unit test。
9. 不要改变任何现有 flag 名称、config key、默认值、条件判断、branch name 生成规则、deploy target 判断规则。
10. 不要改变现有 GitHub Actions 对外接口。
11. 不要删除现有 validation 和 error message。
12. 重构后现有行为必须与重构前一致。

期望的整体结构类似：

async function run(): Promise<void> {
  try {
    const inputs = getActionInputs();

    let config = loadBaseConfig(inputs);
    config = await mergeConfigFiles(config, inputs);

    config = applyRepositoryConfig(config, inputs);
    config = applyEnvironmentConfig(config, inputs);
    config = applyDeployTargetConfig(config, inputs);

    config = createCiBuildFlags(config, inputs);
    config = createCiIntegrationFlags(config, inputs);
    config = createCiTestFlags(config, inputs);
    config = createCdDeployFlags(config, inputs);

    validateContext(config, inputs);

    await outputContext(config, inputs);
  } catch (error) {
    if (error instanceof Error) {
      core.setFailed(error.message);
    }
  }
}

函数名可以根据现有代码内容调整，不要求完全使用以上名称，但必须做到：
- 一个 function 一个明确职责
- test scope 容易理解
- 可以单独 import 到 Jest test
- run() 本身保持简洁

特别优先将当前这些注释块提取为独立 function：
- `// flag設定 ci-build`
- `// flag設定 ci-integration`
- `// flag設定 ci-test`
- `// flag設定 cd-deploy`
- `// バリデーションチェック`
- `// Output context`

重构完成后，请继续做以下事情：

A. 列出你新增/拆分出来的 functions，并说明每个 function 的职责。
B. 指出哪些 functions 最适合做 Jest unit test。
C. 为以下 functions 各生成至少 2～3 个 Jest 测试示例：
   - CI build flag calculation
   - CI integration flag calculation
   - CI test flag calculation
   - CD deploy flag calculation
D. 测试中复杂的 input / expected result 可以使用 JSON fixture，例如：
   `test/fixtures/<case-name>/input.json`
   `test/fixtures/<case-name>/expected.json`
E. Jest 的比较方式优先使用：
   `expect(actual).toEqual(expected)`
F. 对简单函数不要为了使用 JSON 而额外复杂化测试。
G. 在修改前先分析现有 `run()` 的逻辑区块，再进行 refactoring，避免遗漏任何处理。
H. 如果发现某个逻辑拆分后可能改变执行顺序或副作用，请优先保持原顺序，不要擅自优化业务逻辑。

请先完成 refactoring，再生成对应的 Jest test skeleton。
请优先做最小范围、低风险的重构。第一阶段先提取 ci-build、ci-integration、ci-test、cd-deploy、validation、output 这几个明显区块，不要一次性重写整个文件。
错误处理也请按职责进行整理：

- `run()` 最外层的 try/catch 和 `core.setFailed(error.message)` 作为 GitHub Actions 入口层的统一异常处理，保留在 run() 中，不需要单独提取。
- 当前业务规则导致的错误判断（例如 blue-green / rollback、CloudRun Function、macaron2 等）属于 validation，请从 run() 中提取到 `validateContext()`。
- 第一阶段重构时不要改变现有 error message、core.setFailed() 的调用条件和执行行为。
- 文件读取、execSync、JSON.parse 等局部异常处理，如果属于特定处理本身，则保留在对应拆分后的 function 内，不要为了拆分而拆分。

