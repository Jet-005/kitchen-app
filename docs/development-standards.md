# 开发规范文档

- **项目**：厨房管理小程序
- **技术栈**：uni-app (Vue 3) + NestJS + Prisma + SQLite/PostgreSQL，**全 TypeScript**
- **状态**：所有代码必须遵循本规范。规范与实际冲突时，先改规范，再改代码。

> 本文档是项目的"宪法"。新增功能、重构、修 bug 一律按此规范执行。规范本身可随项目演进迭代，但改动需显式更新本文档并说明原因。

---

## 1. 技术栈与版本约定

| 层     | 技术                                            | 包管理 |
| ------ | ----------------------------------------------- | ------ |
| 前端   | uni-app + Vue 3 (`<script setup>`) + TypeScript | pnpm   |
| 后端   | NestJS + TypeScript                             | pnpm   |
| ORM    | Prisma                                          | —      |
| 数据库 | 开发 SQLite / 上线 PostgreSQL                   | —      |
| 校验   | class-validator + class-transformer（后端 DTO） | —      |
| 单测   | Vitest（前后端通用）                            | —      |
| Lint   | ESLint + Prettier                               | —      |
| Commit | Conventional Commits                            | —      |

**Node 版本**：固定 LTS（写在 `.nvmrc`），前后端一致。
**包管理器**：统一 `pnpm`，禁止混用 npm/yarn。

---

## 2. 仓库结构（Monorepo）

采用 pnpm workspace 单仓多包。

```
kitchen/
├── pnpm-workspace.yaml
├── package.json                 # 根脚本、共享 devDeps
├── .nvmrc
├── .editorconfig
├── .prettierrc
├── eslint.config.mjs
├── tsconfig.base.json           # 共享 TS 配置
├── packages/
│   ├── shared/                  # 前后端共享类型、枚举、常量
│   │   ├── src/
│   │   │   ├── enums/           # Role, OrderStatus, DishStatus...
│   │   │   ├── types/           # DTO 接口、业务类型
│   │   │   └── constants/       # 低库存默认阈值、状态流转规则
│   │   └── package.json
│   ├── server/                  # NestJS 后端
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   ├── src/
│   │   │   ├── modules/         # 按领域分模块
│   │   │   ├── common/          # 守卫、拦截器、装饰器、过滤器
│   │   │   └── main.ts
│   │   └── package.json
│   └── app/                     # uni-app 前端
│       ├── src/
│       │   ├── pages/
│       │   ├── components/
│       │   ├── stores/          # Pinia
│       │   ├── api/             # 后端调用封装
│       │   └── utils/
│       └── package.json
└── docs/
    └── superpowers/specs/       # 设计文档
```

**核心原则**：

- `packages/shared` 是前后端的唯一类型真相源。后端 DTO、前端 api、Prisma 生成的类型，最终都收敛到可被 shared 引用。
- 严禁在前端重新定义后端已有的业务类型（复制粘贴），必须从 shared 导入。

---

## 3. TypeScript 规范

### 3.1 严格模式

`tsconfig` 全开严格：

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### 3.2 类型规则

1. **禁止 `any`**。实在无法避免时用 `unknown` 并立即收窄，且必须加注释说明原因。
2. **禁止 `as` 断言**，除非：
   - 解析外部不可信数据（且随后校验）；
   - Prisma 类型扩展等框架要求场景。
     断言处必须注释为什么安全。
3. **禁用非空断言 `!`**。用可选链 `?.`、空值合并 `??`、显式判空。
4. **优先 `interface` 描述对象形状，`type` 描述联合/工具类型**。
5. **枚举放 shared**，前后端共用，避免魔法字符串。

```ts
// shared/enums/order.ts
export enum OrderStatus {
  Pending = "pending",
  Processing = "processing",
  Done = "done",
  Rejected = "rejected",
  Cancelled = "cancelled",
}
```

6. **函数返回值显式标注**，不依赖类型推断。

```ts
// ✅
function calcConsume(qty: number, servings: number): number { ... }
// ❌
function calcConsume(qty: number, servings: number) { ... }
```

---

## 4. 命名规范

