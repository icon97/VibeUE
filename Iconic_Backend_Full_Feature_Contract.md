# Iconic Backend Full Feature Contract

Tài liệu này là hợp đồng triển khai backend cho fork Iconic của VibeUE, nhằm đạt full parity với backend VibeUE gốc. Base URL chuẩn:

```text
https://api.iconic.io.vn
```

Ghi chú tương thích: trong plugin, nhiều tên class/config/log/UI vẫn có thể là `VibeUE` để tránh phá vỡ compatibility. Backend triển khai, domain, auth và vận hành nên là Iconic.

## 1) Overview: plugin kỳ vọng gì từ backend

Plugin cần backend cung cấp:

1. Chat backend OpenAI-compatible/OpenRouter-compatible: validate API key, list models, chat completions, token counting.
2. Tool calling backend: nhận `tools`, `tool_choice`, `parallel_tool_calls`; trả `tool_calls`; chấp nhận follow-up `role: "tool"` messages.
3. Model metadata: `context_length`, thông tin tool support, optional pricing/rating/capabilities.
4. Terrain API: styles, elevation preview, heightmap bytes, map image bytes, water features JSON.

Endpoint chat thường cấu hình là:

```text
https://api.iconic.io.vn/v1/chat/completions
```

Plugin suy ra từ URL trên:

```text
https://api.iconic.io.vn/v1/models
https://api.iconic.io.vn/v1/tokenize
```

Terrain base URL nên là:

```text
https://api.iconic.io.vn
```

## 2) Auth conventions: Authorization Bearer và X-API-Key

Backend nên chấp nhận cả hai header trên mọi endpoint cần auth:

```http
Authorization: Bearer $ICONIC_API_KEY
X-API-Key: $ICONIC_API_KEY
```

Plugin hiện gửi chủ yếu `X-API-Key`. MCP/curl/client tương lai có thể dùng `Authorization: Bearer`.

Quy ước đề xuất:

- Nếu thiếu key ở endpoint cần auth: `401`.
- Nếu key sai/hết hạn: `401`.
- Nếu key hợp lệ nhưng không có quyền: `403`.
- Nếu hết quota/rate limit: `429`.
- Nếu cả `Authorization` và `X-API-Key` cùng có mặt nhưng khác nhau: trả `401` để tránh ambiguity.
- `/v1/tokenize` nên cho phép không auth để đúng hành vi plugin hiện tại.
- `/api/terrain/styles` nên public hoặc auth optional vì plugin gọi không kèm key.

## 3) Required endpoints summary

| Priority | Method | Endpoint | Mục đích | Auth |
|---|---:|---|---|---|
| Core required | GET | `/v1/auth/validate` | Validate API key cho MCP/tools | Có |
| Core required | POST | `/v1/auth/validate` | Validate API key cho client/curl | Có |
| Core required | GET | `/v1/models` | Model dropdown, context length, tool support | Optional/khuyến nghị có |
| Core required | POST | `/v1/chat/completions` | Chat non-stream | Có |
| Optional full parity | POST | `/v1/chat/completions` stream/SSE | Streaming nếu bật về sau | Có |
| Core required | POST | `/v1/tokenize` | Đếm token text/messages | Không bắt buộc |
| Optional full parity | GET | `/api/models/ratings` | Rating model cho UI | Optional |
| Optional full parity | GET | `/api/terrain/styles` | Danh sách map styles | Không bắt buộc |
| Optional full parity | POST | `/api/terrain/preview` | Elevation stats và suggested settings | Có |
| Optional full parity | POST | `/api/terrain/heightmap` | Generate heightmap | Có |
| Optional full parity | POST | `/api/terrain/map-image` | Generate map image | Có |
| Optional full parity | POST | `/api/terrain/water-features` | Rivers/lakes/ocean geometry | Có |

## 4) Detailed endpoint specs

### 4.1 GET/POST `/v1/auth/validate`

Request:

```bash
curl -i https://api.iconic.io.vn/v1/auth/validate \
  -H "X-API-Key: $ICONIC_API_KEY"
```

POST có thể nhận body rỗng hoặc:

```json
{ "api_key": "$ICONIC_API_KEY" }
```

Response `200` khuyến nghị:

```json
{
  "valid": true,
  "user_id": "usr_123",
  "plan": "free",
  "quota": {
    "chat_requests_remaining": 1000,
    "terrain_requests_remaining": 100
  }
}
```

