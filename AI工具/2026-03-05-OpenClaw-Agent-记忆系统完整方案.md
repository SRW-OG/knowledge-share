---
title: OpenClaw Agent 记忆系统完整方案
date: 2026-03-05
category: 技术
tags: [OpenClaw, Agent, 记忆系统, FAISS, Total Recall, 向量搜索, 混合搜索]
---

# OpenClaw Agent 记忆系统完整方案

> **最新更新：2026-03-05** — 整合 Total Recall 五层架构的完整记忆系统
> 
> 本文档整合了 OpenClaw 原生记忆、FAISS 轻量化方案、Total Recall 五层架构
> 目标：为"红移先知"构建一个**零维护、低成本、自动观察**的记忆系统

---

## 一、当前系统状态

### 1.1 已有组件

| 组件 | 状态 | 说明 |
|------|------|------|
| memory/ 目录 | ✅ 在用 | 每日会话日志 |
| MEMORY.md | ✅ 在用 | 长期精选记忆 |
| FAISS 索引 | ✅ 在用 | 本地向量索引 |
| BGE-Small-ZH | ✅ 在用 | 512维中文向量模型 |
| BM25 | ❌ 未集成 | 关键词搜索 |
| Observer 层 | ❌ 未实现 | 需手动调用 |
| Session Recovery | ❌ 未实现 | 无自动补漏 |
| Dream Cycle | ❌ 未实现 | 无夜间整合 |

### 1.2 当前文件结构

```
~/.openclaw/workspace/
├── scripts/memory/
│   ├── sync_memory.py      # 主索引脚本
│   ├── run.sh              # 启动脚本
│   ├── memory.sh           # 便捷调用
│   ├── memory.py           # Python模块
│   ├── venv/               # 虚拟环境
│   └── (其他辅助脚本)
├── memory/                 # 每日记忆目录
│   └── YYYY-MM-DD.md
├── MEMORY.md               # 长期记忆
├── memory_index.faiss      # FAISS 索引文件
└── docs_mapping.json       # 文档映射
```

---

## 二、架构设计

### 2.1 设计目标

| 目标 | 指标 |
|------|------|
| 零维护 | 无需手动保存，LLM 自动观察 |
| 低成本 | $0/月（本地模型 + 免费 API） |
| 高可靠 | 五层冗余，无记忆丢失 |
| 轻量化 | 内存占用 <100MB |

### 2.2 五层架构（整合 Total Recall）

```
Layer 1: Observer (cron 15min)
    ↓ 压缩最近会话 → observations.md
Layer 2: Reflector (自动触发 >8000词)
    ↓ 整合 + 去重 → 40-60% 压缩
Layer 3: Session Recovery (/new 触发)
    ↓ 自动补漏 → 捕获遗漏会话
Layer 4: Reactive Watcher (inotify)
    ↓ 高频活动 → 快速触发 Observer
Layer 5: Pre-compaction Hook
    ↓ 上下文压缩前 → 紧急捕获
```

### 2.3 三层存储结构

```
┌─────────────────────────────────────────────┐
│            感知层 (Recent Context)            │
│  最近 2-3 轮对话（内存，无持久化）          │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│            短程层 (Episodic Memory)          │
│  每日日志: memory/YYYY-MM-DD.md             │
│  Observer 压缩: observations.md             │
│  索引: BM25 + 时间戳                        │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│            长程层 (Semantic Memory)          │
│  MEMORY.md（精选事实）                      │
│  知识库: knowledge-share/                   │
│  索引: FAISS + BGE-Small-ZH                │
└─────────────────────────────────────────────┘
```

---

## 三、核心组件实现

### 3.1 Layer 1: Observer（观察者）

**功能**：每 15 分钟自动压缩最近会话为观察笔记

**实现脚本**：`scripts/memory/observer.sh`