| 对象                 | 规则                           | 示例                                  |
| -------------------- | ------------------------------ | ------------------------------------- |
| 变量、函数           | camelCase                      | `lowStockThreshold`                   |
| 类、接口、类型、枚举 | PascalCase                     | `OrderService`, `RecipeDto`           |
| 常量（真常量）       | UPPER_SNAKE_CASE               | `DEFAULT_ML_THRESHOLD`                |
| 文件（组件）         | PascalCase.vue                 | `OrderCard.vue`                       |
| 文件（其他 ts）      | PascalCase                     | `OrderStatus.ts`                      |
| 数据库表/字段        | snake_case（Prisma 用 `@map`） | `kitchen_id`                          |
| API 路径             | kebab-case 复数                | `/api/orders`, `/api/low-stock-items` |
| 枚举值               | snake_case 字符串              | `'pending'`, `'on_sale'`              |

**语义化命名**：

- 布尔变量以 `is/has/can/should` 开头：`isPublic`, `hasRecipe`, `canCancel`。
- 异步函数以动词开头：`createOrder`, `fetchInventory`。
- 避免缩写：`inventory` 不写 `inv`，`quantity` 不写 `qty`（除数学公式局部变量）。

---

## 5. Prisma 规范

### 5.1 schema 写法

- 表名、字段名用 snake_case，通过 `@@map` / `@map` 映射，**模型类本身用 PascalCase**。
- 所有业务表必须有 `kitchenId`（除 User/Kitchen）。
- 所有表带 `createdAt` / `updatedAt`（`@default(now())` / `@updatedAt`）。
- 枚举用 Prisma `enum`，与 shared 枚举值保持一致。

```prisma
model Inventory {
  id           String   @id @default(cuid())
  kitchenId    String   @map("kitchen_id")
  ingredientId String   @map("ingredient_id")
  quantity     Float
  purchaseDate DateTime @map("purchase_date")
  expireDate   DateTime @map("expire_date")
  storage      Storage

  ingredient   Ingredient @relation(fields: [ingredientId], references: [id])
  kitchen      Kitchen    @relation(fields: [kitchenId], references: [id])

  @@map("inventory")
}
```

### 5.2 迁移纪律

- **schema 改动必须生成迁移**：`pnpm prisma migrate dev --name <描述>`。
- 迁移文件一旦提交，**禁止修改**，只能新增。
- 字段重命名 = 新增字段 + 迁移数据 + 删旧字段，分多个迁移。

### 5.3 查询规范

- 复杂查询封装在 Service 层，Controller 不直接调 Prisma。
- 涉及多表关联写入（如订单完成扣库存）**必须用事务** `$transaction`。

---

## 6. NestJS 后端规范

### 6.1 模块划分（按领域）

```
src/modules/order/
├── order.module.ts
├── order.controller.ts       # 只做路由 + DTO 校验，不含业务逻辑
├── order.service.ts          # 业务逻辑
├── dto/
│   ├── create-order.dto.ts
│   └── query-order.dto.ts
└── order.service.spec.ts     # 单测紧邻
```

每个领域模块：Kitchen / Membership / Recipe / Ingredient / Inventory / Dish / Order / Notification。

### 6.2 分层职责（严格执行）

| 层         | 职责                                        | 禁止                      |
| ---------- | ------------------------------------------- | ------------------------- |
| Controller | 接收请求、调 DTO 校验、调 Service、返回结果 | 写业务逻辑、直接调 Prisma |
| Service    | 业务逻辑、事务、跨模块协作                  | 关心 HTTP、返回响应对象   |
| Prisma     | 数据访问                                    | 出现在 Controller         |

**业务规则集中在 Service**。例如"订单完成扣库存"的逻辑只存在于 `OrderService.completeOrder()`，前端和其他模块调用都走它。

### 6.3 DTO 与校验

- 所有入参用 DTO class + class-validator 装饰器。
- DTO 字段类型从 shared 复用，不重复定义。

```ts
// dto/create-order.dto.ts
import { IsArray, ValidateNested } from "class-validator";
import { Type } from "class-transformer";

class OrderItemDto {
  @IsString() dishId: string;
  @IsInt() @Min(1) quantity: number;
}

export class CreateOrderDto {
  @IsString() kitchenId: string;
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items: OrderItemDto[];
  @IsOptional() @IsString() remark?: string;
}
```

- 全局开启 `ValidationPipe` + `transform: true`，Controller 收到的已是类型安全对象。