Tối thiểu plugin chỉ cần HTTP `200` để coi key hợp lệ.

### 4.2 GET `/v1/models`

Response root phải có `data` array không rỗng.

```json
{
  "object": "list",
  "data": [
    {
      "id": "iconic/auto",
      "name": "Iconic Auto Router",
      "context_length": 131072,
      "supported_parameters": ["tools", "tool_choice", "parallel_tool_calls", "temperature", "top_p", "max_tokens"],
      "capabilities": {
        "tool_calling": true,
        "vision": true,
        "streaming": false,
        "reasoning": true
      },
      "pricing": { "prompt": "0", "completion": "0" }
    }
  ]
}
```

Yêu cầu:

- `id`: string, bắt buộc.
- `name`: string, khuyến nghị.
- `context_length`: integer > 0, bắt buộc để UI context budget chính xác.
- Tool support nên khai báo bằng cả hai kiểu:
  - `supported_parameters` chứa `tools` và nên chứa `tool_choice`, `parallel_tool_calls`.
  - `capabilities.tool_calling: true`.
- `pricing.prompt` và `pricing.completion` optional, string giá/token kiểu OpenRouter; plugin hiển thị quy đổi per 1M token.

Curl:

```bash
curl -s https://api.iconic.io.vn/v1/models \
  -H "X-API-Key: $ICONIC_API_KEY"
```

### 4.3 POST `/v1/chat/completions` non-stream

Plugin VibeUE client hiện gửi `stream: false` để tránh race condition SSE trong Unreal HTTP.

Request ví dụ:

```json
{
  "model": "iconic/auto",
  "messages": [
    { "role": "system", "content": "You are an Unreal Engine assistant." },
    { "role": "user", "content": "Create a cube." }
  ],
  "stream": false,
  "temperature": 0.7,
  "top_p": 1.0,
  "max_tokens": 4096,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "actor",
        "description": "Create and modify actors",
        "parameters": {
          "type": "object",
          "properties": {
            "action": { "type": "string", "description": "Action name" }
          },
          "required": ["action"]
        }
      }
    }
  ],
  "parallel_tool_calls": false
}
```

Response text-only:

```json
{
  "id": "chatcmpl_abc",
  "object": "chat.completion",
  "created": 1779148800,
  "model": "iconic/auto",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Tôi sẽ tạo một cube trong scene."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 100,
    "completion_tokens": 30,
    "total_tokens": 130
  }
}
```

Response tool call:

```json
{
  "id": "chatcmpl_tool_1",
  "object": "chat.completion",
  "created": 1779148800,
  "model": "iconic/auto",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_001",
            "type": "function",
            "function": {
              "name": "actor",
              "arguments": "{\"action\":\"create\",\"class\":\"StaticMeshActor\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ],
  "usage": { "prompt_tokens": 1200, "completion_tokens": 50, "total_tokens": 1250 }
}
```

Plugin cũng đọc các trường reasoning nếu có:

```json
{
  "message": {
    "role": "assistant",
    "content": "...",
    "reasoning": "hidden/summary reasoning",
    "reasoning_content": "hidden/summary reasoning",
    "reasoning_details": []
  }
}
```

Nếu backend route sang provider thực tế khác, hãy set top-level `model` trong response để plugin hiển thị `via <model>`.

Curl:

```bash
curl -s https://api.iconic.io.vn/v1/chat/completions \
  -H "Authorization: Bearer $ICONIC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"iconic/auto","messages":[{"role":"user","content":"Say hi"}],"stream":false}'
```

### 4.4 POST `/v1/chat/completions` stream/SSE

Plugin VibeUE path hiện tắt stream, nhưng để full parity với OpenAI-compatible clients nên hỗ trợ SSE khi request `stream: true`.

Headers:

```http
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

Chunk ví dụ:

```text
data: {"id":"chatcmpl_abc","object":"chat.completion.chunk","model":"iconic/auto","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl_abc","object":"chat.completion.chunk","model":"iconic/auto","choices":[{"index":0,"delta":{"content":"Xin chào"},"finish_reason":null}]}