```bash
#!/bin/bash
# Observer - 自动观察并压缩会话

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
VENV_DIR="$SCRIPT_DIR/venv"
SESSION_DIR="$HOME/.openclaw/agents/main/sessions"
OBSERVATIONS_FILE="$HOME/.openclaw/workspace/memory/observations.md"

# 激活虚拟环境
source "$VENV_DIR/bin/activate"

# 运行 Observer Python 脚本
python3 -c "
import os
import json
import hashlib
from datetime import datetime, timedelta

SESSION_DIR = os.path.expanduser('$SESSION_DIR')
OBSERVATIONS_FILE = os.path.expanduser('$OBSERVATIONS_FILE')
LOOKBACK_MIN = 15

def get_recent_sessions():
    '''获取最近 LOOKBACK_MIN 分钟的会话'''
    sessions = []
    if not os.path.exists(SESSION_DIR):
        return sessions
    
    cutoff = datetime.now() - timedelta(minutes=LOOKBACK_MIN)
    
    for f in os.listdir(SESSION_DIR):
        if not f.endswith('.jsonl'):
            continue
        # 跳过 subagent/cron 会话
        if 'subagent' in f or 'cron' in f:
            continue
        
        path = os.path.join(SESSION_DIR, f)
        mtime = datetime.fromtimestamp(os.path.getmtime(path))
        
        if mtime > cutoff:
            # 读取消息
            messages = []
            try:
                with open(path, 'r') as fp:
                    for line in fp:
                        try:
                            msg = json.loads(line)
                            if msg.get('role') in ['user', 'assistant']:
                                content = msg.get('content', '')
                                if content:
                                    messages.append(f\"{msg.get('role')}: {content[:200]}\")
                        except:
                            pass
                if messages:
                    sessions.append({
                        'file': f,
                        'messages': messages[-10:]  # 最近10条
                    })
            except Exception as e:
                print(f'Error reading {f}: {e}')
    
    return sessions

def compute_hash(sessions):
    '''计算会话哈希用于去重'''
    content = json.dumps(sessions, sort_keys=True)
    return hashlib.md5(content.encode()).hexdigest()

def load_last_hash():
    '''加载上次处理的哈希'''
    hash_file = '$HOME/.openclaw/workspace/memory/.observer-last-hash'
    if os.path.exists(hash_file):
        with open(hash_file, 'r') as f:
            return f.read().strip()
    return None

def save_hash(hash_val):
    '''保存本次哈希'''
    hash_file = '$HOME/.openclaw/workspace/memory/.observer-last-hash'
    with open(hash_file, 'w') as f:
        f.write(hash_val)

# 主逻辑
sessions = get_recent_sessions()
if not sessions:
    print('No recent sessions found')
    exit(0)

current_hash = compute_hash(sessions)
last_hash = load_last_hash()

if current_hash == last_hash:
    print('No new messages since last run')
    exit(0)

# 构建压缩 prompt（简化版，实际使用 LLM 压缩）
compressed = f\"# 观察记录 - {datetime.now().strftime('%Y-%m-%d %H:%M')}\n\n\"
for s in sessions:
    compressed += f\"## {s['file']}\n\"
    for m in s['messages']:
        compressed += f\"- {m[:150]}\n\"
    compressed += \"\n\"

# 追加到 observations.md
with open(OBSERVATIONS_FILE, 'a', encoding='utf-8') as f:
    f.write(compressed + '\n')

save_hash(current_hash)
print(f'Observations saved: {len(sessions)} sessions')
"
```

**配置 cron**：

```bash
# 每 15 分钟运行 Observer
*/15 * * * * cd $HOME/.openclaw/workspace && bash scripts/memory/observer.sh >> /tmp/observer.log 2>&1
```

### 3.2 Layer 2: Reflector（反射者）

**功能**：当 observations.md 超过 8000 词时自动整合

**触发条件**：observations.md 词数 > 8000

**实现**：

