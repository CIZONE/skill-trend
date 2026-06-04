---
name: skill-trend
description: Generate a tech-poster style HTML leaderboard of trending Claude Code skills from GitHub. Use this skill whenever the user asks about "skill趋势", "skill排行榜", "trending skills", "最近热门技能", "最近skill趋势", "热门skill", "技能排行", or wants a ranking/leaderboard of popular skills. This skill searches GitHub, filters low-star projects, and produces a stunning sci-fi poster with weekly and monthly rankings side by side.
---

# Skill Trend - GitHub 技能趋势排行榜生成器

当用户询问 Claude Code 技能趋势时，自动搜索 GitHub 热门项目并生成一张科技风海报。

## 工作流程

### 1. 搜索阶段 (并行搜索)

同时执行 4 个 WebSearch，覆盖周度和月度数据：

```
WebSearch: "github trending claude code skills this week stars 2026"
WebSearch: "awesome claude code skills github most starred weekly 2026"  
WebSearch: "github trending claude code skills this month top 2026"
WebSearch: "claude code MCP skills most popular github monthly stars 2026"
```

每个搜索返回 10 个结果，合并去重后获得候选池。

### 2. 筛选阶段

对候选池进行清洗：

- **去重**：相同 repo 去重，保留 star 数最高的来源
- **最低门槛**：stars < 20 的项目剔除（用户可指定其他阈值）
- **分类标记**：标记每个项目出现在周榜(w)、月榜(m)还是两者都有(both)

### 3. 检查与补充

如果筛选后周榜或月榜不足 5 个条目，用更宽泛的查询补充搜索：
```
WebSearch: "claude code skill github stars trending"
WebSearch: "github claude skills most popular 2026"
```

去重后补入对应榜单，确保每个榜单至少 5 条。

### 4. 生成 HTML 海报

读取 `assets/poster-template.html`，将搜索到的技能数据填入模板：

- 左侧列展示**周榜 Top 10**
- 右侧列展示**月榜 Top 10**
- 每个技能条目包含：排名、名称(repo)、星标数、一句话介绍

**数据注入方式**：
1. 将周榜数据替换模板中的 `__WEEKLY_DATA__` 占位符
2. 将月榜数据替换模板中的 `__MONTHLY_DATA__` 占位符
3. 每个技能卡片的 HTML 结构参考模板中的 `<!-- SKILL_CARD_TEMPLATE -->`

**技能卡片格式**：
```html
<div class="skill-card">
  <div class="rank">#1</div>
  <div class="skill-info">
    <a class="skill-name" href="https://github.com/owner/repo" target="_blank" rel="noopener">owner/repo</a>
    <div class="skill-desc">一句话简要介绍</div>
  </div>
  <div class="skill-stars">★ 12.3K</div>
</div>
```
**链接规则**：skill-name 必须使用 `<a>` 标签，href 指向 `https://github.com/owner/repo`，`target="_blank" rel="noopener"` 在新标签页安全打开。

**介绍编写原则**：
- 控制在 30 字以内
- 说明该技能的核心功能
- 从 WebSearch 结果的摘要中提取，结合项目名推断，不要编造

### 5. 输出

将生成的 HTML 写入：
```
C:\Users\CIZI\Desktop\TEST\skill-trend-by-claude\index.html
```

父目录不存在时自动创建。写入后在对话中告知用户文件路径，让用户直接在浏览器打开。

## 设计约束

- 海报整体深色科技风背景，带有网格线和粒子光效（模板已内置 CSS）
- 移动端自动切换为上下布局（周榜在上，月榜在下）
- 标题区显示当前日期范围
- 底部标注数据来源和时间

## 搜索技巧

- 关键词尽量用英文，GitHub 相关结果更准确
- 如果搜索结果明显不相关（如搜到非 Claude Code 生态的内容），调整关键词重新搜索
- 优先保留有 skills.sh 链接或明确提到 Claude Code skills 的结果