data: {"id":"chatcmpl_abc","object":"chat.completion.chunk","model":"iconic/auto","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

Tool-call streaming nên theo OpenAI delta format (`delta.tool_calls[].index/id/type/function.name/function.arguments`).

### 4.5 POST `/v1/tokenize`

Plugin có hai dạng body.

Text body:

```json
{ "text": "Hello Unreal" }
```

Messages body:

```json
{
  "model": "iconic/auto",
  "messages": [
    { "role": "user", "content": "Hello" },
    {
      "role": "assistant",
      "tool_calls": [
        { "id": "call_1", "type": "function", "function": { "name": "actor", "arguments": "{}" } }
      ]
    },
    { "role": "tool", "tool_call_id": "call_1", "content": "{\"success\":true}" }
  ]
}
```

Response:

```json
{ "token_count": 42 }
```

### 4.6 GET `/api/models/ratings`

Optional cho UI/sorting/full parity.

Response dạng mapping được khuyến nghị:

```json
{
  "ratings": {
    "iconic/auto": "great",
    "openai/gpt-4.1": "good"
  }
}
```

Giá trị rating plugin hiểu: `great`, `good`, `moderate`, `bad`, hoặc rỗng/unrated.

### 4.7 GET `/api/terrain/styles`

Plugin gọi GET không auth. Response nên là JSON:

```json
{
  "styles": [
    { "id": "satellite-v9", "name": "Satellite", "description": "Aerial imagery" },
    { "id": "outdoors-v11", "name": "Outdoors", "description": "Topo/trails/contours" },
    { "id": "streets-v11", "name": "Streets" },
    { "id": "light-v10", "name": "Light" },
    { "id": "dark-v10", "name": "Dark" }
  ]
}
```

### 4.8 POST `/api/terrain/preview`

Request body từ `TerrainDataTools.cpp`:

```json
{
  "lng": 138.7274,
  "lat": 35.3606,
  "map_size": 17.28
}
```

Response JSON khuyến nghị:

```json
{
  "min_height": 340.0,
  "max_height": 3776.0,
  "height_range": 3436.0,
  "suggested_base_level": 340,
  "suggested_height_scale": 27,
  "suggestedZScale": 741,
  "suggestedXYScales": {
    "505": 3429,
    "1009": 1714,
    "2017": 857,
    "4033": 429,
    "8129": 213
  },
  "tile_zoom": 13,
  "tile_count": 9
}
```

Plugin pass-through JSON này trực tiếp cho AI.

### 4.9 POST `/api/terrain/heightmap`

Request body từ plugin:

```json
{
  "lng": 138.7274,
  "lat": 35.3606,
  "format": "png",
  "map_size": 17.28,
  "base_level": 340,
  "height_scale": 27,
  "water_depth": 40,
  "gravity_center": 0,
  "level_correction": 0,
  "blur_passes": 10,
  "blur_post_passes": 2,
  "sharpen": true,
  "draw_streams": true,
  "stream_depth": 7,
  "plains_height": 140,
  "resolution": 1009
}
```

Response thành công là bytes, không phải JSON:

- `format=png`: `Content-Type: image/png`, khuyến nghị 16-bit grayscale PNG.
- `format=raw`: `Content-Type: application/octet-stream`.
- `format=zip`: `Content-Type: application/zip`.

Headers plugin đọc:

```http
X-Heightmap-Min-Height: 340.0
X-Heightmap-Max-Height: 3776.0
X-Heightmap-Size: 1009x1009
```

Plugin lưu bytes ra file và tự tạo JSON success local gồm `file`, `format`, `size_bytes`, `min_height_m`, `max_height_m`, `dimensions`.

### 4.10 POST `/api/terrain/map-image`

Request:

```json
{
  "lng": -122.4194,
  "lat": 37.7749,
  "map_size": 17.28,
  "style": "satellite-v9",
  "width": 1280,
  "height": 1280
}
```

Response thành công là image bytes:

```http
Content-Type: image/png
```

Plugin lưu bytes ra `map_<style>_<lat>_<lng>.png`.

### 4.11 POST `/api/terrain/water-features`

Request:

```json
{
  "lng": -105.0,
  "lat": 39.7,
  "map_size": 17.28
}
```

Response JSON backend cần trả tối thiểu:

```json
{
  "waterways": [
    {
      "name": "Clear Creek",
      "class": "river",
      "estimated_width_m": 12,
      "points": [
        { "lng": -105.01, "lat": 39.70 },
        { "lng": -105.00, "lat": 39.71 }
      ]
    }
  ],
  "water_bodies": [
    {
      "name": "Example Lake",
      "class": "lake",
      "area_sq_m": 12345,
      "rings": [
        [
          { "lng": -105.02, "lat": 39.70 },
          { "lng": -105.01, "lat": 39.70 },
          { "lng": -105.02, "lat": 39.70 }
        ]
      ]
    }
  ]
}
```

Plugin sẽ tự thêm `ue5_points` và `ue5_rings` bằng quy ước `+X=East`, `+Y=North`, map center ở origin.

## 5) Tool calling requirements

Backend phải hỗ trợ chuẩn OpenAI-style tools:

Request fields:

- `tools`: array of `{ "type": "function", "function": { "name", "description", "parameters" } }`.
- `tool_choice`: nếu client gửi, hỗ trợ `auto`, `none`, `required`, hoặc specific function.
- `parallel_tool_calls`: boolean. Khi `false`, model nên trả tối đa một tool call mỗi turn.

Assistant response tool call bắt buộc đúng format:

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_unique_id",
      "type": "function",
      "function": {
        "name": "terrain_data",
        "arguments": "{\"action\":\"preview_elevation\",\"lng\":138.7274,\"lat\":35.3606}"
      }
    }
  ]
}
```

Follow-up từ plugin sau khi execute tool:

```json
{
  "role": "tool",
  "tool_call_id": "call_unique_id",
  "content": "{\"success\":true,\"file\":\"C:/Project/Saved/Terrain/x.png\"}"
}
```

Backend phải chấp nhận conversation tiếp theo có assistant message chứa `tool_calls` và tool messages tương ứng. `function.arguments` phải là JSON string hợp lệ, không phải object.

## 6) Model metadata requirements

Mỗi model trong `/v1/models` nên có:

```json
{
  "id": "iconic/auto",
  "name": "Iconic Auto Router",
  "context_length": 131072,
  "supported_parameters": ["tools", "tool_choice", "parallel_tool_calls", "temperature", "top_p", "max_tokens"],
  "capabilities": {
    "tool_calling": true,
    "vision": true,
    "streaming": false,
    "reasoning": true
  },
  "pricing": { "prompt": "0", "completion": "0" }
}
```

Plugin variants đang hỗ trợ:

- OpenRouter-style: đọc `supported_parameters` để biết `tools`.
- VibeUE worker-style: đọc `capabilities.tool_calling`.
- Pricing optional: nếu `0` hoặc thiếu, UI coi là free.
- `context_length` rất quan trọng cho token budget.

## 7) Terrain contract details

Request bodies được liệt kê ở mục 4.8-4.11 là lấy từ `TerrainDataTools.cpp`.

Response/file requirements:

- `/api/terrain/heightmap`: binary bytes; plugin không parse JSON khi `200`. Với lỗi non-200 có thể trả JSON/text error body.
- Heightmap headers nên gồm `X-Heightmap-Min-Height`, `X-Heightmap-Max-Height`, `X-Heightmap-Size`.
- `/api/terrain/map-image`: image bytes; plugin lưu file PNG theo content bytes.
- `/api/terrain/preview`: JSON pass-through, nên có `suggestedZScale` và `suggestedXYScales`.
- `/api/terrain/water-features`: JSON full geometry với `waterways[].points` và `water_bodies[].rings`; plugin thêm tọa độ UE5 và lưu JSON local.
- Timeout plugin: POST terrain thường 30s; water-features 45s. Backend nên tối ưu hoặc trả lỗi rõ khi quá tải.

## 8) Error handling contract

Backend nên trả JSON error thống nhất:

```json
{
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Daily terrain quota exceeded",
    "type": "quota_error",
    "details": {
      "reset_at": "2026-05-20T00:00:00Z"
    }
  }
}
```

Plugin có thể display `error.message`, hoặc body string nếu terrain endpoint non-200.

Status codes đề xuất:

- `400`: request JSON sai, thiếu `lat/lng`, model không hợp lệ.
- `401`: thiếu/sai API key.
- `403`: không có quyền model/terrain.
- `404`: endpoint/model/style không tồn tại.
- `405`: method sai. Lưu ý `/api/terrain/preview` phải hỗ trợ POST.
- `408`/`504`: timeout upstream.
- `413`: request quá lớn.
- `429`: quota/rate limit.
- `500`/`502`/`503`: lỗi server/upstream.

## 9) Current tested status summary

Tình trạng test hiện tại của Iconic fork:

- Core chat/MCP đã pass.
- Terrain đang fail:
  - `/api/terrain/styles` trả `HTTP_ERROR`.
  - `/api/terrain/preview` trả `HTTP_405`.
- Backend cần triển khai đúng terrain endpoints và method, đặc biệt GET `/api/terrain/styles` và POST `/api/terrain/preview`.

## 10) Verification checklist sau khi backend triển khai

### Auth

```bash
curl -i https://api.iconic.io.vn/v1/auth/validate \
  -H "X-API-Key: $ICONIC_API_KEY"