```python
# scripts/memory/reflector.py
#!/usr/bin/env python3
"""Reflector - 记忆整合脚本"""

import os
import json
import requests
from datetime import datetime

OBSERVATIONS_FILE = os.path.expanduser("~/.openclaw/workspace/memory/observations.md")
BACKUP_DIR = os.path.expanduser("~/.openclaw/workspace/memory/observation-backups/")
WORD_THRESHOLD = 8000

ZHIPU_API_KEY = os.environ.get("ZHIPU_API_KEY", "")

def count_words(text):
    return len(text.split())

def backup_observations():
    """备份当前 observations"""
    os.makedirs(BACKUP_DIR, exist_ok=True)
    backup_file = os.path.join(BACKUP_DIR, f"observations_{datetime.now().strftime('%Y%m%d_%H%M%S')}.md")
    with open(OBSERVATIONS_FILE, 'r', encoding='utf-8') as src:
        with open(backup_file, 'w', encoding='utf-8') as dst:
            dst.write(src.read())
    print(f"Backup saved: {backup_file}")

def consolidate_with_llm(text):
    """使用 LLM 整合记忆"""
    if not ZHIPU_API_KEY:
        print("No ZHIPU_API_KEY, skipping consolidation")
        return text
    
    prompt = f"""请压缩以下记忆，保留关键信息：
1. 重要事实和决策
2. 用户偏好和习惯
3. 项目进度和里程碑
4. 去除重复和过时信息

目标：压缩到原文的 40-60%，保留核心信息。

原文：
{text[:8000]}"""  # 限制输入长度
    
    try:
        response = requests.post(
            "https://open.bigmodel.cn/api/paas/v4/chat/completions",
            headers={"Authorization": f"Bearer {ZHIPU_API_KEY}"},
            json={
                "model": "glm-4-flash",
                "messages": [{"role": "user", "content": prompt}],
                "temperature": 0.3
            },
            timeout=30
        )
        result = response.json()
        return result['choices'][0]['message']['content']
    except Exception as e:
        print(f"LLM consolidation failed: {e}")
        return text

def run_reflector():
    """运行 Reflector"""
    if not os.path.exists(OBSERVATIONS_FILE):
        print("No observations file")
        return
    
    with open(OBSERVATIONS_FILE, 'r', encoding='utf-8') as f:
        content = f.read()
    
    word_count = count_words(content)
    print(f"Current words: {word_count}")
    
    if word_count < WORD_THRESHOLD:
        print(f"Below threshold ({WORD_THRESHOLD}), skipping")
        return
    
    print("Running consolidation...")
    backup_observations()
    
    consolidated = consolidate_with_llm(content)
    
    with open(OBSERVATIONS_FILE, 'w', encoding='utf-8') as f:
        f.write(consolidated)
    
    new_count = count_words(consolidated)
    reduction = (1 - new_count / word_count) * 100
    print(f"Consolidated: {word_count} -> {new_count} words ({reduction:.1f}% reduction)")

if __name__ == "__main__":
    run_reflector()
```

### 3.3 Layer 3: Session Recovery（会话恢复）

**功能**：在 `/new` 或 `/reset` 时自动补漏

**实现**：

```python
# scripts/memory/session_recovery.py
#!/usr/bin/env python3
"""Session Recovery - 会话恢复脚本"""

import os
import json
import hashlib
from datetime import datetime, timedelta

SESSION_DIR = os.path.expanduser("~/.openclaw/agents/main/sessions")
LAST_SESSION_FILE = "$HOME/.openclaw/workspace/memory/.last-session-hash"
LOOKBACK_HOURS = 4

def hash_recent_lines(filepath, lines=50):
    """计算文件最近 N 行的哈希"""
    try:
        with open(filepath, 'r') as f:
            all_lines = f.readlines()
            recent = all_lines[-lines:]
            content = ''.join(recent)
            return hashlib.md5(content.encode()).hexdigest()
    except:
        return None

def get_last_session_hash():
    """获取上次 Observer 运行的会话哈希"""
    if os.path.exists(LAST_SESSION_FILE):
        with open(LAST_SESSION_FILE, 'r') as f:
            return f.read().strip()
    return None

def save_session_hash(h):
    """保存当前会话哈希"""
    with open(LAST_SESSION_FILE, 'w') as f:
        f.write(h)

def recover_missed_sessions():
    """恢复遗漏的会话"""
    if not os.path.exists(SESSION_DIR):
        return []
    
    last_hash = get_last_session_hash()
    cutoff = datetime.now() - timedelta(hours=LOOKBACK_HOURS)
    missed = []
    
    for f in os.listdir(SESSION_DIR):
        if not f.endswith('.jsonl'):
            continue
        
        path = os.path.join(SESSION_DIR, f)
        mtime = datetime.fromtimestamp(os.path.getmtime(path))
        
        if mtime > cutoff:
            current_hash = hash_recent_lines(path)
            if current_hash and current_hash != last_hash:
                missed.append(path)
                print(f"Missed session detected: {f}")
    
    return missed

def main():
    """主函数 - 在 /new 时调用"""
    missed = recover_missed_sessions()
    
    if missed:
        print(f"Found {len(missed)} missed sessions")
        # 运行 Observer 进行恢复
        # 调用 observer.sh 或直接处理
    else:
        print("No missed sessions")

if __name__ == "__main__":
    main()
```

