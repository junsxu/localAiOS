# 用户画像建模与偏好提取

## 用户画像数据结构

```json
{
  "user_id": "default",
  "name": "Jun",
  "preferences": {
    "language":       "zh-CN",
    "code_style":     "简洁，有必要的注释",
    "output_format":  "markdown",
    "detail_level":   "medium",
    "communication":  "直接，不需要客套话"
  },
  "expertise": ["Python", "Linux", "AI/ML", "Docker"],
  "projects": [
    {
      "name":       "Local AI OS",
      "tech_stack": ["Ollama", "Qdrant", "Python", "Docker"],
      "status":     "active",
      "goal":       "构建本地 AI 操作系统"
    }
  ],
  "hardware": {
    "main_machine":   "Windows 11 + WSL2",
    "gpu":            "NVIDIA dGPU + Intel iGPU",
    "nas":            "QNAP",
    "dev_machine":    "MacBook M1"
  },
  "created_at":  "2026-03-01T00:00:00",
  "updated_at":  "2026-03-31T10:00:00"
}
```

---

## 画像存取实现

```python
# user_profile.py
import json
import sqlite3
from datetime import datetime

DB_PATH = "user_profile.db"


class UserProfileService:

    def __init__(self):
        with sqlite3.connect(DB_PATH) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS profiles (
                    user_id      TEXT PRIMARY KEY,
                    profile_json TEXT NOT NULL,
                    updated_at   TEXT NOT NULL
                )
            """)

    def get(self, user_id: str) -> dict:
        with sqlite3.connect(DB_PATH) as conn:
            row = conn.execute(
                "SELECT profile_json FROM profiles WHERE user_id = ?",
                (user_id,)
            ).fetchone()
        if row:
            return json.loads(row[0])
        return self._default_profile(user_id)

    def save(self, user_id: str, profile: dict):
        profile["updated_at"] = datetime.now().isoformat()
        with sqlite3.connect(DB_PATH) as conn:
            conn.execute(
                "INSERT OR REPLACE INTO profiles VALUES (?, ?, ?)",
                (user_id,
                 json.dumps(profile, ensure_ascii=False),
                 profile["updated_at"])
            )

    def update_preference(self, user_id: str, key: str, value: str):
        profile = self.get(user_id)
        profile.setdefault("preferences", {})[key] = value
        self.save(user_id, profile)
        return profile

    def add_project(self, user_id: str, project: dict):
        profile = self.get(user_id)
        projects = profile.setdefault("projects", [])
        # 更新已存在的项目
        for i, p in enumerate(projects):
            if p.get("name") == project.get("name"):
                projects[i].update(project)
                self.save(user_id, profile)
                return profile
        projects.append(project)
        self.save(user_id, profile)
        return profile

    def add_expertise(self, user_id: str, skill: str):
        profile = self.get(user_id)
        skills = profile.setdefault("expertise", [])
        if skill not in skills:
            skills.append(skill)
            self.save(user_id, profile)
        return profile

    def to_system_prompt_block(self, user_id: str) -> str:
        profile = self.get(user_id)
        prefs = profile.get("preferences", {})
        projects = profile.get("projects", [])
        expertise = profile.get("expertise", [])

        proj_text = "\n".join(
            f"  - {p['name']}（{p.get('status', 'unknown')}）：{p.get('goal', '')}"
            for p in projects
        ) or "  暂无"

        return f"""用户偏好：
  语言：{prefs.get('language', 'zh-CN')}
  代码风格：{prefs.get('code_style', '简洁')}
  输出格式：{prefs.get('output_format', 'markdown')}
  沟通风格：{prefs.get('communication', '直接')}

技术背景：{', '.join(expertise) or '未知'}

当前项目：
{proj_text}"""

    def _default_profile(self, user_id: str) -> dict:
        return {
            "user_id":     user_id,
            "preferences": {},
            "expertise":   [],
            "projects":    [],
            "created_at":  datetime.now().isoformat(),
            "updated_at":  datetime.now().isoformat()
        }


profile_service = UserProfileService()
```

---

## 偏好自动提取

### 基于规则的快速提取

```python
import re

PREFERENCE_MAP = {
    # 语言偏好
    r"用中文|说中文|中文回答": ("language", "zh-CN"),
    r"用英文|英文回答|English": ("language", "en-US"),
    # 代码风格
    r"代码(要|得|应该)?简洁|简洁的代码": ("code_style", "简洁"),
    r"代码加注释|详细注释": ("code_style", "详细注释"),
    r"代码(要|得)?有类型注解|类型提示": ("code_style", "类型注解"),
    # 输出格式
    r"不(要|用)markdown|纯文本": ("output_format", "plain"),
    r"用markdown|markdown格式": ("output_format", "markdown"),
    # 沟通风格
    r"直接说|不用客套|不废话": ("communication", "直接"),
    r"详细解释|多解释": ("communication", "详细"),
}

def extract_preference_by_rules(user_input: str) -> list[tuple[str, str]]:
    """基于正则规则快速提取偏好，返回 [(key, value), ...]"""
    found = []
    for pattern, (key, value) in PREFERENCE_MAP.items():
        if re.search(pattern, user_input):
            found.append((key, value))
    return found
```