```

Pass khi status `200`.

### Models

```bash
curl -s https://api.iconic.io.vn/v1/models \
  -H "X-API-Key: $ICONIC_API_KEY"
```

Checklist: có `data[0].id`, `context_length > 0`, tool support metadata.

### Chat non-stream

```bash
curl -s https://api.iconic.io.vn/v1/chat/completions \
  -H "Authorization: Bearer $ICONIC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"iconic/auto","messages":[{"role":"user","content":"Reply with one sentence."}],"stream":false}'
```

Checklist: có `choices[0].message.content` hoặc `choices[0].message.tool_calls`.

### Tool calling

```bash
curl -s https://api.iconic.io.vn/v1/chat/completions \
  -H "X-API-Key: $ICONIC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"iconic/auto","stream":false,"messages":[{"role":"user","content":"Call the test tool with x=1"}],"tools":[{"type":"function","function":{"name":"test_tool","description":"A test tool","parameters":{"type":"object","properties":{"x":{"type":"number"}},"required":["x"]}}}],"parallel_tool_calls":false}'
```

Checklist: nếu model chọn tool, response có `tool_calls[].function.arguments` là JSON string hợp lệ.

### Tokenize

```bash
curl -s https://api.iconic.io.vn/v1/tokenize \
  -H "Content-Type: application/json" \
  -d '{"text":"Hello Unreal Engine"}'
