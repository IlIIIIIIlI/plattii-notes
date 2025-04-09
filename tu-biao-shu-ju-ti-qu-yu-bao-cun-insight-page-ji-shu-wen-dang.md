# 图表数据提取与保存insight page技术文档

### 概述

本文档详细介绍如何实现从聊天消息中提取图表数据并将其保存到 Insights 面板的流程。我们使用 `save-utils.ts` 工具集来处理不同格式的图表数据，确保数据的正确提取、转换和存储。

### 数据流程

整个数据流程包括以下步骤：

1. 从聊天消息中提取原始图表数据
2. 转换和标准化数据格式
3. 创建有效的配置对象
4. 保存到 Insights 存储中
5. 在 Insights 页面进行展示

### 关键组件

#### 1. `save-utils.ts`

这是核心工具文件，包含以下关键函数：

* `extractChartData`: 从消息内容中提取图表数据
* `mapChartType`: 映射图表类型
* `ensureDataFormat`: 确保数据格式正确
* `createValidConfig`: 创建有效的配置对象
* `saveChartToInsights`: 保存图表到 Insights

### 数据格式详解

#### 输入数据格式

系统支持多种输入数据格式：

**1. 数据可视化对象 (`data_visualization`)**

```json
{
  "data_visualization": {
    "chart_type": "bar_chart",
    "title": "示例图表",
    "description": "这是一个示例",
    "data": [
      {"name": "类别A", "value": 30},
      {"name": "类别B", "value": 45},
      {"name": "类别C", "value": 60}
    ]
  }
}
```

**2. 可视化对象 (`visualization`)**

```json
{
  "visualization": {
    "type": "line",
    "title": "趋势图",
    "description": "显示月度趋势",
    "data": [
      {"category": "一月", "value": 35},
      {"category": "二月", "value": 40},
      {"category": "三月", "value": 30}
    ]
  }
}
```

**3. 直接图表对象 (`chart`)**

```json
{
  "chart": {
    "type": "bar",
    "title": "销售数据",
    "data": [
      {"name": "产品A", "value": 100},
      {"name": "产品B", "value": 150}
    ],
    "config": {
      "value": {
        "label": "销售额",
        "color": "hsl(var(--chart-1))"
      }
    }
  }
}
```

**4. SQL查询结果**

```json
{
  "sql_agent": {
    "query_result": [
      {"product": "产品A", "sales": 100},
      {"product": "产品B", "sales": 150}
    ]
  }
}
```

**5. SSE 事件数据**

```
event: agent_visualization
data: {"content":{"data_visualization":{"chart_type":"bar","title":"销售分析","data":[...]}}}
```

#### 标准化后的数据格式

所有输入格式都会被转换为以下统一格式：

```typescript
interface StandardChartData {
  type: "bar" | "line" | "area";  // 支持的图表类型
  data: any[];  // 标准化的数据数组
  rawData: any[];  // 原始数据备份
  config: Record<string, { label: string; color: string }>;  // 配置对象
  title: string;  // 图表标题
  description?: string;  // 图表描述
}
```

#### 保存到 Insights 的数据格式

图表数据保存到 Insights 存储中的完整格式：

```typescript
interface SavedChart {
  id: string;  // 唯一ID
  messageId: string;  // 关联的消息ID
  sessionId: string;  // 关联的会话ID
  title: string;  // 图表标题
  description?: string;  // 图表描述
  type: "bar" | "line" | "area";  // 图表类型
  data: any[];  // 处理后的数据
  rawData: any[];  // 原始数据
  config: Record<string, { label: string; color: string }>;  // 配置对象
  createdAt: number;  // 创建时间
  layout: {  // 布局信息
    x: number;  // 水平位置
    y: number;  // 垂直位置
    w: number;  // 宽度单位
    h: number;  // 高度单位
  }
}
```

### 详细实现

#### 1. 图表类型映射 (`mapChartType`)