### 3.4 Layer 4: Reactive Watcher（反应式监视器）

**功能**：使用 inotify 监测会话文件，高频活动时快速触发 Observer

**要求**：Linux + inotify-tools

**安装**：

```bash
sudo apt install inotify-tools
```

**实现**：

```bash
#!/bin/bash
# observer-watcher.sh - 反应式监视器

WATCH_DIR="$HOME/.openclaw/agents/main/sessions"
COOLDOWN_FILE="$HOME/.openclaw/workspace/memory/.watcher-cooldown"
COOLDOWN_SECS=300  # 5分钟冷却
LINE_THRESHOLD=40

inotifywait -m -e modify -e close_write "$WATCH_DIR" --format '%f' 2>/dev/null | \
while read file; do
    if [[ "$file" == *.jsonl ]] && [[ "$file" != *"subagent"* ]]; then
        # 检查冷却
        if [ -f "$COOLDOWN_FILE" ]; then
            last_run=$(cat "$COOLDOWN_FILE")
            now=$(date +%s)
            if (( now - last_run < COOLDOWN_SECS )); then
                continue
            fi
        fi
        
        # 检查行数
        lines=$(wc -l < "$WATCH_DIR/$file")
        if (( lines >= LINE_THRESHOLD )); then
            echo "High activity detected: $file ($lines lines)"
            date +%s > "$COOLDOWN_FILE"
            
            # 触发 Observer
            cd "$HOME/.openclaw/workspace"
            bash scripts/memory/observer.sh >> /tmp/observer.log 2>&1
        fi
    fi
done
```

**配置 systemd 服务**：

```bash
# ~/.config/systemd/user/total-recall-watcher.service
[Unit]
Description=Total Recall Observer Watcher

[Service]
Type=simple
ExecStart=%h/.openclaw/workspace/scripts/memory/observer-watcher.sh
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now total-recall-watcher
```

### 3.5 Layer 5: Pre-compaction Hook（预压缩钩子）

**功能**：OpenClaw 压缩上下文前自动保存记忆

**实现**：在 OpenClaw 配置中添加 memoryFlush：

```json
{
  "compaction": {
    "memoryFlush": {
      "enabled": true,
      "softThresholdTokens": 4000,
      "prompt": "在上下文被压缩前，将重要信息保存到 memory/YYYY-MM-DD.md"
    }
  }
}
```

---

## 四、搜索系统实现

### 4.1 混合搜索（BM25 + FAISS）

**更新 sync_memory.py**：