```

Checklist: `{ "token_count": <number> }`.

### Terrain styles

```bash
curl -i https://api.iconic.io.vn/api/terrain/styles
```

Checklist: status `200`, JSON có styles hoặc list tương đương.

### Terrain preview

```bash
curl -s https://api.iconic.io.vn/api/terrain/preview \
  -H "X-API-Key: $ICONIC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"lng":138.7274,"lat":35.3606,"map_size":17.28}'
```

Checklist: có `min_height`, `max_height`, `suggestedZScale`, `suggestedXYScales`.

### Heightmap bytes

```bash
curl -i -o heightmap.png https://api.iconic.io.vn/api/terrain/heightmap \
  -H "X-API-Key: $ICONIC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"lng":138.7274,"lat":35.3606,"format":"png","map_size":17.28,"base_level":340,"height_scale":27,"resolution":1009}'
```

Checklist: `Content-Type: image/png`, file non-empty, headers `X-Heightmap-*` có giá trị.

### Map image bytes

```bash
curl -i -o map.png https://api.iconic.io.vn/api/terrain/map-image \
  -H "X-API-Key: $ICONIC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"lng":-122.4194,"lat":37.7749,"map_size":17.28,"style":"satellite-v9","width":1280,"height":1280}'
```

Checklist: image file non-empty.

### Water features

```bash
curl -s https://api.iconic.io.vn/api/terrain/water-features \
  -H "X-API-Key: $ICONIC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"lng":-105.0,"lat":39.7,"map_size":17.28}'
```

Checklist: JSON có `waterways` array và `water_bodies` array.

### MCP/plugin end-to-end

1. Cấu hình plugin VibeUE/Iconic chat endpoint là `https://api.iconic.io.vn/v1/chat/completions`.
2. Cấu hình API key Iconic.
3. Start Unreal Editor và MCP server.
4. Gửi prompt chat đơn giản, xác nhận model response.
5. Gửi prompt cần tool call, xác nhận plugin nhận `tool_calls`, execute tool, gửi follow-up tool message, và model trả final answer.
6. Chạy terrain skill: `terrain_data(action="list_styles")`, `terrain_data(action="preview_elevation", lng=138.7274, lat=35.3606)`, rồi `generate_heightmap` với `resolution=1009`.
