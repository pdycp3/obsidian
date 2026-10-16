set CODEX_API_KEY=sk-local
# 2. 进入你的项目目录
cd D:\你的项目路径
# 3. 用本地模型启动
codex --profile deepseek


你遇到的这个错误，是 Codex CLI 新版本（0.134.0+）修改了配置文件的规则导致的。它现在**不允许**把 `[profiles.deepseek]` 这样的配置块直接写在 `config.toml` 主文件里，而是要求你把它单独放到一个专属的文件里[](https://github.com/crasuna/openai-dev-docs-cn-mirror/blob/main/sources/en/codex/config-advanced.md#1)[](https://github.com/mehmetbaykar/codex-docs-skill/blob/main/skills/codex-docs/references/config-file__config-advanced.md#1#1)。

### 🛠️ 解决方法：把配置移到新文件

你需要把 `config.toml` 里的 `[profiles.deepseek]` 部分，移动到一个名为 `deepseek.config.toml` 的新文件中。这个新文件需要和 `config.toml` 放在同一个目录下（也就是 `C:\Users\pengd\.codex\`）[](https://developers.openai.com/ja-JP/docs/config-file/config-reference)[](https://github.com/crasuna/openai-dev-docs-cn-mirror/blob/main/sources/en/codex/config-advanced.md#1)。

**第一步：修改 `C:\Users\pengd\.codex\config.toml`**

打开这个文件，删除 `[profiles.deepseek]` 和 `[profiles.qwen]` 这两个配置块，只保留 `[model_providers.*]` 的定义。修改后，文件内容应该像这样：

toml

# ========== Provider 定义 ==========
[model_providers.deepseek-local]
name = "DeepSeek-V4-Flash-0731"
base_url = "http://172.16.4.131:30000/v1"
api_key_env = "CODEX_API_KEY"
wire_api = "responses"
[model_providers.qwen38-27b]
name = "Qwen3.8-27B-FP8"
base_url = "http://172.16.4.131:8002/v1"
api_key_env = "CODEX_API_KEY"
wire_api = "responses"

**第二步：创建 `C:\Users\pengd\.codex\deepseek.config.toml`**

新建一个文本文件，命名为 `deepseek.config.toml`，在里面填入原来 `[profiles.deepseek]` 的内容，但注意**要去掉 `[profiles.deepseek]` 这个标题行**，直接把配置写成顶层格式[](https://github.com/crasuna/openai-dev-docs-cn-mirror/blob/main/sources/en/codex/config-advanced.md#1)：

toml

model_provider = "deepseek-local"
model = "DeepSeek-V4-Flash-0731"

**第三步（可选）：为 Qwen 也创建独立文件**

如果你也需要用 `--profile qwen`，就再新建一个 `C:\Users\pengd\.codex\qwen.config.toml`，内容为：

toml

model_provider = "qwen38-27b"
model = "Qwen3.8-27B-FP8"

### ✅ 再次运行

完成上述修改后，回到你的项目目录，就可以正常启动 Codex 了：

powershell

cd D:\codex-test
codex --profile deepseek