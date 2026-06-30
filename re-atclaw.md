# AutoClaw

> **STATUS: Fully Live-Verified** — Auth, LLM proxy, billing, tool-calling, multi-turn,
> dan agentic file/shell loop semuanya sudah diuji terhadap production backend asli.

---

## 1. Auth

### Hosts

| Env | Host |
|-----|------|
| prod / pre | `https://autoglm-api.autoglm.ai` |
| dev / test | `https://autoglm-test-api.autoglm.ai` |
| legacy | `autoglm-api.zhipuai.cn` — LLM proxy-nya 401 "Invalid token" (**DEAD** untuk token ini) |

### App-Signing Headers

> Header ini *forgeable* — mengonfirmasi build, bukan user.

```
X-Auth-Appid:     100003
X-Auth-TimeStamp: <unix_seconds>
X-Auth-Sign:      md5(`${APP_ID}&${ts}&${APP_KEY}`)
                  APP_KEY = 38d2391985e2369a5fb8227d8e6cd5e5  ← baked secret
X-Product:        autoclaw
X-Version:        1.9.1
X-Tm:             win
X-Trace-Id:       <uuid>
```

### Google OAuth (Overseas)

```
POST /userapi/overseasv1/google-oauth-url
  Body: { source_id: "autoclaw", device_id, navigate_uri }
  Response: { oauth_url, state }

  Google client_id: 1070296600523-gjj3c53agiq47m32ad4juolgfae88tdi.apps.googleusercontent.com
  Redirect:         http://localhost:18432/auth/callback-google  (local token-server)
  Scope:            "openid email profile"
  response_type:    code

  (User membuka oauth_url, login, paste balik localhost:18432/...?code=...&state=...)

POST /userapi/overseasv1/google-oauth-login
  Body: { code, state, navigate_uri }
  Response: { access_token: "Bearer eyJ…", refresh_token: "Bearer eyJ…", user_id, user_name, first_login }
```

**Alternatif login:**
- Phone: `POST /userapi/v1/agent-send-code` → `POST /userapi/v1/agent-login/`
- Zai (Z.ai SSO): `/userapi/overseasv1/zai-oauth-url` + `.../zai-oauth-login`

### Token Details

- **access_token** — JWT HS256, payload `{ user_id, device_id, source_id: "autoclawaccess_token", jti: <email>, exp }`, **TTL = 24 jam** (86400 detik).
- **refresh_token** — TTL ~30 hari.
- **Refresh:** `POST /userapi/v1/refresh { source_id, device_id, refresh_token }`
- **Identity check:** `POST /userapi/v1/user-profile` 

### ⚠️ Header 

- **LLM proxy** → gunakan `X-Authorization: Bearer …`
- **assetmgr (wallet/ledger)** → gunakan `authorization: Bearer …` *(lowercase)*

Salah satu → assetmgr kembalikan `410000 "Please log in."`

---

## 2. LLM Proxy Contract

```
POST https://autoglm-api.autoglm.ai/autoclaw-proxy/proxy/autoclaw/chat/completions

Headers:
  X-Authorization: Bearer <access_token>
  X-Request-Id:    <uuid>
  X-Request-Model: <zai_*>        ← MODEL SELECTOR (field "model" di body DIABAIKAN)

Body: OpenAI chat-completions shape (field "model" bisa diisi apa saja, misal "x")
```

### Quirks Penting

1. **Model ditentukan oleh header `X-Request-Model`, bukan JSON body.** Jika tidak ada → `400 code 1211 "模型不存在"`.
2. **DeepSeek-backed labels akan 500 `{"message":"parse response failed"}` jika `stream: false`**, tapi lancar di stream. → Selalu request `stream: true` ke upstream; aggregate untuk client non-stream.
3. **Model substitution nyata** (label yang diiklankan ≠ model yang di-serve):

