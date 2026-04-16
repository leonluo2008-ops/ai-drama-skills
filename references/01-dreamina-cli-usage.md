# Dreamina CLI 使用规范（经验教训）

## 任务状态判断逻辑

```
dreamina text2image --poll=0
  → 立即返回 {submit_id, gen_status: "querying"}
  → 任务在服务器端执行，本地拿到 submit_id

dreamina query_result --submit_id=<id>
  → 返回 {gen_status: "success"/"querying"/"failed"}
  → 不传 poll 参数，没有自动轮询
```

**核心问题：`--poll` 是 text2image 的参数，不是 query_result 的参数。**

---

## 错误做法（导致重复提交）

```bash
# ❌ 错误：没有保存 submit_id，反复用 --poll=30 重新提交
dreamina text2image --prompt="场景1" --ratio=1:1 --poll=30
# → submit_id: 任务1，30秒后返回（如果还在排队，仍返回 querying）
sleep 60
# ❌ 再次调用 --poll=30 → 提交了新任务：任务2
dreamina text2image --prompt="场景1" --ratio=1:1 --poll=30
```

结果：场景1提交了 2 个任务，实际只需要 1 个。

---

## 正确做法

```bash
# ✅ 第一步：提交任务，立即返回 submit_id
result=$(dreamina text2image --prompt="场景1" --ratio=1:1 --poll=0)
submit_id=$(echo "$result" | python3 -c "import json,sys; print(json.load(sys.stdin)['submit_id'])")
echo "submit_id=$submit_id"

# ✅ 第二步：轮询状态（自己写循环，不是 --poll）
while true; do
  status=$(dreamina query_result --submit_id=$submit_id | python3 -c "import json,sys; print(json.load(sys.stdin)['gen_status'])")
  echo "status=$status"
  if [ "$status" = "success" ]; then
    echo "完成，下载图片"
    break
  elif [ "$status" = "failed" ]; then
    echo "失败，需要重新提交"
    break
  fi
  sleep 10
done
```

---

## 关于 --poll 参数的正确理解

`--poll=N` 的行为：
- 提交任务后，等待 N 秒，每秒检查一次结果
- N 秒内完成 → 返回 success
- N 秒后仍在排队 → **返回 querying**，不重复提交
- 所以 `--poll=30` 不会无限提交，它只在 30 秒内轮询

**但是**：超过 30 秒排队是常见现象，此时任务还在服务器跑，你不应该再调用一次 `text2image`，应该继续轮询。

---

## 图片重复发送问题

```python
# ❌ 错误：每次轮询都发送图片
for step in steps:
    generate()
    send_image()  # 每次都发，包括重复的轮询循环

# ✅ 正确：只在任务完成后发送一次
result = submit_task()
poll_until_done(result.submit_id)
download(result)
send_image()  # 只在确认完成后发送一次
```

---

## 并发生图时的正确管理

```python
# 提交所有任务，收集 submit_id
tasks = []
for scene in scenes:
    result = submit(scene)
    tasks.append({"name": scene, "submit_id": result.submit_id})

# 统一轮询所有任务
pending = {t["submit_id"]: t["name"] for t in tasks}
while pending:
    for submit_id, name in list(pending.items()):
        status = query_result(submit_id).gen_status
        if status == "success":
            download_and_send(submit_id, name)
            del pending[submit_id]
        elif status == "failed":
            # 处理失败
            del pending[submit_id]
    if pending:
        sleep(10)
```

---

## 经验总结

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 重复提交多个任务 | 没有保存 submit_id，用 `--poll=30` 反复提交 | 保存 submit_id，用 query_result 轮询 |
| 图片发送两次 | 在循环中每次轮询都发图 | 只在 success 后发一次 |
| 任务状态判断错误 | 以为 --poll=30 会在后台持续等待 | --poll 只等 N 秒，超时返回 querying，需继续轮询 |