### LLM 辅助的深度提取

```python
import requests

OLLAMA_URL   = "http://localhost:11434"
ROUTER_MODEL = "qwen2.5:3b"

def extract_preference_by_llm(user_input: str, current_profile: dict) -> dict:
    """用小模型从自然语言中提取偏好，成本低"""
    current_prefs = json.dumps(current_profile.get("preferences", {}),
                               ensure_ascii=False)

    prompt = f"""从以下用户输入中提取任何明确的偏好或技术背景信息。
以 JSON 格式返回（若无新信息则返回 {{}}）。

可提取的字段（示例）：
- language: "zh-CN" | "en-US"
- code_style: 描述
- expertise: ["Python", "Docker"]（追加到现有列表）
- communication: 描述

用户输入："{user_input}"
当前偏好：{current_prefs}

JSON 结果（只输出 JSON）："""

    resp = requests.post(f"{OLLAMA_URL}/api/generate", json={
        "model": ROUTER_MODEL,
        "prompt": prompt,
        "stream": False,
        "options": {"temperature": 0, "num_predict": 200}
    })
    text = resp.json()["response"].strip()

    # 提取 JSON 部分
    import re
    match = re.search(r'\{.*\}', text, re.DOTALL)
    if not match:
        return {}
    try:
        return json.loads(match.group())
    except json.JSONDecodeError:
        return {}
```

### 综合提取流水线

```python
def update_profile_from_input(user_id: str, user_input: str):
    """在每轮对话中自动检测并更新用户画像"""
    profile = profile_service.get(user_id)
    changed = False

    # ① 规则快速提取
    for key, value in extract_preference_by_rules(user_input):
        if profile.get("preferences", {}).get(key) != value:
            profile_service.update_preference(user_id, key, value)
            print(f"  📝 偏好更新：{key} = {value}")
            changed = True

    # ② 仅当检测到偏好触发词时，调用 LLM 深度提取
    PREF_TRIGGERS = ["我喜欢", "我偏好", "我习惯", "我不喜欢",
                     "以后", "每次", "总是", "我是", "我在"]
    if any(t in user_input for t in PREF_TRIGGERS) and not changed:
        extracted = extract_preference_by_llm(user_input, profile)

        if "expertise" in extracted:
            for skill in extracted.pop("expertise", []):
                profile_service.add_expertise(user_id, skill)

        for key, value in extracted.items():
            if value and profile.get("preferences", {}).get(key) != value:
                profile_service.update_preference(user_id, key, str(value))
                print(f"  📝 偏好更新：{key} = {value}")
```

---

## 项目上下文自动关联

```python
def detect_project_context(user_input: str, user_id: str) -> str | None:
    """根据用户输入检测当前相关项目"""
    profile = profile_service.get(user_id)
    projects = profile.get("projects", [])

    for proj in projects:
        name = proj.get("name", "").lower()
        tech = [t.lower() for t in proj.get("tech_stack", [])]
        keywords = [name] + tech

        if any(k in user_input.lower() for k in keywords):
            return proj["name"]
    return None

# 在 System Prompt 中添加当前项目上下文
def build_full_system_prompt(user_id: str, user_input: str,
                              memory_context: str) -> str:
    profile_block = profile_service.to_system_prompt_block(user_id)
    current_project = detect_project_context(user_input, user_id)
    project_note = (f"\n当前话题相关项目：{current_project}"
                    if current_project else "")

    return f"""你是用户的个人 AI 助手。

{profile_block}{project_note}

{memory_context}

请根据以上用户背景提供个性化、针对性的回答。"""
```

---

## 画像可视化（调试用）

```python
def show_profile(user_id: str = "default"):
    profile = profile_service.get(user_id)
    print(f"\n{'='*50}")
    print(f"用户：{user_id}")
    print(f"更新时间：{profile.get('updated_at', 'N/A')[:19]}")
    print("\n偏好：")
    for k, v in profile.get("preferences", {}).items():
        print(f"  {k}: {v}")
    print(f"\n技术背景：{', '.join(profile.get('expertise', []))}")
    print("\n项目：")
    for p in profile.get("projects", []):
        print(f"  [{p.get('status', '?')}] {p['name']}: {p.get('goal', '')}")
    print('='*50)


if __name__ == "__main__":
    show_profile()
```

---

## 隐私与数据管理

```python
def clear_profile(user_id: str, confirm: bool = False):
    """删除用户画像（不可恢复）"""
    if not confirm:
        print("警告：此操作不可撤销。请传入 confirm=True 确认。")
        return
    with sqlite3.connect(DB_PATH) as conn:
        conn.execute("DELETE FROM profiles WHERE user_id = ?", (user_id,))
    print(f"用户 {user_id} 的画像已删除")

def export_profile(user_id: str, output_path: str):
    """导出用户画像为 JSON 文件"""
    profile = profile_service.get(user_id)
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(profile, f, ensure_ascii=False, indent=2)
    print(f"画像已导出到：{output_path}")
```

---

## 参考链接

- [Memory 架构详解](./memory.md)
- [对话摘要与长期记忆写入](./conversation.md)
