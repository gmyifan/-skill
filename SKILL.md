---
name: video-playlist-scraper
description: |
  抓取视频平台片单并输出Excel表格。适用于爱奇艺、腾讯视频、优酷、芒果TV、B站等
  有反爬机制的视频网站。使用 Playwright 浏览器自动化绕过反爬，通过分析页面DOM结构
  动态生成抓取脚本。支持按类型（电视剧/电影/综艺/动漫）分类、按标签（独播/自制/VIP）
  筛选。当用户提到"抓取片单"、"整理视频网站内容"、"抓取独播/自制内容"、
  "爱奇艺/腾讯/优酷/芒果/B站 片单"时触发。
---

# 视频平台片单抓取 Skill

## 概述

本 Skill 引导 Claude 从视频平台网站抓取片单数据并输出为 Excel 表格。采用引导式工作流——不依赖预置脚本，而是实时分析目标网站的页面结构，动态生成抓取脚本，适应不同平台和网站结构变化。

**核心技术栈**: Python + Playwright（浏览器自动化） + Pandas/openpyxl（Excel输出）

## 前置检查

在开始前，确认环境：
- Python 3 已安装
- `playwright` 已安装（`pip3 list | grep playwright`）
- `pandas` 和 `openpyxl` 已安装
- Playwright 浏览器已安装（`python3 -m playwright install chromium`）

如有缺失，提示用户安装。

## 工作流程（5个阶段）

### 阶段1: 需求确认

通过对话确认以下信息（一次只问一个问题，根据回答追问）：

1. **目标平台**: 哪个视频网站？（爱奇艺/腾讯视频/优酷/芒果TV/B站/其他）
2. **内容类型**: 要抓取哪些类型？（电视剧/电影/综艺/动漫/全部）
3. **筛选条件**: 有什么筛选要求？（独播/自制/VIP/热播/全部）
4. **输出字段**: 需要哪些信息？（片名/类型/链接 为基础，可选：导演/主演/评分/年份/集数/简介）
5. **输出格式**: Excel 分 sheet 方式？（按类型分/全部一个 sheet）

**快速路径**: 如果用户需求明确（如"抓取爱奇艺独播片单"），可跳过部分确认，使用合理默认值。

### 阶段2: 平台探索

**首先检查参考文档**: 读取 `references/platform-patterns.md`（路径相对于本 SKILL.md 所在目录），查看是否已有该平台的结构信息。如果有且信息完整，可跳过探索直接进入阶段3。

**如果需要探索**，使用以下标准化流程：

#### 2.1 找到列表页入口
- 使用 Playwright MCP 工具或 firecrawl 访问平台首页
- 寻找各类型（电视剧/电影/综艺/动漫）的频道入口
- 识别是否有旧版列表页（通常更适合抓取，如爱奇艺的 `list.iqiyi.com`）

#### 2.2 分析内容加载机制
访问列表页后，判断数据加载方式：
- **无限滚动**: 滚动到底部触发加载（需要 Playwright 模拟滚动）
- **分页导航**: URL 中有页码参数或点击翻页按钮
- **API 接口**: 通过 XHR/Fetch 请求加载 JSON 数据（监控网络请求发现）

#### 2.3 分析 DOM 结构
用 Playwright 的 browser_snapshot 或 browser_evaluate 工具提取：
- **列表容器选择器**: 包裹所有内容条目的容器
- **列表项选择器**: 单个内容条目（如 `.qy-mod-li`）
- **标题选择器**: 片名元素及其属性（title/textContent）
- **链接选择器**: 详情页链接及其 href 属性
- **标签标识**: 独播/自制/VIP 等标签的识别方式（角标图片/CSS类名/文字）

#### 2.4 分析分页/滚动边界
- 每次加载多少条
- 总量上限
- 如何判断已到底（无新增/无更多按钮/API返回空）

#### 2.5 记录发现
将新平台的分析结果追加到 `references/platform-patterns.md`，供后续复用。

### 阶段3: 脚本生成

基于阶段2的分析结果，在用户的当前工作目录下生成 Python 抓取脚本。

#### 脚本模板结构