```python
#!/usr/bin/env python3
"""
记忆系统索引构建脚本 - 混合搜索版本
支持 BM25 关键词搜索 + FAISS 向量搜索
"""

import os
import sys
import json
import re
import faiss
import numpy as np
from rank_bm25 import BM25Okapi
from sentence_transformers import SentenceTransformer

# 配置
MEMORY_DIR = os.path.expanduser("~/.openclaw/workspace/memory/")
KNOWLEDGE_DIR = os.path.expanduser("~/Documents/Obsidian/knowledge-share/")
MODEL_NAME = 'BAAI/bge-small-zh-v1.5'
INDEX_FILE = os.path.expanduser("~/.openclaw/workspace/memory_index.faiss")
MAPPING_FILE = os.path.expanduser("~/.openclaw/workspace/docs_mapping.json")

# 忽略文件列表
IGNORE_FILES = ['config.json', '.env', 'secrets.md', 'credentials.md']

# 权重配置
BM25_WEIGHT = 0.3
FAISS_WEIGHT = 0.7


def sanitize_text(text):
    """脱敏处理"""
    patterns = {
        "KEY": r'sk-[a-zA-Z0-9]{32,}',
        "PASSWORD": r'password[:=]\s*[^\s]+',
        "DB_URL": r'postgresql://\w+:\w+@[\w\.-]+:\d+/\w+',
        "HEX_SECRET": r'([0-9a-fA-F]{32,})',
    }
    for label, pattern in patterns.items():
        text = re.sub(pattern, f"[{label}_REDACTED]", text)
    return text


def load_docs():
    """加载所有 Markdown 文档"""
    docs = []
    paths = []
    sources = []
    
    for directory in [MEMORY_DIR, KNOWLEDGE_DIR]:
        if not os.path.exists(directory):
            continue
        
        source = "memory" if directory == MEMORY_DIR else "knowledge"
        
        for root, dirs, files in os.walk(directory):
            for file in files:
                if file.endswith(".md") and file not in IGNORE_FILES:
                    path = os.path.join(root, file)
                    try:
                        with open(path, 'r', encoding='utf-8') as f:
                            content = f.read()
                            if content.strip():
                                docs.append(content)
                                paths.append(path)
                                sources.append(source)
                    except Exception as e:
                        print(f"  ⚠️ 读取失败: {path}")
    
    return docs, paths, sources


def build_index():
    """构建 FAISS 索引"""
    docs, paths, sources = load_docs()
    
    if not docs:
        print("❌ 没有找到 Markdown 文件")
        return
    
    print(f"✅ 找到 {len(docs)} 个文档")
    
    # 生成向量
    print(f"🔄 加载模型: {MODEL_NAME}")
    model = SentenceTransformer(MODEL_NAME)
    
    print("📊 生成向量中...")
    embeddings = model.encode(docs, show_progress_bar=True)
    
    # 创建 FAISS 索引
    dimension = embeddings.shape[1]
    index = faiss.IndexFlatL2(dimension)
    index.add(np.array(embeddings))
    
    # 保存索引
    print(f"💾 保存索引到: {INDEX_FILE}")
    faiss.write_index(index, INDEX_FILE)
    
    # 保存文档映射
    print(f"💾 保存映射到: {MAPPING_FILE}")
    with open(MAPPING_FILE, 'w', encoding='utf-8') as f:
        json.dump({
            "paths": paths, 
            "docs": docs, 
            "sources": sources
        }, f, ensure_ascii=False)
    
    print(f"✅ 索引构建完成!")


def hybrid_search(query, top_k=5):
    """混合搜索：BM25 + FAISS"""
    
    if not os.path.exists(INDEX_FILE):
        print("❌ 索引文件不存在")
        return []
    
    # 加载数据
    with open(MAPPING_FILE, 'r', encoding='utf-8') as f:
        data = json.load(f)
    
    docs = data["docs"]
    paths = data["paths"]
    sources = data.get("sources", ["unknown"] * len(paths))
    
    # BM25 搜索
    print("🔍 BM25 搜索...")
    tokenized_docs = [doc.lower().split() for doc in docs]
    bm25 = BM25Okapi(tokenized_docs)
    bm25_scores = bm25.get_scores(query.lower().split())
    
    # FAISS 搜索
    print("🔍 FAISS 搜索...")
    model = SentenceTransformer(MODEL_NAME)
    index = faiss.read_index(INDEX_FILE)
    query_vec = model.encode([query])
    distances, faiss_indices = index.search(query_vec, top_k * 2)
    
    # 归一化 FAISS 分数（距离转分数）
    max_dist = max(distances[0]) + 1e-6
    faiss_scores = [1 - d / max_dist for d in distances[0]]
    
    # 加权合并
    print("📊 合并结果...")
    combined_scores = []
    all_indices = set(faiss_indices[0]) | set(range(len(bm25_scores)))
    
    for idx in all_indices:
        bm25_score = bm25_scores[idx] if idx < len(bm25_scores) else 0
        faiss_score = faiss_scores[idx] if idx < len(faiss_scores) else 0
        
        combined = BM25_WEIGHT * bm25_score + FAISS_WEIGHT * faiss_score
        combined_scores.append((idx, combined))
    
    # 排序
    combined_scores.sort(key=lambda x: x[1], reverse=True)
    
    # 返回 top k
    results = []
    for idx, score in combined_scores[:top_k]:
        if idx < len(paths):
            safe_content = sanitize_text(docs[idx][:500])
            results.append({
                "path": paths[idx],
                "content": safe_content,
                "source": sources[idx] if idx < len(sources) else "unknown",
                "score": score,
                "bm25": bm25_scores[idx] if idx < len(bm25_scores) else 0,
                "faiss": faiss_scores[idx] if idx < len(faiss_scores) else 0
            })
            print(f"  - [{sources[idx]}] {score:.3f} | {paths[idx]}")
    
    return results


if __name__ == "__main__":
    if len(sys.argv) > 1 and sys.argv[1] == "search":
        query = " ".join(sys.argv[2:]) if len(sys.argv) > 2 else "测试"
        hybrid_search(query)
    else:
        build_index()
```

