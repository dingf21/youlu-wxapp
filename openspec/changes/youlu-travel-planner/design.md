# Design

## Context

- 仓库 `youlu-wxapp` 为微信原生小程序工程（已配置 AppID），业务页面与数据层基本空白；OpenSpec 规划产物维护在本仓库 `openspec/`。
- 动机与范围见 `proposal.md`；行为契约见各 capability 的 delta specs。
- 约束：优先可演示 MVP；地图路网与云端同步可分阶段接入，不能阻塞本地旅程规划闭环。

## Goals / Non-Goals

**Goals:**

- 用清晰的领域模型把旅程、地点/路线、预算、交通四块能力落到小程序页面与本地持久化。
- 约定模块边界与数据关联方式，使后续可替换存储（本地 → 云开发/自建 API）而不推翻页面结构。
- 为地图与路径服务预留接口，MVP 用顺序概览满足 `place-route` 规格。

**Non-Goals:**

- 本期不定稿生产级后端 schema、鉴权体系或多人实时协作。
- 不做票务下单、实时路况导航、离线包。
- 不在本期引入跨端（App/H5）统一框架。

## Decisions

### 1. 技术栈：微信原生小程序 + 本地存储 MVP

- **选择**：沿用现有 `compileType: miniprogram`，使用 WXML/WXSS/JS（或后续升级 TypeScript），数据先用 `wx.setStorage` / 封装的 repository。
- **理由**：仓库已是原生工程；MVP 无需等后端即可验证规划流程。
- **备选**：uni-app / Taro（跨端成本高，本期不需要）；微信云开发从第一天就上（加快同步但增加环境与权限复杂度，留作下一阶段）。

### 2. 领域模型与关联

- **Trip**：`id`, `name`, `startDate`, `endDate`, `createdAt`, `updatedAt`
- **DayPlan**：由 Trip 日期范围派生的按日容器；日程条目 `DayItem` 可引用 Place 或 TransportSegment
- **Place**：`id`, `tripId`, `name`, `address?`, `lat?`, `lng?`, `order`
- **Budget**：`tripId`, `total`, `categories[]`（交通/住宿/餐饮/门票/其他）
- **Expense**：`id`, `tripId`, `category`, `amount`, `kind`（estimate|actual）, `note?`
- **TransportSegment**：`id`, `tripId`, `mode`, `fromPlaceId?`, `toPlaceId?`, `departAt?`, `arriveAt?`, `estimatedCost?`, `dayDate?`
- **理由**：四能力边界清晰，关联用 ID 引用，避免深嵌套难迁移。
- **备选**：把所有内容塞进单一 Trip JSON 大对象（实现快但难做局部更新与校验）。

### 3. 信息架构（页面）

- Tab 或首页：**我的旅程**（列表）
- **旅程详情**：概览（日期、摘要）+ 子页/分段：日程、地点与路线、交通、预算
- **创建/编辑** 旅程、地点、路段、支出的表单页
- **理由**：与用户心智「一条旅程下多块规划」一致，符合 specs 的关联关系。
- **备选**：四个独立 Tab 全局功能（弱化「旅程」主实体，不符合提案）。

### 4. 路线展示策略

- MVP：按 `Place.order` 渲染有序列表 + 可选 `map` 组件打点连线（直线/折线概览），UI 标明「路线概览，非导航」。
- 后续：接入腾讯位置服务路径规划 API，替换概览实现，规格行为保持「有序路线概览」。
- **备选**：第一期就接完整驾车/公交路径（依赖 Key、配额与审核，推迟）。

### 5. 预算与交通费用

- 交通路段的 `estimatedCost` 默认不自动写入 Expense；提供「计入交通预算」操作，生成一条 `category=交通` 的预估支出，并记录来源避免重复一键导入。
- **理由**：满足 transport-plan「避免静默重复计入」。
- **备选**：路段费用实时镜像到预算（易重复、难撤销）。

### 6. 代码组织（实现仓库）

建议目录（实现阶段创建）：

```
miniprogram/
  app.js / app.json / app.wxss
  pages/trip-list | trip-detail | trip-edit | day-plan
                  | places | transport | budget
  services/       # tripStore, placeStore, budgetStore, transportStore
  utils/          # id, date, money
  components/     # empty-state, day-tabs 等
```

## Risks / Trade-offs

- **[本地数据仅本机]** → 换机丢失；缓解：README 标明限制，预留导出 JSON / 云同步接口。
- **[地图能力与隐私]** → 坐标与定位需用户授权；缓解：地点可先纯文本，坐标可选。
- **[日期与时区]** → 小程序端用本地日历日字符串 `YYYY-MM-DD`，避免 UTC 偏移导致「少一天」。
- **[概览被当成导航]** → 文案与空态明确「非实时导航」。

## Migration Plan

1. 落地空小程序骨架与 Tab/页面路由。
2. 实现 Trip CRUD + 本地存储，再挂地点、交通、预算。
3. 演示数据可一键写入便于评审。
4. 后续若上云：为各 Store 增加 remote adapter，保持页面调用不变；旧本地数据通过导入迁移。
5. 回滚：卸载小程序或清除 Storage 即可回到空状态（无服务端脏数据）。

## Open Questions

- 是否在下一迭代直接采用微信云开发作为同步后端（不影响本期本地 MVP 任务拆分）。
- 品牌视觉（主色、Logo）是否已有规范；无则实现时用中性默认主题，可后换。
