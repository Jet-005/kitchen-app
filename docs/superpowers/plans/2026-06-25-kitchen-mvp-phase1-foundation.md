# 厨房小程序 MVP - 阶段一：工程地基 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 搭好 pnpm monorepo 骨架，建好 shared 包（枚举 + 4 个纯业务函数，TDD 覆盖），完成全部 11 张表的 Prisma schema 与首个迁移，并让 NestJS 后端能启动、通过统一响应/异常/校验管道返回健康检查。

**Architecture:** 三包 monorepo（`@kitchen/shared`、`@kitchen/server`、`@kitchen/app`）。shared 是类型与纯函数的真相源，编译成 JS 供 server/app 引用。后端 NestJS 严格分层，全局挂 ValidationPipe + 统一响应拦截器 + 业务异常过滤器。开发库用 SQLite（零安装），上线切 PostgreSQL。

**Tech Stack:** TypeScript（全栈）、pnpm workspace、NestJS、Prisma、SQLite（开发）、Vitest。

---

## 阶段总览（本计划只实现阶段一）

| 阶段 | 内容 | 产出 |
| --- | --- | --- |
| **一（本计划）** | monorepo + shared 纯函数 + Prisma schema + NestJS 地基 | 可启动后端 + 被测纯函数 |
| 二 | 鉴权 + User/Kitchen/Membership（角色 + 厨房隔离） | 登录建圈、多厨房切换 |
| 三 | Ingredient→Inventory→Recipe→Dish→Order(自动扣减)→Notification→购物清单 | 全部后端业务 |
| 四 | uni-app 前端 | 小程序可跑 |

> **重要决策（影响 schema 写法）**：开发用 SQLite，而 **Prisma `enum` 在 SQLite 不支持**。因此 schema 中所有"枚举性质"字段一律用 `String`，权威枚举类型放 `@kitchen/shared`。这同时让 SQLite→PG 迁移零摩擦。后续阶段二的 Service 用 shared 枚举做校验。

---

## 文件结构（阶段一产出）

```
kitchen/
├── .nvmrc
├── .editorconfig
├── .prettierrc
├── .gitignore
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── eslint.config.mjs
└── packages/
    ├── shared/
    │   ├── package.json
    │   ├── tsconfig.json
    │   ├── vitest.config.ts
    │   └── src/
    │       ├── Index.ts                          # 桶导出
    │       ├── enums/Role.ts
    │       ├── enums/OrderStatus.ts
    │       ├── enums/DishStatus.ts
    │       ├── enums/Storage.ts
    │       ├── enums/NotificationType.ts
    │       ├── constants/ThresholdDefaults.ts
    │       ├── types/ApiResponse.ts
    │       ├── types/ErrorCodes.ts
    │       ├── types/Inventory.ts                # InventoryBatch 等内部类型
    │       ├── calc/CalcConsume.ts
    │       ├── calc/CalcConsume.spec.ts
    │       ├── calc/DeductFifo.ts
    │       ├── calc/DeductFifo.spec.ts
    │       ├── calc/IsLowStock.ts
    │       ├── calc/IsLowStock.spec.ts
    │       └── order/OrderTransitions.ts
    │       └── order/OrderTransitions.spec.ts
    └── server/
        ├── package.json
        ├── tsconfig.json
        ├── .env.example
        ├── prisma/schema.prisma
        └── src/
            ├── Main.ts
            ├── AppModule.ts
            ├── prisma/PrismaModule.ts
            ├── prisma/PrismaService.ts
            ├── common/exceptions/BizException.ts
            ├── common/filters/GlobalExceptionFilter.ts
            ├── common/interceptors/ResponseInterceptor.ts
            └── health/HealthController.ts
```

> **命名规范**：开发规范 §4 规定 ts 文件用 **PascalCase.ts**（如 `OrderStatus.ts`、`Main.ts`），目录用小写。NestJS 约定 `AppModule`/`Main` 也用 PascalCase 文件名，与规范一致。

---

## Task 1: 根目录脚手架（workspace + 基础配置）

**Files:**
- Create: `.nvmrc`
- Create: `.editorconfig`
- Create: `.prettierrc`
- Create: `.gitignore`
- Create: `pnpm-workspace.yaml`
- Create: `tsconfig.base.json`
- Create: `package.json`

- [ ] **Step 1: 创建 `.nvmrc`**

```
22
```

- [ ] **Step 2: 创建 `.editorconfig`**

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
```

- [ ] **Step 3: 创建 `.prettierrc`**（开发规范 §11）

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "endOfLine": "lf"
}
```

- [ ] **Step 4: 创建 `.gitignore`**

```gitignore
node_modules/
dist/
*.db
*.db-journal
.env
.env.local
.DS_Store
coverage/
*.log
```

- [ ] **Step 5: 创建 `pnpm-workspace.yaml`**

```yaml
packages:
  - "packages/*"
```

- [ ] **Step 6: 创建 `tsconfig.base.json`**（开发规范 §3.1 全开严格）

