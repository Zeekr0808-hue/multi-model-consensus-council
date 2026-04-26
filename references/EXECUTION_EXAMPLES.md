# 多模型决策委员会 — 执行步骤示例

## 完整执行流程

> **注意**：各轮 Prompt 模板统一存放在 `OUTPUT_TEMPLATE.md` 中，本文件仅描述执行步骤和 Agent 操作逻辑，不重复 Prompt 内容。

### 场景：方案A vs 方案B 多模型评审

---

## Step 1：激活技能 + 动态扫描模型库

**触发**：用户说"帮我决策方案A和方案B哪个更好"

**Agent 执行**：
```python
# 1. 扫描本地模型库（动态读取 openclaw.json）
import json
with open("~/.openclaw/openclaw.json") as f:
    config = json.load(f)
providers = config.get("models", {}).get("providers", {})
available_models = []
for pname, pcfg in providers.items():
    for m in pcfg.get("models", []):
        available_models.append({
            "id": m["id"],
            "name": m.get("name", m["id"]),
            "provider": pname
        })

# 2. 展示模型列表，供用户选择
display_model_list(available_models)  # 格式：编号 | 模型名 | 提供商

# 3. 接收用户配置（或使用默认前3个）
selected = user_selection or available_models[:3]
```

**Agent 响应**：
```
已识别决策任务：
  方案A：（提取的方案A摘要）
  方案B：（提取的方案B摘要）

当前可用模型（动态扫描）：
  [1] GPT-5.4 (N1N)
  [2] GLM-5.1 (火山方舟)
  [3] Kimi K2.6 (火山方舟)
  ...（共 N 个）

当前配置：委员 [1] + [3] + [5]
决策轮数：3轮 | 通过阈值：≥90%

输入"确认配置"开始决策，或说"改轮数"/"换模型"调整参数。
```

---

## Step 2：第1轮 — 并行 Spawn 子会话

**Agent 执行**：
```python
# 从 OUTPUT_TEMPLATE.md 加载第1轮 Prompt 模板
prompt_r1 = load_template("references/OUTPUT_TEMPLATE.md", "第1轮独立评审")

# 并行 spawn N 个子会话
for model_info in selected_models:
    spawn(
        model=model_info["id"],
        prompt=prompt_r1.format(
            task="方案A vs 方案B决策",
            option_a=option_a_content,
            option_b=option_b_content
        ),
        session_label=f"mmc_r1_{model_info['id']}"
    )
```

**预期输出**：各子会话返回独立评审报告（评分 + 推荐顺序 + 核心论点）

---

## Step 3：汇总第1轮结果，判断收敛性

**Agent 执行**：
```python
# 收集所有子会话结果
results_r1 = []
for model_info in selected_models:
    history = sessions_history(f"mmc_r1_{model_info['id']}")
    results_r1.append(extract_review_result(history))

# 提取分歧点
disputes = extract_disputes(results_r1)

# 判断收敛性：若全票推荐顺序一致，跳过 Round 2 和 Round 3
if is_converged(results_r1):
    generate_final_report(results_r1)  # 直接输出6段式报告
else:
    proceed_to_round2(disputes)
```

---

## Step 4：第2轮 — 收敛讨论（仅在有分歧时执行）

**Agent 执行**：
```python
# 从 OUTPUT_TEMPLATE.md 加载第2轮 Prompt 模板
prompt_r2 = load_template("references/OUTPUT_TEMPLATE.md", "第2轮收敛讨论")

for model_info in selected_models:
    spawn(
        model=model_info["id"],
        prompt=prompt_r2.format(
            round1_results=results_r1,
            disputes=disputes
        ),
        session_label=f"mmc_r2_{model_info['id']}"
    )
```

**收敛判定**：若第2轮全票收敛 → 跳过 Round 3，直接生成最终报告

---

## Step 5：第3轮 — 最终辩论（仅在第2轮仍有分歧时执行）

**Agent 执行**：
```python
prompt_r3 = load_template("references/OUTPUT_TEMPLATE.md", "第3轮最终辩论")

for model_info in selected_models:
    spawn(
        model=model_info["id"],
        prompt=prompt_r3.format(remaining_disputes=remaining_disputes),
        session_label=f"mmc_r3_{model_info['id']}"
    )
```

---

## Step 6：生成最终报告

**Agent 执行**：
```python
# 收集所有轮次结果
all_results = {
    "round1": results_r1,
    "round2": results_r2 if has_round2 else None,
    "round3": results_r3 if has_round3 else None,
    "convergence": {
        "r2_converged": is_converged(results_r2) if has_round2 else True,
        "skipped_r3": is_converged(results_r2) if has_round2 else False
    }
}

# 从 OUTPUT_TEMPLATE.md 加载6段式报告模板，生成报告
report = generate_6section_report(all_results)

# 写入状态文件
write_state_file({
    "schema_version": "v1.0.0",
    "schema_fingerprint": compute_fingerprint(state),
    "state": "COMPLETE",
    "report": report
})
```

---

## 状态文件操作

```python
# 读取状态文件
state = read("~/.openclaw/workspace/memory/mmc_state_{session}.json")

# 写入状态文件（每轮结束后必须更新）
write("~/.openclaw/workspace/memory/mmc_state_{session}.json", {
    "schema_version": "v1.0.0",
    "schema_fingerprint": "sha256:xxxx",
    "state": "ROUND_2",
    "round1": results_r1
})
```

---

## 异常处理

```python
# 某委员超时
try:
    result = sessions_history(f"mmc_r1_{model_id}", timeout=60)
except TimeoutError:
    results_r1.append({
        "model": model_id,
        "status": "TIMEOUT",
        "note": "委员超时，结果由其他委员加权推算"
    })

# Schema 版本校验
state = read_state_file(path)
if state["schema_version"] != "v1.0.0":
    raise SchemaVersionError(f"版本不匹配: {state['schema_version']}")
if not verify_fingerprint(state):
    raise FingerprintError("状态文件已被篡改")
```