### 4.2 安装依赖

```bash
pip install faiss-cpu sentence-transformers rank_bm25
```

---

## 五、重要性评分系统

### 5.1 记忆类型定义

| 类型 | TTL | 衰减率 | 说明 |
|------|-----|--------|------|
| event | 14d | -0.5/天 | 重要事件 |
| fact | 90d | -0.1/天 | 事实性信息 |
| preference | 180d | -0.02/天 | 用户偏好 |
| goal | 365d | 无 | 目标规划 |
| habit | 365d | 无 | 习惯模式 |
| rule | never | 无 | 规则约束 |
| context | 30d | 无 | 当前上下文 |

### 5.2 元数据格式

在 observations.md 中使用 HTML 注释：

```markdown
<!-- importance: 7.5, type: fact, created: 2026-03-01, decay: -0.1/day, expires: 2026-06-01 -->

## 用户偏好

观测者偏好简洁直接的沟通风格。
```

### 5.3 衰减脚本

```python
# scripts/memory/importance_decay.py
#!/usr/bin/env python3
"""重要性衰减脚本 - 定期运行"""

import re
import os
from datetime import datetime, timedelta

OBSERVATIONS_FILE = os.path.expanduser("~/.openclaw/workspace/memory/observations.md")
ARCHIVE_DIR = os.path.expanduser("~/.openclaw/workspace/memory/archive/")

# 衰减配置
DECAY_RATES = {
    "event": 0.5,
    "fact": 0.1,
    "preference": 0.02,
    "goal": 0,
    "habit": 0,
    "rule": 0,
    "context": 0.1
}

THRESHOLD = 3.0


def parse_metadata(line):
    """解析元数据"""
    match = re.search(r'importance:\s*([\d.]+),\s*type:\s*(\w+)', line)
    if match:
        return float(match.group(1)), match.group(2)
    return None, None


def apply_decay(content):
    """应用衰减并归档低价值记忆"""
    lines = content.split('\n')
    archived = []
    kept = []
    
    for line in lines:
        importance, mem_type = parse_metadata(line)
        
        if importance is not None and mem_type in DECAY_RATES:
            decay = DECAY_RATES.get(mem_type, 0)
            new_importance = importance - decay
            
            if new_importance < THRESHOLD:
                # 归档
                archived.append(line)
                continue
        
        kept.append(line)
    
    return '\n'.join(kept), archived


def main():
    """主函数"""
    if not os.path.exists(OBSERVATIONS_FILE):
        return
    
    with open(OBSERVATIONS_FILE, 'r', encoding='utf-8') as f:
        content = f.read()
    
    kept_content, archived = apply_decay(content)
    
    if archived:
        # 归档
        os.makedirs(ARCHIVE_DIR, exist_ok=True)
        archive_file = os.path.join(ARCHIVE_DIR, f"archived_{datetime.now().strftime('%Y%m%d')}.md")
        
        with open(archive_file, 'a', encoding='utf-8') as f:
            f.write(f"\n## Archived at {datetime.now()}\n")
            for line in archived:
                f.write(line + "\n")
        
        # 更新原文件
        with open(OBSERVATIONS_FILE, 'w', encoding='utf-8') as f:
            f.write(kept_content)
        
        print(f"Archived {len(archived)} items")
    else:
        print("No items to archive")


if __name__ == "__main__":
    main()
```

