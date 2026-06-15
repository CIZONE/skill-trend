---
name: fetch-skills-trend
description: Generate a tech-poster style HTML leaderboard of trending Claude Code skills from GitHub. Use this skill whenever the user asks about "skill趋势", "skill排行榜", "trending skills", "最近热门技能", "最近skill趋势", "热门skill", "技能排行", or wants a ranking/leaderboard of popular skills. This skill searches GitHub, filters low-star projects, and produces a stunning sci-fi poster with weekly and monthly rankings side by side.
---

# Fetch Skills Trend - GitHub 技能趋势排行榜生成器

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

### 4. 排行榜计算

#### 4.1 热度榜（左列）：按绝对新增星数降序
直接使用搜索结果的「周期新增星数」排序，取 Top 10。分别生成周榜和月榜。

#### 4.2 新秀榜（右列）：按增速百分比降序
计算每项的增速：
```
增速 = 周期新增星数 / (总星数 - 周期新增星数) × 100%
```

按增速降序取 Top 10。增速分档决定 badge 颜色：
- `boom`: > 200%（绿色 `growth-badge boom`）
- `strong`: 100-200%（青色 `growth-badge strong`）
- `solid`: 50-100%（琥珀色 `growth-badge solid`）
- `steady`: < 50%（灰色 `growth-badge steady`）

两张榜单共享 1,000+ 最低星数门槛。

#### 4.3 「为你推荐」加权综合推荐

对每个 skill 计算加权综合得分，取 Top 3 放入「为你推荐」板块：

```
Score = (增量 / 最大增量) × W1 + (增速 / 最大增速) × W2 + (普适性 / 最大普适性) × W3

默认权重: W1 = 0.3   W2 = 0.4   W3 = 0.3
```

- **增量**：周期新增星数（归一化到 0-100）
- **增速**：百分比增长率（归一化到 0-100）
- **普适性**：总星数，作为通用性的代理指标（归一化到 0-100）

**★ 修改权重**：在 README.md 中搜索「权重配置」可找到修改方法。权重值同时标注在 `poster-template.html` 的 CSS 注释中。三个权重之和应为 1.0。

推荐卡片为全宽可点击链接，包含：排名（顶部色条）、技能名+作者、2-3 句详细描述、三项指标条形图（月增量/月增速/总星数）、综合评分。

**推荐卡片模板**：
```html
<a href="https://github.com/owner/repo" target="_blank" rel="noopener" style="text-decoration:none;color:inherit">
<div class="rec-card r1">
  <div class="rec-top">
    <div class="rec-rank">1</div>
    <div class="rec-name-row">
      <span class="rec-name">repo</span>
      <span class="rec-author">owner</span>
    </div>
  </div>
  <div class="rec-desc">详细描述 (2-3 句，突出增长驱动因素与核心价值)</div>
  <div class="rec-metrics">
    <div class="rec-metric">
      <div class="m-val">33.5K</div><div class="m-label">月增量</div>
      <div class="m-bar" style="background:var(--amber);width:100%"></div>
    </div>
    <div class="rec-metric">
      <div class="m-val">+2,600%</div><div class="m-label">月增速</div>
      <div class="m-bar" style="background:var(--green);width:100%"></div>
    </div>
    <div class="rec-metric">
      <div class="m-val">34.8K</div><div class="m-label">总星数</div>
      <div class="m-bar" style="background:var(--teal);width:100%"></div>
    </div>
  </div>
  <div class="rec-score">综合评分 <em>60.6</em></div>
</div>
</a>
```

### 5. 生成 HTML 海报

读取 `assets/poster-template.html`，将数据填入模板：

- 左侧列：**热度榜**（按新增星数），含周/月 Tab 切换
- 右侧列：**新秀榜**（按增速百分比），含周/月 Tab 切换
- 每个技能条目包含：排名、技能名称(repo)、作者(owner)、指标值（热度榜为星数，新秀榜为增速百分比）、一句话介绍
- **技能名与作者分离**：技能名(repo)左侧突出显示，作者名(owner)右侧弱化显示

**数据注入占位符**：
| 占位符 | 对应面板 | 卡片类型 |
|--------|----------|----------|
| `__HOT_WEEKLY_DATA__` | 热度榜 - 周榜 | `.info-stars`（星数） |
| `__HOT_MONTHLY_DATA__` | 热度榜 - 月榜 | `.info-stars`（星数） |
| `__RISE_WEEKLY_DATA__` | 新秀榜 - 周榜 | `.growth-badge`（增速%） |
| `__RISE_MONTHLY_DATA__` | 新秀榜 - 月榜 | `.growth-badge`（增速%） |
| `__TOTAL_COUNT__` | Header 统计 | Skills 总数 |
| `__DATE_RANGE__` | 页脚 | 数据日期范围 |

**热度榜卡片格式**：
```html
<div class="card p1">
  <div class="rank gold">01</div>
  <div class="info">
    <div class="info-row">
      <a class="info-name" href="https://github.com/owner/repo" target="_blank" rel="noopener">repo</a>
      <span class="info-author">owner</span>
    </div>
    <div class="info-desc">一句话简要介绍</div>
  </div>
  <div class="info-stars">17.0K</div>
</div>
```

**新秀榜卡片格式**（用 growth-badge 替代 info-stars）：
```html
<div class="card p1">
  <div class="rank gold">01</div>
  <div class="info">
    <div class="info-row">
      <a class="info-name" href="https://github.com/owner/repo" target="_blank" rel="noopener">repo</a>
      <span class="info-author">owner</span>
    </div>
    <div class="info-desc">一句话简要介绍，突出增长背景</div>
  </div>
  <span class="growth-badge boom">+285%</span>
</div>
```

**链接规则**：info-name 必须使用 `<a>` 标签，href 指向 `https://github.com/owner/repo`，`target="_blank" rel="noopener"`。info-author 仅为展示文本。

**介绍编写原则**：
- 控制在 30 字以内
- 热度榜：说明核心功能
- 新秀榜：除了核心功能外，可点出增长驱动因素（如「全新项目爆发」「官方背书」「垂直场景精准切入」）
- 从 WebSearch 结果摘要中提取，结合项目名推断，不要编造

### 6. 输出

将生成的 HTML 写入桌面：
```
C:\Users\CIZI\Desktop\Skill-Trend.html
```

<!-- ★ 自定义输出路径：修改上面这行即可更改文件保存位置 ★ -->

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
