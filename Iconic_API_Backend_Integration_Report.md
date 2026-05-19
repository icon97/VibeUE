# Iconic API Integration Notes for VibeUE

## Mục tiêu

Muốn fork/plugin VibeUE không phụ thuộc VibeUE cloud/token nữa. Toàn bộ tính năng cần chuyển sang backend riêng:

```text
https://api.iconic.io.vn
```

Người dùng chỉ cần nhập token hợp lệ của Iconic là dùng được plugin.

Plugin Unreal/VibeUE cần backend Iconic hỗ trợ API tương thích OpenAI/OpenRouter/VibeUE ở mức đủ dùng cho:

- Chat LLM
- Model dropdown
- Context length / max output
- Tool calling
- MCP/internal tools validation
- Token counting fallback/optional
- Model ratings optional
- Terrain tools optional

## Kết quả test hiện tại

Đã test `https://api.iconic.io.vn` với token được cung cấp.

Các endpoint thử:

```text
GET  /v1/auth/validate
GET  /v1/models
POST /v1/chat/completions
POST /v1/tokenize
GET  /api/models/ratings
```

Kết quả: **tất cả trả HTTP 403**.

Sau đó probe thêm nhiều route/auth mode.

Routes:

```text
/v1/models
/api/v1/models
/openai/v1/models
/models
/api/models
/v1/auth/validate
/api/v1/auth/validate
/auth/validate
```

Auth modes:

```text
Authorization: Bearer <token>
X-API-Key: <token>
Cả Authorization Bearer + X-API-Key
```

Kết quả tất cả vẫn:

```text
HTTP 403
Cloudflare Error 1010: Access denied
```

Response có dạng Cloudflare:

```json
{
  "type": "https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1010/",
  "title": "Error 1010: Access denied",
  "status": 403
}
```

## Kết luận test

Hiện request bị **Cloudflare chặn trước khi tới backend app**, không phải lỗi schema API hay sai route đơn thuần.

Unreal plugin cũng rất có khả năng bị chặn tương tự vì Unreal HTTP client là non-browser/API client, không xử lý browser challenge/cookie/JS challenge của Cloudflare được.

## Việc backend/Cloudflare cần xử lý trước

Trong Cloudflare, kiểm tra Security Events cho domain:

```text
api.iconic.io.vn
```

Tìm event:

```text
Error 1010
Access denied
```

Có thể do một trong các rule:

- Browser Integrity Check
- Bot Fight Mode / Super Bot Fight Mode
- WAF custom rule
- IP reputation
- Country/IP allow/block
- Rule chặn non-browser User-Agent/API client
- Require browser challenge trên API route

Cần cấu hình bypass/allow cho API paths:

```text
api.iconic.io.vn/v1/*
api.iconic.io.vn/api/*
```

Nên allow:

```text
GET
POST
OPTIONS
```

Nên allow non-browser clients khi có một trong các header:

```text
Authorization: Bearer <token>
X-API-Key: <token>
```

Không nên require JS/browser challenge cho API endpoint vì Unreal HTTP client không chạy được challenge đó.

## Contract backend Iconic cần hỗ trợ

### 1. Auth validate — bắt buộc cho MCP/tools

```http
GET /v1/auth/validate
Authorization: Bearer <token>
X-API-Key: <token>
```

Backend nên accept cả hai kiểu header. Plugin sẽ ưu tiên Bearer nhưng có thể gửi thêm `X-API-Key` để tương thích.

Expected response:

```json
{
  "ok": true
}
```

Hoặc:

```json
{
  "valid": true
}
```

Yêu cầu tối thiểu:

```text
HTTP 200 khi token hợp lệ
HTTP 401/403 khi token sai
```

### 2. Models — bắt buộc cho model dropdown/context/max output

```http
GET /v1/models
Authorization: Bearer <token>
```

Expected OpenRouter-style response:

```json
{
  "data": [
    {
      "id": "cx/gpt-5.4",
      "name": "GPT 5.4",
      "context_length": 1050000,
      "supported_parameters": [
        "tools",
        "tool_choice",
        "parallel_tool_calls",
        "max_tokens",
        "temperature",
        "top_p"
      ],
      "top_provider": {
        "max_completion_tokens": 128000
      },
      "pricing": {
        "prompt": "0",
        "completion": "0"
      }
    }
  ]
}
```

Fields quan trọng:

| Field | Mức độ | Ghi chú |
|---|---:|---|
| `data` array | Bắt buộc | Plugin parse model list từ đây |
| `id` | Bắt buộc | Model id gửi vào chat request |
| `name` | Nên có | Hiển thị UI |
| `context_length` | Rất nên có | Dùng tính context budget/truncate |
| `supported_parameters` | Rất nên có | Xác định model/tool support |
| `top_provider.max_completion_tokens` | Rất nên có | Dùng cho max output/token UI/logic |
| `pricing.prompt/completion` | Optional | UI hiển thị free/price |

Nếu dùng model `cx/gpt-5.4`, nên trả:

```json
{
  "id": "cx/gpt-5.4",
  "name": "GPT 5.4",
  "context_length": 1050000,
  "top_provider": {
    "max_completion_tokens": 128000
  },
  "supported_parameters": [
    "tools",
    "tool_choice",
    "parallel_tool_calls",
    "max_tokens",
    "temperature",
    "top_p"
  ]
}
```

### 3. Chat completions — bắt buộc

```http
POST /v1/chat/completions
Authorization: Bearer <token>
Content-Type: application/json
```

Plugin có thể gửi body dạng:

```json
{
  "model": "cx/gpt-5.4",
  "messages": [
    {
      "role": "system",
      "content": "..."
    },
    {
      "role": "user",
      "content": "..."
    }
  ],
  "stream": true,
  "temperature": 0.2,
  "top_p": 0.95,
  "max_tokens": 128000,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "tool_name",
        "description": "...",
        "parameters": {}
      }
    }
  ],
  "parallel_tool_calls": true
}
```

Backend cần hỗ trợ:

- `stream: true` SSE format OpenAI-compatible.
- `stream: false` non-stream response.
- Tool calling format OpenAI-compatible.
- Tool result messages.
- `temperature`, `top_p`, `max_tokens`.
- Multimodal image content nếu muốn dùng tính năng gửi ảnh.

Non-stream response expected:

```json
{
  "id": "chatcmpl-...",
  "model": "cx/gpt-5.4",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 123,
    "completion_tokens": 456,
    "total_tokens": 579
  }
}
```

Streaming response expected:

```text
data: {"choices":[{"delta":{"content":"hello"},"finish_reason":null}]}

data: {"choices":[{"delta":{"content":" world"},"finish_reason":null}]}

data: {"choices":[{"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

Tool call streaming nên theo OpenAI-compatible format.

### 4. Tokenize — optional nhưng nên có

```http
POST /v1/tokenize
Authorization: Bearer <token>
Content-Type: application/json
```

Plugin cần token counting cho context estimate chính xác.

Có thể support text:

```json
{
  "text": "hello world"
}
```

Và messages:

```json
{
  "model": "cx/gpt-5.4",
  "messages": [
    {
      "role": "user",
      "content": "hello world"
    }
  ]
}
```

Expected response:

```json
{
  "token_count": 2
}
```

Có thể alias được:

```json
{
  "tokens": 2
}
```

Hoặc:

```json
{
  "count": 2
}
```

Nếu chưa có `/v1/tokenize`, plugin có thể fallback heuristic local, nhưng context indicator/summarization sẽ kém chính xác hơn.

### 5. Model ratings — optional

Hiện VibeUE gọi:

```http
GET /api/models/ratings
```

Expected:

```json
{
  "ratings": {
    "cx/gpt-5.4": "best",
    "some/model": "good"
  }
}
```

Nếu backend chưa có, plugin có thể skip ratings, không ảnh hưởng chat.

### 6. Terrain APIs — optional nếu muốn đủ toàn bộ terrain tools

VibeUE hiện có terrain/map tools như:

```text
generate_heightmap
preview_elevation
get_map_image
list_styles
get_water_features
```

Hiện code cũ gọi VibeUE terrain API qua:

```text
https://www.vibeue.com
```

và header:

```text
X-API-Key: <VibeUEApiKey>
```

Nếu muốn toàn bộ tính năng terrain chạy bằng Iconic token, backend Iconic cần cung cấp endpoint tương đương cho các action trên.

Nếu chưa có, plugin nên trả lỗi rõ:

```json
{
  "success": false,
  "error": "FEATURE_UNAVAILABLE",
  "message": "Iconic terrain API is not available on this backend yet."
}
```

## Hướng sửa plugin sau khi backend pass

Sau khi Cloudflare/API pass, plugin nên sửa theo hướng:

### Config mới

```ini
[VibeUE]
Provider=Iconic
IconicBaseUrl="https://api.iconic.io.vn"
IconicApiKey="..."
IconicLastModel="cx/gpt-5.4"
MaxTokens=128000
```

API key vẫn chỉ lưu local trong:

```text
Saved/Config/WindowsEditor/EditorPerProjectUserSettings.ini
```

Không commit key.

### Auth behavior

Plugin gửi:

```http
Authorization: Bearer <IconicApiKey>
X-API-Key: <IconicApiKey>
```

Backend nên accept cả hai, nhưng chuẩn chính là Bearer.

### Thay các hard-code VibeUE cloud URL

Cần thay hoặc route qua Iconic config:

```text
https://llm.vibeue.com/v1/chat/completions
https://llm.vibeue.com/v1/auth/validate
https://www.vibeue.com/api/models/ratings
https://www.vibeue.com terrain APIs
https://www.vibeue.com/login UI links
```

### Gating/token

Hiện plugin có nhiều chỗ bắt `VibeUEApiKey`. Cần đổi thành `IconicApiKey` cho bản Iconic:

- Chat provider
- MCP validation
- Terrain tools
- UI settings
- Error messages
- Login/help buttons

Local/internal tools không nên bị khóa bởi VibeUE token nữa; chỉ cần token Iconic khi tính năng đó gọi backend Iconic.

## Test acceptance sau khi backend fix

Chạy lại các test sau:

```text
1. GET /v1/auth/validate
   Expect: HTTP 200 + ok/valid true

2. GET /v1/models
   Expect: HTTP 200 + data[] non-empty
   First model has id, name, context_length, supported_parameters
   Prefer top_provider.max_completion_tokens

3. POST /v1/chat/completions stream=false
   Expect: HTTP 200 + choices[0].message.content

4. POST /v1/chat/completions stream=true
   Expect: SSE data chunks + [DONE]

5. POST /v1/chat/completions with tools
   Expect: tool_calls emitted in OpenAI-compatible format

6. POST /v1/tokenize
   Optional: HTTP 200 + token_count/tokens/count

7. GET /api/models/ratings
   Optional: HTTP 200 + ratings object

8. Terrain endpoints
   Optional: only required if Iconic wants terrain tools fully working
```

## Current blocker

Cloudflare currently blocks all tested API requests with:

```text
HTTP 403
Error 1010: Access denied
```

This must be fixed before plugin integration can be validated.