```typescript
function mapChartType(typeStr: string | undefined): "bar" | "line" | "area" {
  if (!typeStr) return "bar";
  
  const type = typeStr.toLowerCase();
  if (type.includes("bar")) return "bar";
  if (type.includes("line")) return "line";
  if (type.includes("area")) return "area";
  
  // 默认为柱状图
  return "bar";
}
```

这个函数将各种可能的图表类型字符串（如 "bar\_chart", "line\_graph" 等）映射到我们支持的三种标准类型："bar", "line", "area"。

#### 2. 数据格式标准化 (`ensureDataFormat`)

```typescript
function ensureDataFormat(data: any): any[] {
  // 如果数据不是数组，尝试转换
  if (!Array.isArray(data)) {
    // 处理特殊格式...
    return [/* 标准化的数据 */];
  }
  
  // 标准化数组中的每个项
  return data.map((item, index) => {
    const result: Record<string, any> = { };
    
    // 处理name字段
    if (item.name) {
      result.name = item.name;
    } else if (item.category) {
      result.name = item.category;
    } else if (item.label) {
      result.name = item.label;
    } else {
      result.name = `Item ${index + 1}`;
    }
    
    // 处理数据值...
    
    return result;
  });
}
```

这个函数确保数据是标准格式的数组，每个项都有 `name` 属性和至少一个数值属性。它能处理多种输入格式，包括非数组和数组中缺少必要属性的情况。

#### 3. 配置对象创建 (`createValidConfig`)

```typescript
function createValidConfig(data: any[], existingConfig: any): Record<string, { label: string; color: string }> {
  // 使用现有配置或创建新配置
  const config: Record<string, { label: string; color: string }> = {};
  
  // 分析数据中的值字段并创建配置
  Object.keys(firstItem).forEach((key, index) => {
    // 跳过name字段
    if (key === 'name' || key === 'category' || key === 'label') return;
    
    // 为数值字段创建配置
    if (typeof value === 'number' || (typeof value === 'string' && !isNaN(parseFloat(value)))) {
      config[key] = {
        label: key.charAt(0).toUpperCase() + key.slice(1).replace(/_/g, ' '),
        color: `hsl(var(--chart-${(valueFieldCount % 5) + 1}))`
      };
    }
  });
  
  return config;
}
```

这个函数创建图表所需的配置对象，指定每个数据系列的标签和颜色。如果已有配置则使用现有配置，否则根据数据自动生成。

#### 4. 图表数据提取 (`extractChartData`)

```typescript
export function extractChartData(
  content: string,
  messageId: string,
  sessionId: string,
  title: string = "Chart"
): {
  type: "bar" | "line" | "area";
  data: any[];
  rawData: any[];
  config: Record<string, { label: string; color: string }>;
  title: string;
  description?: string;
} | null {
  try {
    // 尝试解析内容为JSON
    if (content.trim().startsWith('{') || content.trim().startsWith('[')) {
      // 处理各种JSON格式...
    }
    
    // 检查SSE格式
    if (content.includes('event:') && content.includes('data:')) {
      // 处理SSE格式...
    }
    
    return null; // 无法提取有效数据
  } catch (error) {
    console.error("Error extracting chart data:", error);
    return null;
  }
}
```

这是核心函数，负责从消息内容中提取图表数据。它能处理多种格式，包括JSON对象和SSE事件流。

#### 5. 保存到 Insights (`saveChartToInsights`)

```typescript
export function saveChartToInsights(
  content: string,
  messageId: string,
  sessionId: string,
  title: string = "Chart"
): boolean {
  try {
    // 提取图表数据
    const chartData = extractChartData(content, messageId, sessionId, title);
    
    if (!chartData) {
      console.error("Could not extract valid chart data");
      return false;
    }
    
    // 使用store保存图表
    const { saveChart } = useInsightStore.getState();
    
    saveChart({
      title: chartData.title,
      description: chartData.description || "",
      messageId,
      sessionId,
      type: chartData.type,
      data: chartData.data,
      rawData: chartData.rawData,
      config: chartData.config,
    });
    
    return true;
  } catch (error) {
    console.error("Error saving chart to insights:", error);
    return false;
  }
}
```