| X-Request-Model | Model yang Di-serve | Self-ID | Catatan |
|-----------------|---------------------|---------|---------|
| `zai_auto` | **deepseek-v4-pro** | "I'm DeepSeek" | "Auto" route ke DeepSeek, bukan GLM. ~7× lebih mahal |
| `zaicoding_glm-5.2` | **deepseek-v4-pro** | "I'm DeepSeek-V3" | Label berbohong — disubstitusi |
| `openrouter_glm-5.2` | **z-ai/glm-5.2-20260616** | "GLM by Z.ai" | **Satu-satunya GLM-5.2 asli — model terbaik** |
| `zai_glm-5-turbo` / `zai_glm-5v-turbo` / `zai_glm-5.1` / `zai_pony-alpha-2` | **glm-5-turbo** | "GLM by Z.ai" | Termurah (-1 pt) |
| `zai_glm-4.x`, `zai_glm-5`, `zai_glm-5.2`, `huawei_glm-5` | 400 / 404 | — | Tidak terprovisikan untuk akun ini |

### Billing (Diverifikasi via Ledger)

- **Wallet:** `GET /agent-assetmgr/api/v2/wallets?biz_app_id=autoclaw` → `total_balance` (reward points).
  Dimulai ~2300; setelah sesi test berakhir di 2234.
- **Ledger:** `GET /agent-assetmgr/api/v1/ledgers_std?asset_type=point&wallet_type=all`
  Menampilkan debit per-call dengan metadata lengkap:
  - glm-5-turbo → `amount: -1`, `token_usage: "input:17, output:269"`, model `glm-5-turbo`
  - auto call → `amount: -7`, `output: 2052`, model `deepseek-v4-pro`
  - `wallet_instance_balance_before/after` tercatat di setiap entri.
- **Biaya** ≈ volume output-token. glm-5-turbo paling murah; zai_auto / DeepSeek paling mahal.

---

## 3. Konstanta & Secret yang Hardcoded

| Konstanta | Nilai |
|-----------|-------|
| App signing | `APP_KEY = 38d2391985e2369a5fb8227d8e6cd5e5`, `APP_ID / X-Auth-Appid = 100003` |
| LLM proxy base | `https://autoglm-api.autoglm.ai/autoclaw-proxy/proxy/autoclaw` |
| User API base | `https://autoglm-api.autoglm.ai` |
| Google OAuth client_id | `1070296600523-gjj3c53agiq47m32ad4juolgfae88tdi.apps.googleusercontent.com` |
| OAuth redirect | `http://localhost:18432/auth/callback-google` |
| Local gateway WS | `ws://127.0.0.1:18789` (proto v3, ed25519 connect) |
| Sandbox identity | `http://127.0.0.1:29001/.__sandbox_harness__/identity` (overseas: disabled) |
| Safety approval ports | `18432, 19654, 19723, 53699` |
| Token file (gateway reads) | `~/.openclaw-autoclaw/request-headers.json` |
| State dir / profile | `~/.openclaw-autoclaw/` (env: `OPENCLAW_PROFILE=autoclaw`) |
| Telemetry | ByteDance DataRanger + APMPlus (`tab.ab.ap-southeast-1.volces.com`), Tencent IM |

---

## 4. Rekomendasi Model

| Model | Keterangan |
|-------|------------|
| **`openrouter_glm-5.2`** ✅ | **Terbaik overall** — GLM-5.2 asli oleh Z.ai. Tool-call bersih, markdown bagus, self-ID jujur, agentic chaining berjalan. |
| `zai_glm-5-turbo` | **Termurah** (-1 pt/call). |
| `zai_auto` | **Hindari** — diam-diam menggunakan DeepSeek-V4-Pro, output reasoning-token besar (~7× lebih mahal). |

> Ini adalah backend chat OpenAI-compatible murni (bukan agentic-sandbox backend) —
> cocok untuk di-proxy secara bersih. Layer agentic/sandbox/file hanya ada di desktop gateway,
> yang di-bypass oleh proxy ini.
