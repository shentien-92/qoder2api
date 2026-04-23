# Qoder2API

将 Qoder 的 Chat API 桥接为 OpenAI 兼容的 `/v1/chat/completions` 接口，支持 stream 和非 stream 模式。

![q](images/img.png)

## 原理

项目伪装成 Qoder CLI 客户端（v0.1.43），通过抓包还原的请求格式与 Qoder 后端通信，并将响应转换为 OpenAI API 格式。这样任何支持 OpenAI API 的工具都能直接对接 Qoder 的模型。

请求流程：

```
客户端 --OpenAI格式--> Bridge(:8963) --Qoder格式--> api3.qoder.sh
客户端 <-OpenAI格式--- Bridge(:8963) <--SSE流---- api3.qoder.sh
```

## 环境要求

- Java 17+
- Maven

## 构建

```bash
mvn clean package
```

生成 fat jar: `target/qoder-client-0.1.0.jar`

## 运行

```bash
java -DQODER_PAT=<your_token> -jar target/qoder-client-0.1.0.jar
```

启动后监听 `http://127.0.0.1:8963/v1/chat/completions`

### 获取 PAT

Qoder 的 Personal Access Token。两种方式：

1. **从本地凭据读取** — 如果已登录过 Qoder CLI，凭据存在 `~/.qoder/.auth/`，运行 `LocalAuth.main()` 可解密查看
2. **从 Qoder 设置页面生成**

### 模型选择

请求体中的 `model` 字段会透传给 Qoder 后端作为 `model_config.key`，默认为 `lite`。

## 使用示例

```bash
curl http://127.0.0.1:8963/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "lite",
    "stream": true,
    "messages": [{"role": "user", "content": "hello"}]
  }'
```

## 注意事项

- 必须在项目根目录运行，程序会读取当前目录下的 `baseprompt.json` 作为请求模板
- `baseprompt.json` 包含完整的 Qoder 系统提示词和工具定义（~65KB，约 16k-18k tokens），每次请求固定消耗约 10% 的输入窗口
- 仅监听 127.0.0.1，不对外暴露

## 项目结构

```
src/main/java/us/cubk/
  OpenAiBridge.java       # 主入口，HTTP 服务器 + OpenAI 格式转换
  SignatureApiClient.java # Qoder API 签名与认证
  BearerApiClient.java    # Bearer token 鉴权的 HTTP 客户端
  BearerBuilder.java      # 会话构建器
  JobTokenClient.java     # Job token 交换
  LocalAuth.java          # 本地凭据解密（~/.qoder/.auth/）
  QoderEncoding.java      # 请求体编码
  Signature.java          # 请求签名生成
baseprompt.json           # Qoder Chat API 请求模板
```
