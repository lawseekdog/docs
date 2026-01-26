# 知识库种子数据白名单（建议）

目标：让种子库“更全、更新、可审计”，同时尽量降低噪声（避免抓到转载/自媒体/培训稿）。

## 1) 案例来源（优先官方）

- 最高人民法院：`court.gov.cn`
- 最高人民检察院：`spp.gov.cn`
- 省级高院（示例）：`gdcourts.gov.cn`、`fjcourt.gov.cn`、`hncourt.gov.cn`
- 其他官方司法宣传站点（可选）：`chinacourt.org`

## 2) 法条/司法解释来源（优先权威数据库）

- 国家法律法规数据库：`flk.npc.gov.cn`
- 全国人大：`npc.gov.cn`
- 中国政府网：`gov.cn`
- 司法部：`moj.gov.cn`
- 司法解释/法院动态：`court.gov.cn`

## 3) Tavily（搜索抓取）使用建议

推荐只用“白名单 include_domains”，把 `TAVILY_INCLUDE_DOMAINS` 配成以上域名集合；不要放泛域名或新闻聚合站。

示例（更偏“新”）：
- `TAVILY_LAW_QUERIES="司法解释 2025 发布,行政法规 2025 发布,部门规章 2025 发布"`
- `TAVILY_CASE_QUERIES="最高人民法院 典型案例 2025,最高人民检察院 典型案例 2025,高院 典型案例 2025"`

## 4) 裁判文书网（wenshu）说明

项目只支持“正常访问”方式尝试（通过 Tavily 抓取网页内容），不做任何反爬绕过；抓到的很多会是壳页面/列表页（已在抽取侧降噪为 `doc_type=other`）。

如需稳定、可规模化的裁判文书全文，建议走：
- 授权数据源 / 合规 API
- 或离线下载（人工/批量）后作为本地种子入库（seed-first）

