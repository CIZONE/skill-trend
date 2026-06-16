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
| `__RECOMMENDED_DATA__` | 为你推荐 | 3 张 `.rec-card` 推荐卡片 |
| `__TOTAL_COUNT__` | Header 统计 | Skills 生态总数 |
| `__LISTED_COUNT__` | Header 统计 | 上榜项目数（周榜+月榜去重） |
| `__DATE_RANGE__` | 页脚 | 数据日期范围 |
| `__INSIGHTS_DATA__` | 底部趋势总结 | 3 张 `.ins-card` 趋势卡片 |

**★ 动态计算说明**：
- `__LISTED_COUNT__`：对周榜和月榜中出现的所有项目按 `owner/repo` 去重后计数，不是固定值。例如周榜 8 个 + 月榜 10 个，去重后可能是 14。
- `__INSIGHTS_DATA__`：基于本轮搜索结果中的实际数据分布，提炼 3 个核心趋势。每个趋势用 1 个 `.ins-card` 呈现，包含趋势标签(`.ins-label`)、标题(`.ins-title`)和描述(`.ins-desc`)。描述应引用榜上具体项目名和数字增强说服力，不要编造不存在的趋势。模板中已注释掉完整的卡片 HTML 结构供参考。

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

### 5.5 动态内容生成（Agent 必须执行）

#### 上榜项目计数

`__LISTED_COUNT__` 不能写死。Agent 必须：
1. 收集周榜和月榜中出现的所有项目
2. 按 `owner/repo` 去重（同一项目同时出现在周榜和月榜只算 1 次）
3. 将去重后的总数填入 `__LISTED_COUNT__`

#### 趋势总结

`__INSIGHTS_DATA__` 不能写死。Agent 必须基于本轮搜索结果提炼 3 个核心趋势，每个趋势写成一张 `.ins-card`：

```html
<div class="ins-card">
  <div class="ins-icon a|b|c"><svg>...</svg></div>
  <div class="ins-label a|b|c">趋势标签（4-6字）</div>
  <div class="ins-title">趋势标题（8-15字）</div>
  <div class="ins-desc">趋势描述（2-4句），引用榜上具体项目名和数字增强说服力。基于搜索结果中的实际数据分布总结，不编造不存在的趋势。</div>
</div>
```

三张卡片分别使用 `ins-icon a / ins-label a`、`ins-icon b / ins-label b`、`ins-icon c / ins-label c`（颜色主题：a=琥珀金, b=绿色, c=青色）。

**趋势提炼原则**：
- 观察榜上项目的共性（哪些品类集中出现？增速分布有何规律？）
- 引用具体项目名和数字（如「codegraph 月增 +2,527%」「mattpocock/skills 月增 71K」）
- 结合 ecosystem 背景（Skills 总数、增长阶段等）
- 每个趋势一个角度，三个趋势合起来覆盖本轮数据的主要信号

### 6. 输出

将生成的 HTML 写入配置的输出路径。Agent 执行时按以下优先级确定路径：
1. 如果 `<OUTPUT_PATH>` 已被用户替换为具体路径，使用该路径
2. 否则回退到默认路径：`~/Skills-Trend.html`（`~` 指本技能文件夹根目录，即 `SKILL.md` 所在目录）

```
<OUTPUT_PATH>
```

<!-- ★ 配置输出路径：将上面 <OUTPUT_PATH> 替换为你的目标路径，例如 D:\reports\Skills-Trend.html ★ -->

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