---

## 六、自动更新 MEMORY.md

### 6.1 每周提炼脚本

```python
# scripts/memory/weekly_consolidate.py
#!/usr/bin/env python3
"""每周记忆提炼脚本"""

import os
import requests
from datetime import datetime, timedelta

ZHIPU_API_KEY = os.environ.get("ZHIPU_API_KEY", "")
MEMORY_DIR = os.path.expanduser("~/.openclaw/workspace/memory/")
MEMORY_FILE = os.path.expanduser("~/.openclaw/workspace/MEMORY.md")


def collect_week_logs():
    """收集本周日志"""
    week_ago = (datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d")
    content = []
    
    for f in os.listdir(MEMORY_DIR):
        if f.endswith(".md") and f >= week_ago:
            path = os.path.join(MEMORY_DIR, f)
            with open(path, 'r', encoding='utf-8') as fp:
                content.append(fp.read())
    
    return "\n\n---\n\n".join(content)


def extract_to_memory(combined):
    """提炼到 MEMORY.md"""
    if not ZHIPU_API_KEY:
        print("No API key")
        return
    
    prompt = f"""从以下一周日志中提取重要信息，以简洁的 Markdown 格式输出：

## 用户信息
- 姓名、关系

## 项目进度
- 进行中的项目
- 关键里程碑

## 偏好习惯
- 沟通风格偏好
- 工作习惯

## 决策记录
- 重要决定

## 待办事项
- 未完成的任务

日志内容：
{combined[:6000]}"""
    
    try:
        response = requests.post(
            "https://open.bigmodel.cn/api/paas/v4/chat/completions",
            headers={"Authorization": f"Bearer {ZHIPU_API_KEY}"},
            json={
                "model": "glm-4-flash",
                "messages": [{"role": "user", "content": prompt}],
                "temperature": 0.3
            },
            timeout=30
        )
        result = response.json()
        extracted = result['choices'][0]['message']['content']
        
        with open(MEMORY_FILE, 'a', encoding='utf-8') as f:
            f.write(f"\n\n## {datetime.now().strftime('%Y-%m-%d')} 周总结\n\n")
            f.write(extracted)
        
        print("MEMORY.md updated")
    except Exception as e:
        print(f"Error: {e}")


def main():
    combined = collect_week_logs()
    if combined:
        extract_to_memory(combined)
    else:
        print("No logs to process")


if __name__ == "__main__":
    main()
```

### 6.2 配置 cron

```bash
# 每周日 23:00 运行
0 23 * * 0 cd $HOME/.openclaw/workspace && source scripts/memory/venv/bin/activate && python3 scripts/memory/weekly_consolidate.py >> /tmp/weekly_memory.log 2>&1
```

---

## 七、完整 Cron 配置

