nim-defect-bridge 
## 架构（文本图）

```
 iOS (SwiftUI, ios)
   DefectCaseListView + ViewModel (MVVM)
   DefectAPIClient (JWT Bearer, multipart 上传)
            |  HTTP / JSON
            v
 Flask backend (app)
   routes.py   cases / upload-log / upload-image / analyze / report / auth
   auth.py     JWT issue/verify (demo-only, HS256 + env secret)
   db.py       SQLite: defect_cases / uploads / analyses / reports
   agents/log_agents.py   —— 四智能体，各管一段、接口明确
     LogParserAgent       原始日志 -> 结构化条目（保留原文行供引用）
     FaultClassifierAgent 条目 -> 故障类别（6 类）+ 严重度（critical/high/medium/low）
     RootCauseAgent       类别+时间线 -> 可能根因 + 逐字引用的证据行
     ReportAgent          以上 -> 结构化事故报告（摘要/证据/根因/修复步骤/Runbook）
   agents/image_agent.py
     ImageAnalysisAgent   端侧优先的占位 CV 接口（见下「诚实清单」）
            |
            v
 data/  app.db + uploads/case-<id>/{log,image}/...
```

四个智能体的分工借鉴多智能体事故分析的通行做法（解析、研判、取证、出报告
彼此解耦），但实现是原创的关键词规则 + 时间线相关性分析：不调任何外部模型，
离线可测，面试时每一行都能讲清楚。证据纪律是硬约束：报告里引用的日志行
必须逐字来自解析结果，测试专门断言了这一点——不编证据，是这个项目最重要的
设计决定。

## 怎么跑

需要 Python 3.11+。

```bash
cd apple-defect-bridge
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python app.py            # 或： flask --app app run
# 后端在 http://127.0.0.1:5000
```

跑测试：

```bash
pytest -q   # 当前 16 个测试
```

## API 示例（curl）

先拿一个演示 token（其余接口都要带 `Authorization: Bearer <token>`）：

```bash
TOKEN=$(curl -s -X POST localhost:5000/auth/token \
  -H 'Content-Type: application/json' -d '{"username":"demo"}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["token"])')
```

建 case、传日志、分析、取报告：

```bash
curl -s -X POST localhost:5000/cases -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"title":"Unit A flicker + heat","device":"proto-01"}'

curl -s -X POST localhost:5000/cases/1/upload-log -H "Authorization: Bearer $TOKEN" \
  -F file=@samples/sample.log

curl -s -X POST localhost:5000/cases/1/analyze -H "Authorization: Bearer $TOKEN"
curl -s localhost:5000/cases/1/report -H "Authorization: Bearer $TOKEN"
```

主要接口：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/health` | 健康检查（无需登录） |
| POST | `/auth/token` | 发演示 JWT |
| POST / GET | `/cases`、`/cases/<id>` | 建 / 列 / 查缺陷 case |
| POST | `/cases/<id>/upload-log` | 上传诊断日志（multipart） |
| POST | `/cases/<id>/upload-image` | 上传缺陷照片（multipart） |
| POST | `/cases/<id>/analyze` | 跑四智能体 + 图像检查，分析与报告一并落库 |
| POST | `/cases/<id>/report` | 基于最新日志重新生成事故报告 |
| GET | `/cases/<id>/report` | 取最新一份落库的事故报告（JSON + Markdown） |
| GET | `/cases/<id>/analyses` | 查历史分析 |

## 报告长什么样（真实输出）

对仓库自带 `samples/sample.log`（12 行）跑一遍，`GET /cases/1/report` 返回的
Markdown 原文节选：

```markdown
# Incident report: Unit A flicker + heat

**Summary:** Reviewed 12 log lines for case 'Unit A flicker + heat':
5 errors, 4 warnings. Dominant fault category: thermal (3 fault lines).
**Overall severity:** critical
**Likely root cause:** Most likely root cause sits in the thermal subsystem:
3 of 9 fault lines fall in this category, and the first fault in the timeline
(line 1) is thermal/ERROR. Faults also appear in power, display, sensor,
network; treat those as possible downstream effects until the primary
category is cleared. (heuristic confidence: low)

## Evidence (verbatim log lines)
- line 1: `ERROR: thermal throttling detected, CPU temperature 92C`
- line 11: `ERROR: thermal shutdown imminent`

## Remediation steps
1. Check thermal path: re-run with a thermal camera and log CPU temperature over time.
2. Check display path: reseat the panel connector and re-run the panel self-test.
...

## Runbook checklist
- [ ] Confirm the device, build, and test conditions recorded on the case match the log.
- [ ] Re-run analysis on the post-fix log and confirm the dominant category clears.
- [ ] Close the case only when the root-cause category shows zero fault lines.
```

注意置信度写的是 low——热故障只占 9 条故障行的 1/3，证据强度就这么多。
报告宁可把不确定说清楚，也不把规则推断包装成模型结论。

## iOS 客户端（`ios/`）

现在是一个能在 Xcode 打开的完整 App 工程（不再是散装桩代码）：

```
ios/
  DefectBridgeApp.xcodeproj     用 Xcode 打开这个
  DefectBridgeApp/
    DefectBridgeApp.swift       @main 入口
    Models.swift                与后端 JSON 严格对齐的 Decodable 模型
    DefectAPIClient.swift       JWT 登录 / case / 上传 / 分析
    DefectCaseViewModel.swift   MVVM 状态与错误提示
    DefectCaseListView.swift    列表、新建、上传示例日志、分析结果
    SampleLog.swift             内置示例日志（与 samples/sample.log 同步）
    Info.plist                  ATS 例外：仅允许本地 http（见下）
    Assets.xcassets/
  DefectBridgeAppTests/         模型解码单元测试（真实 /analyze JSON 形状）
```

怎么跑：

```bash
# 1) 先启动后端（仓库根目录）
flask --app app run        # 或 python app.py，后端在 http://127.0.0.1:5000

# 2) 用 Xcode 打开工程，选模拟器运行
open ios/DefectBridgeApp.xcodeproj
```

App 启动后自动以 `demo-user` 登录并拉取 case 列表；可新建 case、点选后上传
内置示例日志并触发四智能体分析，界面展示总行数、ERROR/WARN 数、最常见类别、
根因 statement + heuristic 置信度 + 证据原文行，以及事故报告摘要。网络或后端
错误会弹可读提示（先确认后端在跑）。

说明与边界：

- 后端地址默认 `http://127.0.0.1:5000`（模拟器/本机）。**真机调试要把
  `DefectAPIClient.baseURL` 改成 Mac 的局域网 IP**，127.0.0.1 在手机上指手机自己。
- `Info.plist` 里配了 ATS 例外，只对 127.0.0.1 / localhost 放行 http，
  仅供本地开发；正式环境必须走 HTTPS 并删掉这个例外。
- Decodable 模型按后端真实字段写（total_lines / counts_by_* / root_cause /
  report 等），旧桩代码里那组后端从不返回的 summary/suggested_next_steps
  假字段已删除；测试 target 用真实响应形状断言了 error_count=5、
  top_category=thermal、证据行号 [1, 11]。
- token 目前只存内存，真实项目应换 Keychain；JWT 仍是演示级。
- 图像分析在 App 里只展示后端返回的占位状态（元数据检查），没有假装跑了
  任何 CV/VLM 模型。