```json
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "lib": ["ES2021"],
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "exactOptionalPropertyTypes": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

- [ ] **Step 7: 创建根 `package.json`**

```json
{
  "name": "kitchen",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "build:shared": "pnpm --filter @kitchen/shared build",
    "dev:server": "pnpm build:shared && pnpm --filter @kitchen/server dev",
    "build": "pnpm -r build",
    "test": "pnpm -r test",
    "lint": "pnpm -r lint",
    "format": "prettier --write \"packages/**/*.{ts,vue,json}\""
  },
  "devDependencies": {
    "prettier": "^3.3.0",
    "typescript": "^5.5.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

- [ ] **Step 8: 验证 pnpm 可用**

Run: `pnpm -v`
Expected: 输出版本号（如 9.x）。若未安装，先 `npm i -g pnpm`。

- [ ] **Step 9: Commit**

```bash
git add .nvmrc .editorconfig .prettierrc .gitignore pnpm-workspace.yaml tsconfig.base.json package.json
git commit -m "chore: monorepo 根脚手架"
```

---

## Task 2: ESLint 扁平配置

**Files:**
- Create: `eslint.config.mjs`
- Modify: 根 `package.json`（加 devDeps）

- [ ] **Step 1: 更新根 `package.json` devDependencies**

在 `devDependencies` 中加入：

```json
"eslint": "^9.5.0",
"@typescript-eslint/eslint-plugin": "^8.0.0",
"@typescript-eslint/parser": "^8.0.0",
"eslint-config-prettier": "^9.1.0"
```

并在根 `scripts` 增加：

```json
"lint:fix": "pnpm -r exec eslint --fix ."
```

- [ ] **Step 2: 创建 `eslint.config.mjs`**（扁平配置）

```js
// @ts-check
import tseslint from '@typescript-eslint/eslint-plugin';
import tsparser from '@typescript-eslint/parser';
import prettier from 'eslint-config-prettier';

export default [
  {
    files: ['packages/**/*.ts'],
    languageOptions: {
      parser: tsparser,
      parserOptions: { ecmaVersion: 2021, sourceType: 'module' },
    },
    plugins: { '@typescript-eslint': tseslint },
    rules: {
      ...tseslint.configs.recommended.rules,
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-non-null-assertion': 'error',
      'no-console': ['error', { allow: ['warn', 'error'] }],
    },
  },
  {
    ignores: ['**/dist/**', '**/node_modules/**', '**/*.d.ts'],
  },
  prettier,
];
```

- [ ] **Step 3: 安装依赖并验证**

Run: `pnpm install`
Expected: 安装成功，无错误。

Run: `pnpm exec eslint --version`
Expected: 输出 ESLint 版本。

- [ ] **Step 4: Commit**

```bash
git add eslint.config.mjs package.json pnpm-lock.yaml
git commit -m "chore: eslint 扁平配置"
```

---

## Task 3: shared 包初始化

**Files:**
- Create: `packages/shared/package.json`
- Create: `packages/shared/tsconfig.json`
- Create: `packages/shared/vitest.config.ts`

- [ ] **Step 1: 创建 `packages/shared/package.json`**

```json
{
  "name": "@kitchen/shared",
  "version": "0.1.0",
  "private": true,
  "main": "dist/Index.js",
  "types": "dist/Index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "dev": "tsc -p tsconfig.json --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src"
  },
  "devDependencies": {
    "vitest": "^1.6.0",
    "typescript": "^5.5.0"
  }
}
```

- [ ] **Step 2: 创建 `packages/shared/tsconfig.json`**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

- [ ] **Step 3: 创建 `packages/shared/vitest.config.ts`**

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
    include: ['src/**/*.spec.ts'],
  },
});
```

- [ ] **Step 4: 安装 shared 依赖**

Run: `pnpm install`
Expected: vitest 安装成功。

- [ ] **Step 5: Commit**

```bash
git add packages/shared
git commit -m "chore(shared): 包初始化"
```

---

## Task 4: shared 枚举（角色/订单/菜品/存储/通知）

**Files:**
- Create: `packages/shared/src/enums/Role.ts`
- Create: `packages/shared/src/enums/OrderStatus.ts`
- Create: `packages/shared/src/enums/DishStatus.ts`
- Create: `packages/shared/src/enums/Storage.ts`
- Create: `packages/shared/src/enums/NotificationType.ts`

> 枚举值用 snake_case 字符串（开发规范 §4）。这些字符串必须与 Prisma `schema` 中实际存入的值完全一致。

- [ ] **Step 1: `Role.ts`**

```ts
export enum Role {
  Owner = 'owner',
  Member = 'member',
}
```

- [ ] **Step 2: `OrderStatus.ts`**

```ts
export enum OrderStatus {
  Pending = 'pending',
  Processing = 'processing',
  Done = 'done',
  Rejected = 'rejected',
  Cancelled = 'cancelled',
}
```

- [ ] **Step 3: `DishStatus.ts`**

```ts
export enum DishStatus {
  OnSale = 'on_sale',
  OffSale = 'off_sale',
}
```

- [ ] **Step 4: `Storage.ts`**

```ts
export enum Storage {
  Fridge = 'fridge',
  Freezer = 'freezer',
  Room = 'room',
}
```

- [ ] **Step 5: `NotificationType.ts`**

```ts
export enum NotificationType {
  OrderAccepted = 'order_accepted',
  OrderRejected = 'order_rejected',
  OrderDone = 'order_done',
  OrderCancelled = 'order_cancelled',
  StockShortage = 'stock_shortage',
  ExpiringSoon = 'expiring_soon',
  Expired = 'expired',
}
```

- [ ] **Step 6: 类型检查通过**

Run: `pnpm --filter @kitchen/shared exec tsc --noEmit`
Expected: 无错误。

- [ ] **Step 7: Commit**

```bash
git add packages/shared/src/enums
git commit -m "feat(shared): 业务枚举"
```

---

## Task 5: shared 常量与类型（阈值默认值 / 统一响应 / 错误码）

**Files:**
- Create: `packages/shared/src/constants/ThresholdDefaults.ts`
- Create: `packages/shared/src/types/ApiResponse.ts`
- Create: `packages/shared/src/types/ErrorCodes.ts`
- Create: `packages/shared/src/types/Inventory.ts`

- [ ] **Step 1: `ThresholdDefaults.ts`**（设计文档 §3.3，按单位预填默认阈值）

```ts
/**
 * 各计量单位下的默认低库存阈值（设计文档 §3.3）。
 * 主理人录入食材时可被覆盖。
 */