```bash
# OpenClaw 记忆系统 Cron 配置

# Observer - 每 15 分钟
*/15 * * * * cd $HOME/.openclaw/workspace && bash scripts/memory/observer.sh >> /tmp/observer.log 2>&1

# Reflector - 每小时检查
0 * * * * cd $HOME/.openclaw/workspace && source scripts/memory/venv/bin/activate && python3 scripts/memory/reflector.py >> /tmp/reflector.log 2>&1

# 索引重建 - 每天 3 点
0 3 * * * cd $HOME/.openclaw/workspace && source scripts/memory/venv/bin/activate && python3 scripts/memory/sync_memory.py >> /tmp/memory_build.log 2>&1

# 周总结 - 每周日 23 点
0 23 * * 0 cd $HOME/.openclaw/workspace && source scripts/memory/venv/bin/activate && python3 scripts/memory/weekly_consolidate.py >> /tmp/weekly_memory.log 2>&1

# 签到任务 - 每天 9:30
30 9 * * * cd $HOME/.openclaw/workspace && source scripts/checkin/venv/bin/activate && python3 scripts/checkin/scripts/signin_manager.py run >> /tmp/signin.log 2>&1

# 签到任务 - 每天 8:00 (备用)
0 8 * * * cd $HOME/.openclaw/workspace && source scripts/checkin/venv/bin/activate && python3 scripts/checkin/scripts/signin_manager.py run >> /tmp/signin.log 2>&1

# Moltbook 任务 - 每天 5:00
0 5 * * * cd $HOME/.openclaw/workspace && source scripts/checkin/venv/bin/activate && python3 scripts/checkin/scripts/moltbook_manager.py >> /tmp/moltbook_tasks.log 2>&1

# Moltbook Ability - 每天 5:05
5 5 * * * cd $HOME/.openclaw/workspace && source scripts/checkin/venv/bin/activate && python3 scripts/checkin/scripts/moltbook_ability.py >> /tmp/moltbook_ability.log 2>&1
```

---

## 八、文件位置汇总

| 文件 | 位置 | 说明 |
|------|------|------|
| Observer | `scripts/memory/observer.sh` | 15分钟自动观察 |
| Reflector | `scripts/memory/reflector.py` | 记忆整合 |
| Session Recovery | `scripts/memory/session_recovery.py` | 会话补漏 |
| Reactive Watcher | `scripts/memory/observer-watcher.sh` | 高频触发 |
| Sync Memory | `scripts/memory/sync_memory.py` | 索引构建+搜索 |
| Weekly Consolidate | `scripts/memory/weekly_consolidate.py` | 周总结 |
| Importance Decay | `scripts/memory/importance_decay.py` | 重要性衰减 |

| 数据 | 位置 | 说明 |
|------|------|------|
| 每日记忆 | `memory/YYYY-MM-DD.md` | 会话日志 |
| 观察笔记 | `memory/observations.md` | Observer 输出 |
| 长期记忆 | `MEMORY.md` | 精选事实 |
| 向量索引 | `memory_index.faiss` | FAISS 索引 |
| 文档映射 | `docs_mapping.json` | 索引映射 |
| 归档 | `memory/archive/` | 过期记忆 |

---

## 九、依赖安装

```bash
# 核心依赖
pip install faiss-cpu sentence-transformers rank_bm25

# 可选：inotify-tools (Linux)
sudo apt install inotify-tools
```

---

## 十、使用方法

### 10.1 搜索记忆

```bash
# 方式1：直接搜索
./scripts/memory/run.sh search "关键词"

# 方式2：使用 Python 模块
python3 -c "from sync_memory import hybrid_search; import json; print(json.dumps(hybrid_search('你的问题'), indent=2))"
```

### 10.2 手动触发 Observer

```bash
bash scripts/memory/observer.sh
```

### 10.3 手动重建索引

```bash
python3 scripts/memory/sync_memory.py
```

### 10.4 查看日志

```bash
tail -f /tmp/observer.log
tail -f /tmp/reflector.log
tail -f /tmp/memory_build.log
```

---

## 十一、后续规划

### Phase 1（已完成）
- [x] FAISS 索引构建
- [x] 基础向量搜索

### Phase 2（本方案）
- [ ] Observer 脚本部署
- [ ] Reflector 脚本部署
- [ ] Session Recovery 部署
- [ ] 混合搜索集成
- [ ] BM25 + FAISS 加权

### Phase 3（可选）
- [ ] Reactive Watcher（需要 inotify）
- [ ] Importance Decay
- [ ] Dream Cycle 夜间整合
- [ ] Memory Type TTL

---

## 参考资料

- [Total Recall GitHub](https://github.com/gavdalf/total-recall)
- [OpenClaw Memory 文档](https://docs.openclaw.ai/concepts/memory)
- [openclaw-local-memory](https://github.com/tituss-bit/openclaw-local-memory)
- [BGE-Small-ZH](https://huggingface.co/BAAI/bge-small-zh-v1.5)

---

*最后更新：2026-03-05 — 整合 Total Recall 五层架构*
