# CFA金融工具箱 - 功能整合架构文档

## 📋 整合概述

本项目实现了6个金融分析工具的有机整合，通过统一的数据架构、API接口和用户体验，形成了一个完整的金融工具生态系统。

## 🏗️ 整合架构

### 1. 数据层整合

#### 统一数据表设计
```sql
CREATE TABLE calculations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  tool_type calculation_tool_type NOT NULL,  -- 工具类型枚举
  calculation_name TEXT NOT NULL,
  input_params JSONB NOT NULL,  -- 灵活存储各工具的输入参数
  result_data JSONB NOT NULL,   -- 灵活存储各工具的计算结果
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 工具类型枚举
CREATE TYPE calculation_tool_type AS ENUM (
  'dcf',        -- DCF估值模型
  'portfolio',  -- 投资组合优化
  'bond',       -- 债券定价分析
  'option',     -- 期权定价模型
  'relative',   -- 相对估值分析
  'var'         -- 风险价值计算
);
```

#### 数据结构优势
- **单一数据源**: 所有工具共享同一张表，简化数据管理
- **灵活扩展**: JSONB字段可以存储任意结构的数据
- **类型安全**: 枚举类型确保工具类型的一致性
- **性能优化**: 索引优化查询性能

### 2. API层整合

#### 统一API接口
```typescript
// 核心类型定义
export type ToolType = 'dcf' | 'portfolio' | 'bond' | 'option' | 'relative' | 'var';

export interface Calculation {
  id: string;
  user_id: string;
  tool_type: ToolType;
  calculation_name: string;
  input_params: any;  // 根据工具类型动态解析
  result_data: any;   // 根据工具类型动态解析
  created_at: string;
  updated_at: string;
}

// 统一CRUD操作
saveCalculation(toolType, name, params, result)    // 保存计算
getUserCalculations(toolType?, limit?)             // 获取用户计算记录
getRecentCalculations(limit?)                      // 获取最近计算
getCalculation(id)                                 // 获取单个计算
updateCalculation(id, updates)                     // 更新计算
deleteCalculation(id)                              // 删除计算
getToolUsageStats()                                // 工具使用统计
```

#### API设计原则
- **统一接口**: 所有工具使用相同的API方法
- **类型筛选**: 支持按工具类型过滤数据
- **跨工具查询**: 可以查询所有工具的计算记录
- **统计分析**: 提供工具使用频率统计

### 3. 导出功能整合

#### 通用导出函数
```typescript
// 通用Excel导出
exportToolToExcel(
  toolType: ToolType,
  calculationName: string,
  inputParams: any,
  resultData: any,
  language: 'zh' | 'en'
)

// 支持的工具类型
- DCF估值: 参数表 + 结果表 + 现金流表
- 投资组合: 资产配置表 + 组合指标表
- 债券定价: 参数表 + 结果表
- 期权定价: 参数表 + Greeks指标表
- 相对估值: 公司对比表 + 估值倍数表
- 风险价值: 参数表 + VaR指标表
```

#### 导出功能特点
- **智能格式化**: 根据工具类型自动生成合适的Excel格式
- **多语言支持**: 所有导出都支持中英文切换
- **一致体验**: 所有工具页面都有相同的导出操作

### 4. 安全层整合

#### 统一RLS策略
```sql
-- 用户可以查看自己的计算记录
CREATE POLICY "Users can view own calculations" ON calculations
  FOR SELECT TO authenticated USING (auth.uid() = user_id);

-- 用户可以创建自己的计算记录
CREATE POLICY "Users can create own calculations" ON calculations
  FOR INSERT TO authenticated WITH CHECK (auth.uid() = user_id);

-- 用户可以更新自己的计算记录
CREATE POLICY "Users can update own calculations" ON calculations
  FOR UPDATE TO authenticated USING (auth.uid() = user_id);

-- 用户可以删除自己的计算记录
CREATE POLICY "Users can delete own calculations" ON calculations
  FOR DELETE TO authenticated USING (auth.uid() = user_id);

-- 管理员可以查看所有计算记录
CREATE POLICY "Admins can view all calculations" ON calculations
  FOR SELECT TO authenticated USING (is_admin(auth.uid()));
```

## 🔄 数据流整合

### 完整数据流程
```
┌─────────────┐
│  用户输入   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  实时计算   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  结果展示   │
└──────┬──────┘
       │
       ├──────────────┐
       │              │
       ▼              ▼
┌─────────────┐  ┌─────────────┐
│  保存记录   │  │  导出文件   │
└──────┬──────┘  └─────────────┘
       │
       ▼
┌─────────────┐
│calculations │
│    表       │
└──────┬──────┘
       │
       ├──────────────┬──────────────┐
       │              │              │
       ▼              ▼              ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  查询历史   │  │  更新记录   │  │  删除记录   │
└─────────────┘  └─────────────┘  └─────────────┘
```

### 跨工具数据流
```
DCF估值 ──┐
投资组合 ──┤
债券定价 ──┼──→ calculations表 ──→ 统一查询/统计/导出
期权定价 ──┤
相对估值 ──┤
风险价值 ──┘
```

## 📊 各工具数据结构

### DCF估值模型
```typescript
input_params: {
  baseFcf: number,
  forecastGrowthRate: number,
  forecastYears: number,
  wacc: number,
  perpetualGrowthRate: number,
  currentPrice: number,
  // WACC组成参数（可选）
  riskFreeRate?: number,
  marketRiskPremium?: number,
  beta?: number,
  equityWeight?: number,
  debtWeight?: number,
  costOfDebt?: number,
  taxRate?: number
}

result_data: {
  enterpriseValue: number,
  equityValue: number,
  safetyMargin: number,
  terminalValue: number,
  terminalValuePV: number,
  forecastCashFlows: number[],
  presentValues: number[]
}
```