### 6.4 错误处理

- **禁用 HTTP 状态码硬编码在业务层**。Service 抛业务异常，由全局异常过滤器转 HTTP。
- 定义统一业务异常基类 `BizException`，带错误码。

```ts
// common/exceptions/biz.exception.ts
export class BizException extends Error {
  constructor(
    public code: string,
    message: string,
  ) {
    super(message);
  }
}
// 用法
throw new BizException("KITCHEN.NOT_OWNER", "仅主理人可操作");
```

- 全局响应格式统一：

```ts
// 统一返回
{ "code": "OK", "data": {...} }
// 错误
{ "code": "KITCHEN.NOT_OWNER", "message": "仅主理人可操作" }
```

### 6.5 鉴权与权限

- JWT 守卫识别用户身份，注入 `req.user`。
- 自定义 `@Roles()` 装饰器 + `RolesGuard` 做角色检查。
- **厨房隔离守卫**：所有 `kitchenId` 相关查询，在 Service 层强制校验"当前用户是该厨房成员"，防止越权访问别的厨房数据。

```ts
// 伪代码：每次涉及 kitchenId 的操作
async assertMembership(userId: string, kitchenId: string): Promise<Membership> {
  const m = await this.prisma.membership.findFirst({ where: { userId, kitchenId } });
  if (!m) throw new BizException('KITCHEN.NOT_MEMBER', '非该厨房成员');
  return m;
}
async assertOwner(userId: string, kitchenId: string): Promise<void> {
  const m = await this.assertMembership(userId, kitchenId);
  if (m.role !== Role.Owner) throw new BizException('KITCHEN.NOT_OWNER', '仅主理人可操作');
}
```

---

## 7. uni-app 前端规范

### 7.1 组件写法

- 统一 Vue 3 `<script setup lang="ts">` + Composition API。
- 不用 Options API，不用 mixins。
- 状态管理用 Pinia（不用 Vuex）。

```vue
<script setup lang="ts">
import { ref, computed } from "vue";
import { OrderStatus } from "@kitchen/shared";

const props = defineProps<{
  status: OrderStatus;
}>();

const statusText = computed(() => {
  const map: Record<OrderStatus, string> = {
    [OrderStatus.Pending]: "待处理",
    [OrderStatus.Processing]: "备餐中",
    [OrderStatus.Done]: "已完成",
    [OrderStatus.Rejected]: "已拒绝",
    [OrderStatus.Cancelled]: "已取消",
  };
  return map[props.status];
});
</script>
```

### 7.2 API 调用封装

- 所有后端调用走 `src/api/` 封装，**禁止在组件里直接 `uni.request`**。
- 封装统一请求拦截器：注入 token、处理统一响应格式、错误码转提示。

```ts
// api/request.ts
export async function request<T>(config: RequestConfig): Promise<T> {
  const res = await uni.request({ ... });
  if (res.data.code !== 'OK') {
    uni.showToast({ title: res.data.message, icon: 'none' });
    throw new ApiError(res.data.code, res.data.message);
  }
  return res.data.data as T;
}

// api/order.ts
export const orderApi = {
  create: (dto: CreateOrderDto) => request<OrderVo>({ url: '/orders', method: 'POST', data: dto }),
};
```

### 7.3 类型共享

- 后端返回的 VO 类型从 shared 导入，前端不重定义。
- DTO 也从 shared 导入，前后端一致。

---

## 8. 业务规则实现锚点

设计文档 §7 的 12 条业务规则，实现时必须在代码中有明确落点：

| 规则                                     | 落点                                           |
| ---------------------------------------- | ---------------------------------------------- |
| 主理人固定一人、转移需先指派             | `MembershipService.transferOwner()` + DB 校验  |
| 自动扣减比例 `qty × orderQty ÷ servings` | `OrderService.calcConsume()`（纯函数，可单测） |
| FIFO 扣减（按 expireDate 升序）          | `InventoryService.deductFifo()`                |
| 无菜谱菜品不扣库存                       | `OrderService.completeOrder()` 判空            |
| 库存不足不阻断（扣到 0、标记缺料、通知） | `InventoryService.deductFifo()` 返回缺料列表   |
| 低库存阈值每食材独立                     | `Ingredient.lowStockThreshold` 字段 + 查询     |
| 菜谱公开开关                             | 查询菜谱时 `where isPublic`                    |
| 订单可拒、撤单状态流转                   | `OrderService` 状态机校验函数                  |
| 订单明细快照                             | 创建订单时写入 `OrderItem.dishSnapshot`        |
| 单位一致性提示                           | 菜谱用料录入时前端校验 + 后端校验              |

