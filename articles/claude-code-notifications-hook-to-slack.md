---
title: "Claude Code の notifications hook で作業内容を良い感じに slack に送信する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["slack", "claude code"]
published: true
---
1. 以下を参考に incoming webhooks の設定をして webhook の URL をコピーする
https://api.slack.com/messaging/webhooks

2. ~/.claude/settings.json に以下の設定をする



```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "transcript_path=$(jq -r '.transcript_path') && jq -s '.[-3:]' \"$transcript_path\" | claude -p 'この会話の内容を日本語で簡潔に要約してください' | jq -Rs '{\"text\": .}' | curl -X POST -H 'Content-type: application/json' -d @- https://hooks.slack.com/services/XXXXXXXXXXXXXXXXXXXXXXX"
          }
        ]
      }
    ]
  }
}
```

- `CLAUDE_CONFIG_DIR` を設定している方はファイルパスを読み替えてください
- *webhook の URL は実際のものに置き換えてください*