这个函数将提取的图表数据保存到 Insights 存储中，供 Insights 面板展示使用。

### 集成到ChatItem组件

在 `ChatItem.tsx` 中，我们需要修改保存图表的逻辑，使用 `save-utils.ts` 中的函数：

```typescript
import { extractChartData } from "@/utils/chart/save-utils";

const handleSaveClick = () => {
  // 重置错误状态
  setChartError(null);
  // 设置默认标题
  const defaultTitle = `Chart from ${new Date().toLocaleDateString()}`;
  setChartTitle(defaultTitle);
  setSaveModalVisible(true);
}

const handleSaveConfirm = () => {
  if (!chartTitle.trim()) {
    setChartError("Please enter a title");
    return;
  }

  try {
    // 使用改进的图表提取函数
    const chartData = extractChartData(content, id, sessionId, chartTitle);
    
    if (!chartData) {
      throw new Error("Could not extract chart data from this message");
    }
    
    // 保存图表
    saveChart({
      title: chartTitle, // 使用用户输入的标题
      description: chartData.description || "Saved from chat",
      messageId: id,
      sessionId: sessionId,
      type: chartData.type,
      data: chartData.data,
      rawData: chartData.rawData,
      config: chartData.config,
    });

    // 显示成功提示
    message.success("Chart saved to Insights");
    setSaveModalVisible(false);
  } catch (error) {
    console.error("Error saving chart:", error);
    setChartError(error instanceof Error ? error.message : "Failed to save chart");
  }
}
```

### Insights页面展示

在 `InsightsDashboard.tsx` 中，我们从存储中获取已保存的图表并进行展示：

```typescript
const InsightsDashboard = () => {
  const { savedCharts, deleteChart, updateChartLayout } = useInsightStore()
  
  // 生成布局配置
  const layouts = {
    lg: savedCharts.map(chart => ({
      i: chart.id,
      x: chart.layout.x,
      y: chart.layout.y,
      w: chart.layout.w,
      h: chart.layout.h,
    })),
  }
  
  return (
    <div className="insights-dashboard">
      {/* ... */}
      <ResponsiveGridLayout layouts={layouts} /* ... */>
        {savedCharts.map(chart => (
          <div key={chart.id}>
            <Card /* ... */>
              <Chart
                data={chart.data}
                rawData={chart.rawData}
                config={chart.config}
                type={chart.type}
                title={chart.title}
                description={chart.description}
              />
            </Card>
          </div>
        ))}
      </ResponsiveGridLayout>
    </div>
  )
}
```

### 常见问题解决

#### 1. 数据不显示或显示为默认数据

如果 Insights 页面显示默认数据而非实际图表数据，可能有以下原因：

* 数据提取失败：检查控制台中的错误日志，确认 `extractChartData` 是否成功提取数据
* 数据格式错误：确保数据格式符合预期，包含必要字段
* 存储问题：检查 `useInsightStore` 中的 `saveChart` 函数是否正确保存数据

解决方案：

* 添加日志记录提取过程
* 验证提取的数据格式
* 检查存储逻辑

#### 2. 图表显示不正确

如果图表显示不正确（如尺寸问题、溢出等），可能需要：

* 调整 Chart 组件的尺寸和样式
* 确保数据格式正确，特别是数值字段
* 检查配置对象，确保每个数据系列都有正确的配置

### 总结

通过 `save-utils.ts` 工具集，我们实现了从聊天消息中提取图表数据并保存到 Insights 页面的功能。该工具能够处理多种输入格式，确保数据被正确提取、标准化并保存，从而实现数据可视化的无缝体验。

这套系统的优势在于其灵活性和健壮性，能够处理不同数据源和格式，并提供标准化的输出供图表组件使用。通过合理的错误处理和日志记录，也便于开发者进行调试和故障排除。