### 投资组合优化
```typescript
input_params: {
  assets: Array<{
    name: string,
    expectedReturn: number,
    volatility: number,
    weight: number
  }>,
  riskFreeRate: number
}

result_data: {
  expectedReturn: number,
  volatility: number,
  sharpeRatio: number
}
```

### 债券定价分析
```typescript
input_params: {
  faceValue: number,
  couponRate: number,
  ytm: number,
  yearsToMaturity: number,
  frequency: number
}

result_data: {
  price: number,
  duration: number,
  modifiedDuration: number,
  convexity: number,
  cashFlows: Array<{
    period: number,
    payment: number,
    pv: number
  }>
}
```

### 期权定价模型
```typescript
input_params: {
  optionType: 'call' | 'put',
  spotPrice: number,
  strikePrice: number,
  timeToMaturity: number,
  riskFreeRate: number,
  volatility: number,
  dividendYield: number
}

result_data: {
  price: number,
  delta: number,
  gamma: number,
  theta: number,
  vega: number,
  rho: number
}
```

### 相对估值分析
```typescript
input_params: {
  companies: Array<{
    name: string,
    price: number,
    eps: number,
    bookValue: number,
    revenue: number,
    ebitda: number
  }>
}

result_data: {
  multiples: Array<{
    company: string,
    pe: number,
    pb: number,
    ps: number,
    evEbitda: number
  }>,
  averages: {
    avgPE: number,
    avgPB: number,
    avgPS: number,
    avgEVEbitda: number
  }
}
```

### 风险价值计算
```typescript
input_params: {
  portfolioValue: number,
  expectedReturn: number,
  volatility: number,
  confidenceLevel: string,
  timeHorizon: number
}

result_data: {
  var1Day: number,
  varNDay: number,
  cvar: number
}
```

## 🎯 整合优势

### 1. 开发效率
- **代码复用**: 统一的API和导出函数减少重复代码
- **快速扩展**: 添加新工具只需定义数据结构，无需修改基础架构
- **易于维护**: 集中管理降低维护成本

### 2. 用户体验
- **一致性**: 所有工具有相同的操作流程
- **便捷性**: 可以在一个地方查看所有计算历史
- **灵活性**: 支持跨工具的数据分析和对比

### 3. 系统性能
- **查询优化**: 单表查询比多表JOIN更高效
- **索引优化**: 针对常用查询场景建立索引
- **缓存友好**: 统一的数据结构便于缓存

### 4. 数据安全
- **统一策略**: 所有工具共享相同的RLS安全策略
- **权限控制**: 用户只能访问自己的数据
- **审计追踪**: 完整的创建和更新时间记录

## 🚀 使用示例

### 保存计算结果
```typescript
import { saveCalculation } from '@/db/api';

// 保存投资组合计算
await saveCalculation(
  'portfolio',
  '我的投资组合',
  { assets: [...], riskFreeRate: 2 },
  { expectedReturn: 10, volatility: 15, sharpeRatio: 0.53 }
);
```

### 查询计算历史
```typescript
import { getUserCalculations, getRecentCalculations } from '@/db/api';

// 获取所有投资组合计算
const portfolioCalcs = await getUserCalculations('portfolio');

// 获取最近10条计算（跨所有工具）
const recentCalcs = await getRecentCalculations(10);
```

### 导出计算结果
```typescript
import { exportToolToExcel } from '@/utils/export-excel';

// 导出投资组合结果
exportToolToExcel(
  'portfolio',
  '我的投资组合',
  inputParams,
  resultData,
  'zh'
);
```

### 工具使用统计
```typescript
import { getToolUsageStats } from '@/db/api';

// 获取各工具使用次数
const stats = await getToolUsageStats();
// { dcf: 5, portfolio: 3, bond: 2, option: 1, relative: 0, var: 0 }
```

## 📈 扩展性设计

### 添加新工具的步骤
1. 在枚举中添加新工具类型
```sql
ALTER TYPE calculation_tool_type ADD VALUE 'new_tool';
```

2. 定义数据结构
```typescript
// 在types.ts中定义
interface NewToolParams { ... }
interface NewToolResult { ... }
```

3. 实现计算逻辑
```typescript
// 在utils/new-tool-calculator.ts中实现
export function calculateNewTool(params: NewToolParams): NewToolResult { ... }
```

4. 创建页面组件
```typescript
// 在pages/NewToolPage.tsx中实现
export default function NewToolPage() { ... }
```

5. 添加导出支持
```typescript
// 在export-excel.ts中添加case
case 'new_tool': { ... }
```

6. 更新路由配置
```typescript
// 在routes.tsx中添加路由
{ path: '/new-tool', element: <NewToolPage /> }
```

## 🔧 维护指南

### 数据迁移
当需要修改数据结构时：
1. 创建新的migration文件
2. 使用ALTER TABLE修改表结构
3. 编写数据迁移脚本
4. 更新TypeScript类型定义

### API更新
当需要添加新API时：
1. 在api.ts中添加新函数
2. 确保遵循统一的错误处理模式
3. 添加适当的类型注解
4. 更新API文档

### 安全策略更新
当需要修改权限时：
1. 使用DROP POLICY删除旧策略
2. 使用CREATE POLICY创建新策略
3. 测试各种权限场景
4. 更新安全文档

## 📝 总结

通过统一的数据架构、API接口和用户体验，CFA金融工具箱实现了6个金融分析工具的有机整合。这种整合不仅提高了开发效率和系统性能，还为用户提供了一致、便捷的使用体验。未来可以基于这个架构轻松扩展更多金融工具，形成更完整的金融分析生态系统。
