---
name: send-to-grok-bot
description: 把一項任務透過 webhook 交給使用者的 Grok Bot。使用方式：/send-to-grok-bot <任務>。當使用者要把任務轉交、派送、丟給 Grok Bot 時使用。
---

# send-to-grok-bot

把 `<任務>` 用 HTTP POST 送到 Grok Bot 的 webhook，並回報 HTTP 狀態碼。

## 憑證（不可寫進 repo）

Webhook URL 與 Authorization header 屬於機密，**只從環境變數讀取**，絕不寫入檔案、commit 或回覆內容：

- `GROK_BOT_WEBHOOK_URL`：webhook 完整網址
- `GROK_BOT_AUTH_HEADER`：完整一行，例如 `Authorization: Bearer xxxxx`

任一變數未設定時，不要猜，請使用者提供（或請他設定到雲端環境的環境變數），本次工作階段直接用使用者給的值即可，不要存檔。

## 步驟

1. **任務**：取 skill 參數（`/send-to-grok-bot` 後面的全部文字）。參數為空就先問使用者要送什麼任務。
2. **背景**：整理一段簡短的相關背景（目前 repo／分支、正在做的事、Grok Bot 執行任務需要知道的資訊）；沒有相關背景就用空字串。不要放入任何機密。
3. **組 JSON**：用 `jq` 產生 body，確保引號、換行、中文等正確跳脫，不要手動拼接字串：

   ```bash
   jq -n --arg task "$TASK" --arg context "$CONTEXT" \
     '{source: "Claude Code", task: $task, context: $context}' > "$SCRATCH/grok_body.json"
   ```

   （沒有 `jq` 時改用 `python3 -c 'import json,sys; print(json.dumps({"source":"Claude Code","task":sys.argv[1],"context":sys.argv[2]}, ensure_ascii=False))' "$TASK" "$CONTEXT"`。）

4. **送出**：

   ```bash
   curl -sS -o "$SCRATCH/grok_resp.txt" -w '%{http_code}' -X POST "$GROK_BOT_WEBHOOK_URL" \
     -H "$GROK_BOT_AUTH_HEADER" \
     -H 'Content-Type: application/json' \
     --data-binary @"$SCRATCH/grok_body.json"
   ```

   `$SCRATCH` 用工作階段的 scratchpad 目錄。執行時不要把憑證 echo 出來。

5. **回報**：告訴使用者 HTTP 狀態碼。2xx 就是成功；不是 2xx 就附上回應內容（`grok_resp.txt`）的重點，方便排查。curl 本身失敗（網路／proxy）時要照實說明，不要當成送出成功。