**纯函数优先**：`calcConsume` 这类无副作用的计算逻辑，写成纯函数放 shared 或 service，配单测。

---

## 9. 测试规范

### 9.1 优先级

1. **必须测**：业务规则纯函数（`calcConsume`、FIFO 扣减、状态机校验、阈值判断）。
2. **应该测**：Service 层关键流程（下单、完成扣库存、主理人转移）。
3. **可选测**：Controller、组件。

### 9.2 测试文件位置

- 单测文件紧邻被测文件：`order.service.ts` → `order.service.spec.ts`。
- 命名：`<被测文件名>.spec.ts`。

### 9.3 测试原则

- **测试行为，不测实现**：断言结果，不断言内部调用了哪个 Prisma 方法。
- 用真实数据库 schema（SQLite 内存库）跑集成测试，不大量 mock。

---

## 10. Git 与提交规范

### 10.1 Conventional Commits

```
<type>(<scope>): <subject>

type:  feat | fix | refactor | docs | test | chore | style | perf
scope: kitchen | membership | order | inventory | recipe | dish | ingredient | notification | shared | app | server
```

示例：

- `feat(order): 实现订单完成自动扣库存`
- `fix(inventory): FIFO 扣减单位不一致`
- `refactor(shared): 抽取状态机校验为纯函数`

### 10.2 分支

- `main`：可发布状态
- `dev`：开发集成
- `feat/<scope>-<描述>`：功能分支
- `fix/<描述>`：修复分支

### 10.3 提交粒度

- 一次提交一件事。下单功能别和改 ESLint 配置混在一个 commit。
- 提交前必须过 lint + 类型检查 + 测试。

---

## 11. 代码风格（Prettier + ESLint）

统一配置（根目录）：

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "endOfLine": "lf"
}
```

- 引号单引号、尾逗号、分号、2 空格缩进、行宽 100。
- import 排序交由 ESLint 规则自动处理。
- **禁止 `console.log` 进入提交**（用统一的 logger）。

---

## 12. 数据库环境切换

### 12.1 开发期 SQLite

- `.env.development`：
  ```
  DATABASE_URL="file:./dev.db"
  ```
- 零安装，git 忽略 `*.db`。

### 12.2 上线 PostgreSQL

- `.env.production`：
  ```
  DATABASE_URL="postgresql://..."
  ```
- 切换只需改连接串，Prisma 迁移分别管理两套。

### 12.3 注意事项

- **不依赖 SQLite 独有特性**（如 `PRAGMA`），保证迁移到 PG 不报错。
- 写跨库兼容的 SQL/Prisma 查询，避免特定方言。

---

## 13. 待补充

随着开发推进，以下内容会逐步补充进本规范：

- [ ] 具体 API 路由清单（RESTful 端点表）
- [ ] Pinia store 结构约定
- [ ] 组件目录组织与命名细则
- [ ] 部署流程（上线 PG + 后端托管）
- [ ] CI/CD 配置

---

## 附：决策记录

| #   | 决策                        | 理由                                          |
| --- | --------------------------- | --------------------------------------------- |
| T1  | 全 TypeScript               | 前后端共享类型，Prisma 类型贯穿，减少重复定义 |
| T2  | pnpm workspace monorepo     | shared 包被前后端共同引用，monorepo 最自然    |
| T3  | Prisma                      | 类型安全 + SQLite/PG 无缝切换，迁移管理好     |
| T4  | NestJS 分层严格             | 单人长期维护，结构清晰避免腐烂                |
| T5  | DTO + class-validator       | 入参校验集中在 DTO，类型从 shared 复用        |
| T6  | 统一响应格式 + BizException | 业务层不碰 HTTP，错误码前后端共用             |
| T7  | 前端 API 封装层             | 组件不直接请求，统一鉴权与错误处理            |
| T8  | 业务规则纯函数优先          | 可测、可复用、易推理                          |