export const DEFAULT_LOW_STOCK_THRESHOLD: Readonly<Record<string, number>> = {
  个: 2,
  ml: 200,
  g: 20,
  kg: 1,
  把: 1,
  包: 1,
};

export function defaultThresholdForUnit(unit: string): number {
  return DEFAULT_LOW_STOCK_THRESHOLD[unit] ?? 1;
}
```

- [ ] **Step 2: `ApiResponse.ts`**（开发规范 §6.4 统一响应格式）

```ts
export interface ApiResponse<T> {
  code: string;
  data?: T;
  message?: string;
}

export const OK_CODE = 'OK';
```

- [ ] **Step 3: `ErrorCodes.ts`**（错误码集中定义，前后端共用）

```ts
export const ErrorCode = {
  KitchenNotFound: 'KITCHEN.NOT_FOUND',
  NotMember: 'KITCHEN.NOT_MEMBER',
  NotOwner: 'KITCHEN.NOT_OWNER',
  OrderNotFound: 'ORDER.NOT_FOUND',
  InvalidTransition: 'ORDER.INVALID_TRANSITION',
  ValidationFailed: 'VALIDATION.FAILED',
  Internal: 'INTERNAL.ERROR',
} as const;

export type ErrorCodeValue = (typeof ErrorCode)[keyof typeof ErrorCode];
```

- [ ] **Step 4: `Inventory.ts`**（纯函数用到的内部类型）

```ts
export interface InventoryBatch {
  id: string;
  quantity: number;
  expireDate: Date;
}

export interface DeductResult {
  /** 扣减后各批次剩余量（quantity 可能为 0，保留 id 以便上层置零/删除） */
  updated: InventoryBatch[];
  /** 未能扣减的缺口（>=0） */
  shortage: number;
}
```

- [ ] **Step 5: 类型检查通过**

Run: `pnpm --filter @kitchen/shared exec tsc --noEmit`
Expected: 无错误。

- [ ] **Step 6: Commit**

```bash
git add packages/shared/src/constants packages/shared/src/types
git commit -m "feat(shared): 阈值默认值与共享类型"
```

---

## Task 6: 纯函数 `calcConsume`（TDD）

> 设计文档 §7 规则 4：消耗 = RecipeItem.quantity × OrderItem.quantity ÷ Recipe.servings。

**Files:**
- Create: `packages/shared/src/calc/CalcConsume.ts`
- Test: `packages/shared/src/calc/CalcConsume.spec.ts`

- [ ] **Step 1: 写失败测试**

```ts
// CalcConsume.spec.ts
import { describe, it, expect } from 'vitest';
import { calcConsume } from './CalcConsume';