```python
#!/usr/bin/env python3
"""[平台名]片单抓取脚本"""

import time
import pandas as pd
from playwright.sync_api import sync_playwright

# ============ 配置 ============
CATEGORIES = {
    # "类型名": "列表页URL",
}

OUTPUT_FILE = "[平台名]片单.xlsx"
SCROLL_PAUSE = 2.0       # 每次滚动/请求间隔（秒）
MAX_SCROLLS = 50          # 安全上限
CATEGORY_PAUSE = 4.0      # 类型间间隔（秒）

USER_AGENT = (
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
    "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
)

# 浏览器内执行的JS：提取列表项数据
JS_EXTRACT = """() => {
    // 根据平台DOM结构定制
    const items = document.querySelectorAll('[列表项选择器]');
    return Array.from(items).map(item => {
        // 提取标题、链接、标签
        return { title, href, matchesFilter };
    });
}"""


def load_all_content(page):
    """加载所有内容（根据平台机制选择滚动/翻页/API）"""
    # 无限滚动模式
    prev_count = 0
    no_change_count = 0
    for i in range(MAX_SCROLLS):
        curr_count = page.locator('[列表项选择器]').count()
        print(f"  第{i+1}次滚动，已加载 {curr_count} 条", flush=True)
        if curr_count == prev_count:
            no_change_count += 1
            if no_change_count >= 3:
                break
        else:
            no_change_count = 0
        prev_count = curr_count
        page.evaluate('window.scrollTo(0, document.body.scrollHeight)')
        time.sleep(SCROLL_PAUSE)
    return prev_count


def scrape_category(page, url, category_name):
    """抓取单个类型"""
    print(f"\n{'='*50}")
    print(f"正在抓取: {category_name}")

    try:
        page.goto(url, timeout=30000, wait_until='domcontentloaded')
        time.sleep(3)
    except Exception as e:
        print(f"  页面加载失败: {e}，重试中...")
        try:
            page.goto(url, timeout=30000, wait_until='domcontentloaded')
            time.sleep(3)
        except Exception:
            print(f"  重试失败，跳过此类型")
            return []

    try:
        page.wait_for_selector('[列表项选择器]', timeout=10000)
    except Exception:
        print(f"  未找到列表项，跳过")
        return []

    total = load_all_content(page)
    print(f"  共加载 {total} 条")

    raw_items = page.evaluate(JS_EXTRACT)

    # 过滤并格式化
    results = []
    for item in raw_items:
        if not item.get('matchesFilter'):
            continue
        href = item.get('href', '')
        if href.startswith('//'):
            href = 'https:' + href
        title = item.get('title', '').strip()
        if title and href:
            results.append({
                '片名': title,
                '类型': category_name,
                '网址链接': href,
            })

    print(f"  筛选后: {len(results)} 条")
    return results


def save_to_excel(all_data, output_file):
    """保存到Excel，每个类型一个sheet"""
    with pd.ExcelWriter(output_file, engine='openpyxl') as writer:
        for category, items in all_data.items():
            df = pd.DataFrame(items) if items else pd.DataFrame(columns=['片名', '类型', '网址链接'])
            df.to_excel(writer, sheet_name=category, index=False)
        for sheet_name in writer.sheets:
            ws = writer.sheets[sheet_name]
            ws.column_dimensions['A'].width = 35
            ws.column_dimensions['B'].width = 10
            ws.column_dimensions['C'].width = 55
    print(f"\n已保存至: {output_file}")


def main():
    all_data = {}
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        context = browser.new_context(
            viewport={'width': 1920, 'height': 1080},
            user_agent=USER_AGENT,
        )
        page = context.new_page()
        try:
            for i, (name, url) in enumerate(CATEGORIES.items()):
                items = scrape_category(page, url, name)
                all_data[name] = items
                if i < len(CATEGORIES) - 1:
                    time.sleep(CATEGORY_PAUSE)
        finally:
            browser.close()

    save_to_excel(all_data, OUTPUT_FILE)

    print(f"\n{'='*50}")
    print("抓取完成!")
    total = 0
    for name, items in all_data.items():
        count = len(items)
        total += count
        print(f"  {name}: {count} 条")
    print(f"  总计: {total} 条")


if __name__ == '__main__':
    main()
```

#### 定制要点

根据平台探索结果，需要定制的部分：
1. `CATEGORIES` 字典：各类型的列表页 URL
2. `JS_EXTRACT`：DOM 选择器和数据提取逻辑
3. `load_all_content()`：加载机制（滚动/翻页/API）
4. 过滤条件：独播/自制/VIP 的判断逻辑
5. `OUTPUT_FILE`：输出文件名

#### 反爬最佳实践

所有脚本必须包含以下反爬策略：
- **User-Agent**: 使用最新的 Chrome 桌面浏览器 UA
- **请求间隔**: 滚动/翻页间隔至少 2 秒
- **类型间隔**: 不同类型之间等待 3-5 秒
- **Headless 模式**: `headless=True`
- **标准 Viewport**: 1920x1080
- **超时重试**: 页面加载失败时重试一次
- **安全上限**: 设置 MAX_SCROLLS 防止无限循环

### 阶段4: 执行抓取

1. 将生成的脚本写入用户工作目录
2. 使用 Bash 工具执行脚本：`python3 <脚本名>.py`
3. 设置足够的 timeout（建议 600000ms / 10分钟）
4. 实时观察输出日志

### 阶段5: 输出验证

脚本执行完成后：
1. 用 Python 读取 Excel 文件，展示每个 sheet 的前 5 条数据
2. 汇总各类型的数量统计
3. 如果某个类型数据为 0，告知用户可能的原因并建议重试
4. 告知用户输出文件的完整路径

## 注意事项

- 视频平台的页面结构可能随时更新，如果抓取失败，需要重新执行阶段2探索
- 部分平台可能需要登录才能看到完整列表（如 VIP 专区），提醒用户这种情况
- 每个平台的列表页总量通常有上限（如爱奇艺约 768 条/类型），告知用户这是平台限制
- 如果用户需要更多字段（评分、主演等），需要在 JS_EXTRACT 中增加提取逻辑，或进入详情页二次抓取（会显著增加时间）
