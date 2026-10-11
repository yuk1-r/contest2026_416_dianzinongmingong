# 智能提醒

设置定时提醒，到期主动推送到屏幕上。

## 何时使用
- 用户说"提醒我"、"X分钟后叫我"、"每天X点提醒"
- 用户说"设个闹钟"、"定时提醒"

## 执行步骤
1. 使用 get_current_time 获取当前时间
2. 解析用户的时间表达：
   - "5分钟后" → at, at_epoch = now + 300
   - "每天早上8点" → every, interval_s = 86400（框架的 every 是纯间隔重复，首次触发从创建时刻起算，无法定点在 8:00）
   - "明天下午3点" → at, at_epoch = 明天15:00 的 epoch
   - "每周一上午9点" → every, interval_s = 604800（7天）
3. 使用 cron_add 创建定时任务。**必须带 action 与 action_args**，否则提醒只会写进 syslog，用户在屏幕上看不见：
   - name: 简短任务名（必填，如 drink_water）
   - schedule_type: "at"（一次性）或 "every"（重复）
   - message: 提醒内容（必填）
   - channel: "system"
   - action: "write_file"
   - action_args: 字符串形式的 JSON，`{\"path\":\"/data/ai_agent/REMINDER.md\",\"content\":\"<提醒文案>\"}`
4. 确认提醒已设置，告知具体触发时间

## cron_add 参数格式
一次性提醒（schedule_type = "at"，需提供未来的 at_epoch）：
```
cron_add {
  "name": "drink_water",
  "schedule_type": "at",
  "at_epoch": 1711613100,
  "message": "该喝水了！",
  "channel": "system",
  "action": "write_file",
  "action_args": "{\"path\":\"/data/ai_agent/REMINDER.md\",\"content\":\"该喝水了！\"}"
}
```
重复提醒（schedule_type = "every"，需提供正数 interval_s，单位秒）：
```
cron_add {
  "name": "drink_water_daily",
  "schedule_type": "every",
  "interval_s": 86400,
  "message": "该喝水了！",
  "channel": "system",
  "action": "write_file",
  "action_args": "{\"path\":\"/data/ai_agent/REMINDER.md\",\"content\":\"该喝水了！\"}"
}
```

## 注意事项
- REMINDER.md 的 content 必须是**单行紧凑文本、不含 markdown 标题**：UI 只读第一个非空行，，且**不要自带"提醒："前缀**
- 时间表达要转换为具体 epoch（at_epoch）或间隔秒数（interval_s）
- 一次性任务用 schedule_type="at"（框架没有 "once"），且 at_epoch 必须是未来时间
- at 类型任务触发后默认自动删除（delete_after_run 默认为 true）
- 用自然语言确认，如"好的，5分钟后提醒你喝水"
- 如果用户没说具体提醒内容，从上下文推断