describe('calcConsume', () => {
  it('按比例计算单道菜对某食材的消耗', () => {
    // 菜谱用量 3 个鸡蛋是 2 人份；点了 4 份 → 3*4/2 = 6
    expect(calcConsume({ recipeItemQty: 3, orderItemQty: 4, servings: 2 })).toBe(6);
  });

  it('不点单消耗为 0', () => {
    expect(calcConsume({ recipeItemQty: 3, orderItemQty: 0, servings: 2 })).toBe(0);
  });

  it('servings 为 0 时抛错（非法）', () => {
    expect(() => calcConsume({ recipeItemQty: 3, orderItemQty: 4, servings: 0 })).toThrow(
      /servings/,
    );
  });

  it('支持小数结果', () => {
    // 1 个 * 1 份 / 3 人份
    expect(calcConsume({ recipeItemQty: 1, orderItemQty: 1, servings: 3 })).toBeCloseTo(
      1 / 3,
      5,
    );
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `pnpm --filter @kitchen/shared test`
Expected: FAIL，提示找不到 `calcConsume`。

- [ ] **Step 3: 实现**

```ts
// CalcConsume.ts
export interface CalcConsumeInput {
  recipeItemQty: number;
  orderItemQty: number;
  servings: number;
}

/**
 * 计算一道菜（订单明细）对某食材的消耗量。
 * 公式：recipeItemQty × orderItemQty ÷ servings（设计文档 §7 规则 4）。
 */
export function calcConsume(input: CalcConsumeInput): number {
  const { recipeItemQty, orderItemQty, servings } = input;
  if (servings <= 0) {
    throw new Error('servings must be greater than 0');
  }
  return (recipeItemQty * orderItemQty) / servings;
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `pnpm --filter @kitchen/shared test`
Expected: PASS（4 个用例全过）。

- [ ] **Step 5: Commit**

```bash
git add packages/shared/src/calc/CalcConsume.ts packages/shared/src/calc/CalcConsume.spec.ts
git commit -m "feat(shared): calcConsume 消耗量纯函数"
```

---

## Task 7: 纯函数 `deductFifo`（TDD）

> 设计文档 §7 规则 5：同 ingredient_id 多份库存按 expireDate 升序扣；规则 7：不足则扣到 0，返回缺口（缺料）。

**Files:**
- Create: `packages/shared/src/calc/DeductFifo.ts`
- Test: `packages/shared/src/calc/DeductFifo.spec.ts`

- [ ] **Step 1: 写失败测试**

```ts
// DeductFifo.spec.ts
import { describe, it, expect } from 'vitest';
import { deductFifo } from './DeductFifo';
import type { InventoryBatch } from '../types/Inventory';

const batch = (id: string, quantity: number, days: number): InventoryBatch => ({
  id,
  quantity,
  expireDate: new Date(2026, 0, days),
});

describe('deductFifo', () => {
  it('按过期日期升序先扣早过期的', () => {
    // 传入顺序故意乱序，期望按 expireDate 排序后扣
    const batches = [batch('b2', 5, 10), batch('b1', 5, 5)];
    const result = deductFifo(batches, 6);
    // 先扣 b1(过期最早) 5 → 剩 0；再扣 b2 1 → 剩 4
    expect(result.shortage).toBe(0);
    const b1 = result.updated.find((x) => x.id === 'b1');
    const b2 = result.updated.find((x) => x.id === 'b2');
    expect(b1?.quantity).toBe(0);
    expect(b2?.quantity).toBe(4);
  });

  it('库存不足时扣到 0 并返回缺口', () => {
    const batches = [batch('b1', 2, 5), batch('b2', 3, 10)];
    const result = deductFifo(batches, 10);
    expect(result.shortage).toBe(5);
    expect(result.updated.every((b) => b.quantity === 0)).toBe(true);
  });

  it('扣减量为 0 时不变更', () => {
    const batches = [batch('b1', 5, 5)];
    const result = deductFifo(batches, 0);
    expect(result.shortage).toBe(0);
    expect(result.updated[0]?.quantity).toBe(5);
  });

  it('空批次列表时全部为缺口', () => {
    const result = deductFifo([], 3);
    expect(result.shortage).toBe(3);
    expect(result.updated).toEqual([]);
  });

  it('不修改原始批次（纯函数）', () => {
    const batches = [batch('b1', 5, 5)];
    deductFifo(batches, 2);
    expect(batches[0]?.quantity).toBe(5);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `pnpm --filter @kitchen/shared test`
Expected: FAIL，找不到 `deductFifo`。

- [ ] **Step 3: 实现**

```ts
// DeductFifo.ts
import type { DeductResult, InventoryBatch } from '../types/Inventory';

/**
 * FIFO 扣减：按 expireDate 升序依次扣减，扣到 0 为止。
 * 设计文档 §7 规则 5（先过期先扣）、规则 7（不足扣到 0，缺口返回）。
 * 纯函数：不修改入参。
 */
export function deductFifo(batches: InventoryBatch[], amount: number): DeductResult {
  if (amount <= 0) {
    return { updated: batches.map((b) => ({ ...b })), shortage: 0 };
  }

  const sorted = [...batches].sort((a, b) => a.expireDate.getTime() - b.expireDate.getTime());
  let remaining = amount;
  const updated: InventoryBatch[] = [];

  for (const b of sorted) {
    if (remaining <= 0) {
      updated.push({ ...b });
      continue;
    }
    const take = Math.min(b.quantity, remaining);
    remaining -= take;
    updated.push({ ...b, quantity: b.quantity - take });
  }

  return { updated, shortage: Math.max(0, remaining) };
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `pnpm --filter @kitchen/shared test`
Expected: PASS（5 个用例全过）。

- [ ] **Step 5: Commit**

```bash
git add packages/shared/src/calc/DeductFifo.ts packages/shared/src/calc/DeductFifo.spec.ts
git commit -m "feat(shared): deductFifo 先过期先扣纯函数"
```

---

## Task 8: 纯函数 `isLowStock`（TDD）

> 设计文档 §7 规则 8：库存量 <= 该食材低库存阈值。

**Files:**
- Create: `packages/shared/src/calc/IsLowStock.ts`
- Test: `packages/shared/src/calc/IsLowStock.spec.ts`

- [ ] **Step 1: 写失败测试**

```ts
// IsLowStock.spec.ts
import { describe, it, expect } from 'vitest';
import { isLowStock } from './IsLowStock';

describe('isLowStock', () => {
  it('等于阈值算低库存', () => {
    expect(isLowStock(2, 2)).toBe(true);
  });
  it('低于阈值算低库存', () => {
    expect(isLowStock(1, 2)).toBe(true);
  });
  it('高于阈值不算', () => {
    expect(isLowStock(3, 2)).toBe(false);
  });
  it('阈值为 0 时仅 0 及以下算低', () => {
    expect(isLowStock(0, 0)).toBe(true);
    expect(isLowStock(1, 0)).toBe(false);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `pnpm --filter @kitchen/shared test`
Expected: FAIL。

- [ ] **Step 3: 实现**

```ts
// IsLowStock.ts
/**
 * 设计文档 §7 规则 8：库存量 <= 食材低库存阈值即低库存。
 */
export function isLowStock(quantity: number, threshold: number): boolean {
  return quantity <= threshold;
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `pnpm --filter @kitchen/shared test`
Expected: PASS。

- [ ] **Step 5: Commit**

```bash
git add packages/shared/src/calc/IsLowStock.ts packages/shared/src/calc/IsLowStock.spec.ts
git commit -m "feat(shared): isLowStock 阈值判断纯函数"
```

---

## Task 9: 纯函数订单状态机 `canTransition`（TDD）

> 设计文档 §3.2⑨ + §7 规则 9。状态流转：pending→{processing, rejected, cancelled}；processing→{done, cancelled}；done/rejected/cancelled 为终态。

**Files:**
- Create: `packages/shared/src/order/OrderTransitions.ts`
- Test: `packages/shared/src/order/OrderTransitions.spec.ts`

- [ ] **Step 1: 写失败测试**

```ts
// OrderTransitions.spec.ts
import { describe, it, expect } from 'vitest';
import { canTransition, assertTransition } from './OrderTransitions';
import { OrderStatus } from '../enums/OrderStatus';

describe('order transitions', () => {
  it('pending 可接单/拒单/撤单', () => {
    expect(canTransition(OrderStatus.Pending, OrderStatus.Processing)).toBe(true);
    expect(canTransition(OrderStatus.Pending, OrderStatus.Rejected)).toBe(true);
    expect(canTransition(OrderStatus.Pending, OrderStatus.Cancelled)).toBe(true);
  });
  it('processing 可完成/撤单', () => {
    expect(canTransition(OrderStatus.Processing, OrderStatus.Done)).toBe(true);
    expect(canTransition(OrderStatus.Processing, OrderStatus.Cancelled)).toBe(true);
  });
  it('终态不可再流转', () => {
    expect(canTransition(OrderStatus.Done, OrderStatus.Processing)).toBe(false);
    expect(canTransition(OrderStatus.Rejected, OrderStatus.Pending)).toBe(false);
    expect(canTransition(OrderStatus.Cancelled, OrderStatus.Pending)).toBe(false);
  });
  it('非法流转抛错', () => {
    expect(() => assertTransition(OrderStatus.Done, OrderStatus.Pending)).toThrow();
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run: `pnpm --filter @kitchen/shared test`
Expected: FAIL。

- [ ] **Step 3: 实现**

```ts
// OrderTransitions.ts
import { OrderStatus } from '../enums/OrderStatus';
import { ErrorCode } from '../types/ErrorCodes';

const TRANSITIONS: Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>> = {
  [OrderStatus.Pending]: [OrderStatus.Processing, OrderStatus.Rejected, OrderStatus.Cancelled],
  [OrderStatus.Processing]: [OrderStatus.Done, OrderStatus.Cancelled],
  [OrderStatus.Done]: [],
  [OrderStatus.Rejected]: [],
  [OrderStatus.Cancelled]: [],
};

export function canTransition(from: OrderStatus, to: OrderStatus): boolean {
  return TRANSITIONS[from]?.includes(to) ?? false;
}

export function assertTransition(from: OrderStatus, to: OrderStatus): void {
  if (!canTransition(from, to)) {
    throw new Error(`${ErrorCode.InvalidTransition}: ${from} -> ${to}`);
  }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `pnpm --filter @kitchen/shared test`
Expected: PASS。

- [ ] **Step 5: Commit**

```bash
git add packages/shared/src/order
git commit -m "feat(shared): 订单状态机校验纯函数"
```

---

## Task 10: shared 桶导出 + 构建验证

**Files:**
- Create: `packages/shared/src/Index.ts`

- [ ] **Step 1: 创建桶导出**

```ts
// Index.ts
export * from './enums/Role';
export * from './enums/OrderStatus';
export * from './enums/DishStatus';
export * from './enums/Storage';
export * from './enums/NotificationType';

export * from './constants/ThresholdDefaults';
export * from './types/ApiResponse';
export * from './types/ErrorCodes';
export * from './types/Inventory';

export * from './calc/CalcConsume';
export * from './calc/DeductFifo';
export * from './calc/IsLowStock';
export * from './order/OrderTransitions';
```

- [ ] **Step 2: 全量测试 + 构建**

Run: `pnpm --filter @kitchen/shared test`
Expected: 全部 PASS。

Run: `pnpm --filter @kitchen/shared build`
Expected: 生成 `packages/shared/dist/Index.js` 等文件，无报错。

- [ ] **Step 3: 验证 dist 产物**

Run: `pnpm --filter @kitchen/shared exec node -e "console(require('./dist/Index.js').OK_CODE)"` 改为：
Run: `pnpm --filter @kitchen/shared exec node -e "const s=require('./dist/Index.js'); console.log(s.OK_CODE, s.calcConsume({recipeItemQty:3,orderItemQty:4,servings:2}))"`
Expected: 输出 `OK 6`。

- [ ] **Step 4: Commit**

```bash
git add packages/shared/src/Index.ts
git commit -m "feat(shared): 桶导出与构建验证"
```

---

## Task 11: server 包初始化 + NestJS 引导

**Files:**
- Create: `packages/server/package.json`
- Create: `packages/server/tsconfig.json`
- Create: `packages/server/.env.example`
- Create: `packages/server/src/Main.ts`
- Create: `packages/server/src/AppModule.ts`

- [ ] **Step 1: 创建 `packages/server/package.json`**

```json
{
  "name": "@kitchen/server",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "nest start --watch",
    "build": "nest build",
    "start": "node dist/Main.js",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "test": "vitest run",
    "lint": "eslint src"
  },
  "dependencies": {
    "@kitchen/shared": "workspace:*",
    "@nestjs/common": "^10.3.0",
    "@nestjs/core": "^10.3.0",
    "@nestjs/platform-express": "^10.3.0",
    "@prisma/client": "^5.16.0",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.1",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.1"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.4.0",
    "prisma": "^5.16.0",
    "typescript": "^5.5.0",
    "vitest": "^1.6.0",
    "ts-node": "^10.9.0"
  }
}
```

- [ ] **Step 2: 创建 `packages/server/tsconfig.json`**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "module": "CommonJS",
    "target": "ES2021"
  },
  "include": ["src"]
}
```

- [ ] **Step 3: 创建 `packages/server/.env.example`**

```env
DATABASE_URL="file:./dev.db"
PORT=3000
```

- [ ] **Step 4: 创建 `packages/server/src/Main.ts`**（挂全局管道/过滤器/拦截器，占位先引用待建类）

```ts
// Main.ts
import 'reflect-metadata';
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './AppModule';
import { GlobalExceptionFilter } from './common/filters/GlobalExceptionFilter';
import { ResponseInterceptor } from './common/interceptors/ResponseInterceptor';

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(
    new ValidationPipe({
      transform: true,
      whitelist: true,
      forbidNonWhitelisted: true,
    }),
  );
  app.useGlobalFilters(new GlobalExceptionFilter());
  app.useGlobalInterceptors(new ResponseInterceptor());
  const port = Number(process.env['PORT'] ?? 3000);
  await app.listen(port);
}

bootstrap();
```

- [ ] **Step 5: 创建最小 `AppModule.ts`**（先空，后续 Task 补控制器/模块）

```ts
// AppModule.ts
import { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/PrismaModule';
import { HealthController } from './health/HealthController';

@Module({
  imports: [PrismaModule],
  controllers: [HealthController],
})
export class AppModule {}
```

> 注：此 Task 引用了尚未创建的 `PrismaModule`/`HealthController`/过滤器/拦截器，下面 Task 12-16 会补齐。先创建本 Task 的文件，**本 Task 末尾不要求编译通过**（待 Task 16 统一联调）。

- [ ] **Step 6: 安装依赖**

Run: `pnpm install`
Expected: NestJS/Prisma 安装成功。

- [ ] **Step 7: Commit**

```bash
git add packages/server/package.json packages/server/tsconfig.json packages/server/.env.example packages/server/src/Main.ts packages/server/src/AppModule.ts pnpm-lock.yaml
git commit -m "chore(server): 包初始化与 NestJS 引导"
```

---

## Task 12: Prisma schema（全部 11 表）

> 所有枚举字段用 `String`（SQLite 不支持 Prisma enum），值由 shared 枚举约束。所有业务表带 `kitchenId`（除 User/Kitchen）。所有表带 `createdAt`/`updatedAt`。

**Files:**
- Create: `packages/server/prisma/schema.prisma`

- [ ] **Step 1: 创建 schema.prisma**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model User {
  id             String   @id @default(cuid())
  nickname       String
  avatar         String?
  openid         String?  @unique
  pointsBalance  Int      @default(0) // [预留] MVP=0
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")

  ownedKitchens Kitchen[]    @relation("OwnedKitchens")
  memberships   Membership[]
  orders        Order[]
  notifications Notification[]

  @@map("user")
}

model Kitchen {
  id           String   @id @default(cuid())
  name         String
  ownerUserId  String   @map("owner_user_id") // 主理人固定一人
  inviteCode   String   @unique @map("invite_code")
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")

  owner        User         @relation("OwnedKitchens", fields: [ownerUserId], references: [id])
  memberships  Membership[]
  ingredients  Ingredient[]
  inventories  Inventory[]
  recipes      Recipe[]
  dishes       Dish[]
  orders       Order[]
  notifications Notification[]

  @@map("kitchen")
}

model Membership {
  id        String   @id @default(cuid())
  kitchenId String   @map("kitchen_id")
  userId    String   @map("user_id")
  role      String   // 'owner' | 'member'（shared.Role）
  joinedAt  DateTime @default(now()) @map("joined_at")

  kitchen   Kitchen @relation(fields: [kitchenId], references: [id])
  user      User    @relation(fields: [userId], references: [id])

  @@unique([kitchenId, userId])
  @@map("membership")
}

model Ingredient {
  id                 String  @id @default(cuid())
  kitchenId          String  @map("kitchen_id")
  name               String
  unit               String
  category           String?
  defaultShelfLife   Int?    @map("default_shelf_life") // 天
  lowStockThreshold  Float   @default(1) @map("low_stock_threshold")

  kitchen     Kitchen     @relation(fields: [kitchenId], references: [id])
  inventories Inventory[]
  recipeItems RecipeItem[]

  @@map("ingredient")
}

model Inventory {
  id           String   @id @default(cuid())
  kitchenId    String   @map("kitchen_id")
  ingredientId String   @map("ingredient_id")
  quantity     Float
  purchaseDate DateTime @map("purchase_date")
  expireDate   DateTime @map("expire_date")
  storage      String   // 'fridge'|'freezer'|'room'（shared.Storage）
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")

  kitchen    Kitchen    @relation(fields: [kitchenId], references: [id])
  ingredient Ingredient @relation(fields: [ingredientId], references: [id])

  @@map("inventory")
}

model Recipe {
  id         String   @id @default(cuid())
  kitchenId  String   @map("kitchen_id")
  name       String
  steps      String?
  servings   Int      @default(1)
  coverImage String?  @map("cover_image")
  isPublic   Boolean  @default(true) @map("is_public")
  createdAt  DateTime @default(now()) @map("created_at")
  updatedAt  DateTime @updatedAt @map("updated_at")

  kitchen    Kitchen      @relation(fields: [kitchenId], references: [id])
  recipeItems RecipeItem[]
  dishes     Dish[]

  @@map("recipe")
}

model RecipeItem {
  id            String  @id @default(cuid())
  recipeId      String  @map("recipe_id")
  ingredientId  String  @map("ingredient_id")
  quantity      Float
  remark        String?

  recipe      Recipe     @relation(fields: [recipeId], references: [id])
  ingredient  Ingredient @relation(fields: [ingredientId], references: [id])

  @@map("recipe_item")
}

model Dish {
  id          String   @id @default(cuid())
  kitchenId   String   @map("kitchen_id")
  name        String
  recipeId    String?  @map("recipe_id") // 可空：有菜无谱
  points      Int      @default(0) // [预留] MVP=0
  status      String   @default("on_sale") // shared.DishStatus
  coverImage  String?  @map("cover_image")
  description String?
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  kitchen    Kitchen      @relation(fields: [kitchenId], references: [id])
  recipe     Recipe?      @relation(fields: [recipeId], references: [id])
  orderItems OrderItem[]

  @@map("dish")
}

model Order {
  id         String   @id @default(cuid())
  kitchenId  String   @map("kitchen_id")
  userId     String   @map("user_id")
  status     String   @default("pending") // shared.OrderStatus
  totalPoints Int     @default(0) // [预留] MVP=0
  remark     String?
  createdAt  DateTime @default(now()) @map("created_at")
  updatedAt  DateTime @updatedAt @map("updated_at")

  kitchen Kitchen      @relation(fields: [kitchenId], references: [id])
  user    User         @relation(fields: [userId], references: [id])
  items   OrderItem[]

  @@map("order")
}

model OrderItem {
  id           String @id @default(cuid())
  orderId      String @map("order_id")
  dishId       String @map("dish_id")
  dishSnapshot Json   @map("dish_snapshot") // 下单时快照
  quantity     Int

  order Order @relation(fields: [orderId], references: [id])
  dish  Dish  @relation(fields: [dishId], references: [id])

  @@map("order_item")
}

model Notification {
  id            String   @id @default(cuid())
  kitchenId     String   @map("kitchen_id")
  targetUserId  String   @map("target_user_id")
  type          String   // shared.NotificationType
  content       String
  refType       String?  @map("ref_type")
  refId         String?  @map("ref_id")
  isRead        Boolean  @default(false) @map("is_read")
  createdAt     DateTime @default(now()) @map("created_at")

  kitchen Kitchen @relation(fields: [kitchenId], references: [id])
  user    User    @relation("NotificationTarget", fields: [targetUserId], references: [id])

  @@map("notification")
}
```

- [ ] **Step 2: 生成 Client + 首个迁移**

Run: `pnpm --filter @kitchen/server exec prisma generate`
Expected: 生成 Prisma Client。

> 迁移命令前需 `.env`。复制：在仓库根执行创建 `packages/server/.env`（内容同 `.env.example`）。该文件已被 gitignore，不提交。

Run: `cp packages/server/.env.example packages/server/.env`（Windows Git Bash）
Run: `pnpm --filter @kitchen/server exec prisma migrate dev --name init`
Expected: 生成 `packages/server/prisma/migrations/<时间戳>_init/` 与 `packages/server/dev.db`。

- [ ] **Step 3: 验证表已建**

Run: `pnpm --filter @kitchen/server exec prisma db pull --print | head -5`
Expected: 输出含 `model User` 等（确认 schema 与库一致）。

- [ ] **Step 4: Commit**

```bash
git add packages/server/prisma/schema.prisma packages/server/prisma/migrations
git commit -m "feat(server): Prisma schema 全量模型与首个迁移"
```

---

## Task 13: PrismaService + PrismaModule

**Files:**
- Create: `packages/server/src/prisma/PrismaService.ts`
- Create: `packages/server/src/prisma/PrismaModule.ts`

- [ ] **Step 1: `PrismaService.ts`**

```ts
// PrismaService.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit(): Promise<void> {
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

- [ ] **Step 2: `PrismaModule.ts`**（全局导出，供各领域模块注入）

```ts
// PrismaModule.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './PrismaService';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

- [ ] **Step 3: Commit**

```bash
git add packages/server/src/prisma
git commit -m "feat(server): PrismaService 全局模块"
```

---

## Task 14: BizException（统一业务异常）

> 开发规范 §6.4：业务层抛 BizException 带错误码，不碰 HTTP。

**Files:**
- Create: `packages/server/src/common/exceptions/BizException.ts`

- [ ] **Step 1: 创建 BizException**

```ts
// BizException.ts
import { HttpException, HttpStatus } from '@nestjs/common';
import type { ErrorCodeValue } from '@kitchen/shared';

/**
 * 业务异常基类。Service 抛出它，由 GlobalExceptionFilter 统一转响应。
 * 错误码来自 @kitchen/shared 的 ErrorCode（前后端共用）。
 */
export class BizException extends HttpException {
  constructor(
    public readonly bizCode: ErrorCodeValue | string,
    message: string,
    status: HttpStatus = HttpStatus.BAD_REQUEST,
  ) {
    super({ code: bizCode, message }, status);
  }
}
```

- [ ] **Step 2: 类型检查**

Run: `pnpm --filter @kitchen/server exec tsc --noEmit -p tsconfig.json`
Expected: 无错误。

- [ ] **Step 3: Commit**

```bash
git add packages/server/src/common/exceptions/BizException.ts
git commit -m "feat(server): BizException 业务异常"
```

---

## Task 15: 全局异常过滤器 + 响应拦截器

**Files:**
- Create: `packages/server/src/common/filters/GlobalExceptionFilter.ts`
- Create: `packages/server/src/common/interceptors/ResponseInterceptor.ts`

- [ ] **Step 1: `GlobalExceptionFilter.ts`**（把 BizException 与未知异常统一成 `{code,message}`）

```ts
// GlobalExceptionFilter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, Logger } from '@nestjs/common';
import type { Response } from 'express';
import { ErrorCode } from '@kitchen/shared';
import { BizException } from '../exceptions/BizException';

interface BizBody {
  code: string;
  message: string;
}

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const res = host.switchToHttp().getResponse<Response>();

    if (exception instanceof BizException) {
      const body = exception.getResponse() as BizBody;
      res.status(exception.getStatus()).json(body);
      return;
    }

    if (exception instanceof HttpException) {
      const body = exception.getResponse();
      const message = typeof body === 'string' ? body : (body as BizBody).message ?? 'error';
      res.status(exception.getStatus()).json({ code: ErrorCode.ValidationFailed, message });
      return;
    }

    this.logger.error(exception);
    res.status(500).json({ code: ErrorCode.Internal, message: '服务器内部错误' });
  }
}
```

- [ ] **Step 2: `ResponseInterceptor.ts`**（统一成功返回 `{code:'OK', data}`）

```ts
// ResponseInterceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { map, Observable } from 'rxjs';
import { OK_CODE } from '@kitchen/shared';

@Injectable()
export class ResponseInterceptor<T> implements NestInterceptor<T, { code: string; data: T }> {
  intercept(
    _context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<{ code: string; data: T }> {
    return next.handle().pipe(map((data) => ({ code: OK_CODE, data })));
  }
}
```

- [ ] **Step 3: 类型检查**

Run: `pnpm --filter @kitchen/server exec tsc --noEmit -p tsconfig.json`
Expected: 无错误。

- [ ] **Step 4: Commit**

```bash
git add packages/server/src/common
git commit -m "feat(server): 全局异常过滤器与统一响应拦截器"
```

---

## Task 16: HealthController + 启动联调

**Files:**
- Create: `packages/server/src/health/HealthController.ts`

- [ ] **Step 1: 创建 `HealthController.ts`**

```ts
// HealthController.ts
import { Controller, Get } from '@nestjs/common';

@Controller('health')
export class HealthController {
  @Get()
  check(): { status: string; time: string } {
    return { status: 'ok', time: new Date().toISOString() };
  }
}
```

- [ ] **Step 2: 全量类型检查 + 构建 shared**

Run: `pnpm --filter @kitchen/shared build`
Expected: 成功。

Run: `pnpm --filter @kitchen/server exec tsc --noEmit -p tsconfig.json`
Expected: 无错误。

- [ ] **Step 3: 启动后端（后台）**

Run（后台）: `pnpm dev:server`
Expected: 日志出现 `Nest application successfully started`，监听 3000。

- [ ] **Step 4: 验证统一响应格式**

Run: `curl -s http://localhost:3000/health`
Expected: 输出 `{"code":"OK","data":{"status":"ok","time":"..."}}`。

- [ ] **Step 5: 验证校验/异常路径（造一个 404）**

Run: `curl -s http://localhost:3000/not-found`
Expected: 输出形如 `{"code":"VALIDATION.FAILED","message":"Cannot GET /not-found"}` 或 Nest 默认 404 体被过滤器包裹为 `{code,message}`。若返回 HTML/默认体，说明 `NotFoundException` 未被 `@Catch()` 过滤——这属正常（Nest 的 NotFoundException 走 HttpException 分支，应输出 JSON）。确认是 JSON 即通过。

- [ ] **Step 6: 关停后台服务后 Commit**

```bash
git add packages/server/src/health/HealthController.ts
git commit -m "feat(server): 健康检查端点与统一响应联调"
```

---

## 自检（Self-Review 记录）

**1. Spec 覆盖（阶段一相关）：**
- 数据模型 11 表 → Task 12 全覆盖（User/Kitchen/Membership/Ingredient/Inventory/Recipe/RecipeItem/Dish/Order/OrderItem/Notification）。
- 业务纯函数（设计 §8 落点）：calcConsume(T6)、deductFifo(T7)、isLowStock(T8)、状态机(T9) 全部 TDD。
- 统一响应 + BizException（开发规范 §6.4）→ T14/T15。
- 开发规范文件命名 PascalCase.ts → 全程遵循。
- 枚举放 shared（规范 §3.2）→ T4；SQLite 不支持 Prisma enum 故 schema 用 String（计划开头已声明决策）。

**2. 阶段一不覆盖的（留给后续阶段计划）：** 鉴权(JWT)、User/Kitchen/Membership 的 Service/Controller、各领域模块、uni-app 前端。这些会在阶段二/三/四计划中各自完整展开。

**3. 占位符扫描：** 无 TBD/TODO；每个代码步骤均给出完整代码。

**4. 类型一致性：** `calcConsume` 入参字段、`InventoryBatch`/`DeductResult`、`OrderStatus` 枚举值、`ErrorCode` 键名在引用处均一致。`dishSnapshot` 用 Prisma `Json`，与设计 §3.2⑩ 一致。
