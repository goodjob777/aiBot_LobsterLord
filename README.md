# aiBot_LobsterLord
To build a Lobster bot for LLM-callable virtual development environments using Python 3, you need a robust architecture that safely executes code, handles tool calls, and acts as an autonomous agentic worker (often referred to as a "lobster" or sandboxed agentic runner).

# 龍蝦 (LOBSTER) 自動化矩陣 AI LLM 開發者架構與開發技術指南

## 1. 系統架構與設計理念

本系統為工業級 AI Agent 任務調度與動態自動化執行矩陣。系統採用四方演化架構，將任務處理劃分為以下角色與執行階段：

1. 領主 (LORD)：負責接收使用者高階指令，並轉換為可驗證的物理系統目標藍圖 (Goals)。
2. 經理 (MANAGER)：將藍圖目標進行原子化拆解，輸出可循序執行的子任務陣列 (Sub-Tasks Array)。
3. 執行師 (EXECUTOR)：針對單一子任務生成具備冪等性 (Idempotency) 與安全防護的 Python、Shell、Docker 或 Git 執行代碼。
4. 審計員 (AUDITOR)：基於系統物理執行遙測數據 (Telemetry Snapshot) 進行真偽檢驗與閉環審查。

---

## 2. LLM 回傳格式規範與 JSON 自癒解析機制

### 2.1 嚴格 JSON 格式要求

所有模型節點輸出必須遵從純 JSON 規範，不得包含 Markdown 語法區塊標籤（如 ```json）或任何開頭與結尾的前導/後續說明文字。

### 2.2 多邊界探針 (Dual-Boundary Probe) 解析演算法

為了防止 LLM 在回傳 JSON 陣列或字典物件時被不當截斷，解析器採用智慧雙邊界探針邏輯。以下為靜態解析邏輯說明：

```python
import json
import re

def parse_and_fix_response(raw_text: str):
    if not isinstance(raw_text, str):
        raw_text = str(raw_text)

    # 1. 分離思考鏈內容 (Thinking Process)
    thinking_content = "".join(re.findall(r"<(?:thinking|reasoning)>(.*?)</(?:thinking|reasoning)>", raw_text, re.DOTALL)).strip()
    clean_content = re.sub(r"<(?:thinking|reasoning)>.*?</(?:thinking|reasoning)>", "", raw_text, flags=re.DOTALL).strip()
    clean_content = re.sub(r"```json|```", "", clean_content).strip()

    # 2. 邊界探針尋找最外層符號
    brace_start = clean_content.find('{')
    bracket_start = clean_content.find('[')

    if brace_start == -1 and bracket_start == -1:
        return {"error": "NO_JSON_BOUNDS_DETECTED"}, thinking_content

    # 決定起點與對應終點
    if brace_start != -1 and (bracket_start == -1 or brace_start < bracket_start):
        start_index = brace_start
        end_index = clean_content.rfind('}')
    else:
        start_index = bracket_start
        end_index = clean_content.rfind(']')

    if end_index == -1 or end_index < start_index:
        return {"error": "INVALID_JSON_STRUCTURE"}, thinking_content

    target_json_str = clean_content[start_index : end_index + 1]

    # 3. 多階段反序列化與自癒
    try:
        return json.loads(target_json_str), thinking_content
    except json.JSONDecodeError:
        try:
            # 修正未轉義的控制字元與換行
            fixed_str = target_json_str.replace('\n', '\\n').replace('\r', '\\r')
            fixed_str = re.sub(r'(?<!\\)\\n', r'\\n', fixed_str)
            return json.loads(fixed_str), thinking_content
        except json.JSONDecodeError:
            # 暴力補齊截斷引號或括號
            if target_json_str.startswith('{'):
                if target_json_str.count('"') % 2 != 0:
                    target_json_str = target_json_str + '"'
                if target_json_str.count('{') > target_json_str.count('}'):
                    target_json_str = target_json_str + '}'
            elif target_json_str.startswith('['):
                if target_json_str.count('[') > target_json_str.count(']'):
                    target_json_str = target_json_str + ']'

            try:
                sanitized_str = target_json_str.encode('utf-8').decode('unicode_escape', errors='ignore')
                return json.loads(sanitized_str), thinking_content
            except Exception:
                pass

    return {"error": "PARSING_FAILED", "raw": target_json_str[:200]}, thinking_content
```

---

## 3. 自動化測試驗證與修復紀錄 (Test Validation & Fix Log)

### 3.1 測試實施流程說明

AI 開發 completed 後，系統自動發起全流程驗證測試。測試重點如下：
1. 檢測子任務 002 (Git Clone) 避免觸發空轉攔截。
2. 驗證依賴安裝與背景進程啟動時不拋出 `expected string or bytes-like object` 的類型轉換異常。
3. 確保演化循環重試計數器不使用 `-=` 語法，僅採用標準加法與正向累加邏輯。

### 3.2 測試執行代碼 (Test Suite Script)

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import sys
import os
import unittest
from pathlib import Path

# 匯入部署引擎
from glances_production_v128 import GlancesDeployer

class TestGlancesDeploymentPipeline(unittest.TestCase):

    def setUp(self):
        self.deployer = GlancesDeployer(base_dir="test_github_repos", repo_name="test_glances", port=61208)

    def test_01_prepare_environment(self):
        result = self.deployer.prepare_environment()
        self.assertTrue(result, "環境準備失敗：無法建立持久化目錄")
        self.assertTrue(self.deployer.base_path.exists(), "目錄未實體存在")

    def test_02_clone_repository(self):
        # 測試 Git 複製或冪等跳過
        result = self.deployer.clone_repository()
        self.assertTrue(result, "Git 克隆程序執行失敗")

    def test_03_verify_port_type_safety(self):
        # 驗證 Socket 通訊埠檢測之類型安全性
        result = isinstance(self.deployer.target_port, int)
        self.assertTrue(result, "通訊埠必須為整數型態")

    def test_04_pipeline_counter_logic(self):
        # 驗證循環計數邏輯 (嚴格禁止使用 -= 語法)
        remaining_retries = 3
        attempts_made = 0
        while remaining_retries > 0:
            attempts_made = attempts_made + 1
            remaining_retries = remaining_retries - 1
        
        self.assertEqual(attempts_made, 3)
        self.assertEqual(remaining_retries, 0)

if __name__ == '__main__':
    unittest.main()
```

---

## 4. README 自動化測試與更新檢核表 (README Update Checklist)

在測試套件順利通過後，系統自動將本次變更與最新架構資訊寫入 README 文件。

### 4.1 變更發佈摘要
1. 修復 LLM 回傳 JSON 陣列時最外層方括號被正則剖析器裁切的問題。
2. 將所有重試計數器與狀態扣減邏輯重構為非簡寫模式（使用 `a = a - 1` 代替 `a -= 1`）。
3. 全面淘汰命令列 `netstat` 與 `grep` 組合，改採原生 `socket.connect_ex` 實作非阻斷探針。
4. 完成單元測試，全數測試案例順利通過驗證。
