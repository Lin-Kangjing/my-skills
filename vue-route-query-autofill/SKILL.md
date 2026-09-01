---
name: vue-route-query-autofill
description: |
  当用户需要在 Vue 页面中处理从其他页面跳转过来的 URL 查询参数时，使用此 skill。
  自动将 $route.query 中的参数填充到页面的 queryParams 中，实现页面间数据传递和自动搜索。
  
  触发场景：
  - 用户提到"从其他页面跳转"、"URL 参数"、"query 参数"
  - 用户提到"自动填充搜索条件"、"带参数打开页面"
  - 用户需要在 onMounted 中处理路由参数
  - 用户提到类似 `for (const key in query)` 的参数遍历模式
  
  适用框架：Vue 3 + Vue Router 4
---

# Vue 路由查询参数自动填充

## 功能说明

此 skill 用于处理 Vue 页面从 URL 查询参数自动填充到页面搜索表单的场景。常见于：
- 列表页点击某行跳转到详情页，并传递搜索条件
- 页面间跳转时保留搜索状态
- 从通知/消息点击跳转到具体列表并自动筛选

## 实现模式

### 范式一：使用 proxy?.$route.query（推荐，如果已有 proxy）

```typescript
onMounted(() => {
  // 从 URL 查询参数中获取数据
  const query = proxy?.$route.query;
  
  // 遍历所有查询参数，自动填充到 queryParams
  for (const key in query) {
    if (key in queryParams.value) {
      queryParams.value[key] = query[key];
    }
  }
  
  // 执行搜索
  getList();
});
```

### 范式二：使用 useRoute()（推荐，Composition API 标准写法）

```typescript
import { useRoute } from 'vue-router';

// 在 script setup 顶层或函数内部使用
const route = useRoute();

onMounted(() => {
  // 从 URL 查询参数中获取数据
  const query = route.query;
  
  // 遍历所有查询参数，自动填充到 queryParams
  for (const key in query) {
    if (key in queryParams.value) {
      queryParams.value[key] = query[key];
    }
  }
  
  // 执行搜索
  getList();
});
```

### 带类型转换的实现

如果参数需要类型转换（如数字、日期等）：

```typescript
import { useRoute } from 'vue-router';

const route = useRoute();

onMounted(() => {
  const query = route.query;
  
  for (const key in query) {
    if (key in queryParams.value) {
      // 根据字段类型进行转换
      if (key === 'pageNum' || key === 'pageSize') {
        queryParams.value[key] = Number(query[key]);
      } else if (key === 'startDate' || key === 'endDate') {
        queryParams.value[key] = query[key] as string;
      } else {
        queryParams.value[key] = query[key];
      }
    }
  }
  
  getList();
});
```

### 特定字段映射

如果 URL 参数名和 queryParams 字段名不一致：

```typescript
import { useRoute } from 'vue-router';

const route = useRoute();

onMounted(() => {
  const query = route.query;
  
  // 字段映射表：URL参数名 -> queryParams字段名
  const fieldMapping: Record<string, string> = {
  };
  
  for (const key in query) {
    const targetKey = fieldMapping[key] || key;
    if (targetKey in queryParams.value) {
      queryParams.value[targetKey] = query[key];
    }
  }
  
  getList();
});
```

## 是否需要放在 onMounted 里面？

### ✅ 必须放在 onMounted 中的场景

1. **页面初始化时需要从 URL 读取参数并搜索**
   - 用户直接访问带参数的 URL
   - 从其他页面跳转过来需要自动填充并执行搜索

2. **需要等待组件挂载后才能安全访问路由**
   - 虽然 useRoute() 在 setup 中就可以调用
   - 但某些场景（如需要操作 DOM 或子组件）需要等待挂载完成

### ❌ 不需要放在 onMounted 中的场景

1. **仅在用户交互时读取参数**
   ```typescript
   const handleSomeAction = () => {
     const query = route.query;
     // 处理参数
   };
   ```

2. **使用 watch 监听路由变化**
   ```typescript
   import { watch } from 'vue';
   import { useRoute } from 'vue-router';
   
   const route = useRoute();
   
   watch(() => route.query, (newQuery) => {
     for (const key in newQuery) {
       if (key in queryParams.value) {
         queryParams.value[key] = newQuery[key];
       }
     }
     getList();
   }, { immediate: true });
   ```

### 💡 最佳实践建议

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 页面首次加载自动填充 | `onMounted` | 确保组件初始化完成后再执行搜索 |
| 路由参数变化时自动更新 | `watch(() => route.query)` | 可以响应浏览器前进/后退 |
| 用户点击按钮时读取 | 直接在方法中读取 | 不需要在 onMounted 中处理 |

## 跳转页面示例

在源页面中构建带参数的 URL：

```typescript
import { useRouter } from 'vue-router';

const router = useRouter();

/**
 * 跳转到目标页面，传递查询参数
 */
const handleNavToTarget = (row: SomeVO) => {
  const routeUrl = router.resolve({
    path: '/target/path',
    query: {
      name: row.name,
      status: row.status,
      // 其他需要传递的参数
    }
  });
  window.open(routeUrl.href, '_blank');
};
```

## 完整示例

### 通知记录页面（跳转源）

```vue
<template>
  <el-button link type="primary" @click="handleNavNoticePolicy(scope.row)">
    查看策略
  </el-button>
</template>

<script setup>
import { useRouter } from 'vue-router';

const router = useRouter();

const handleNavNoticePolicy = (row) => {
  const routeUrl = router.resolve({
    path: '/alarm/alarmSettings/noticeSettings/noticePolicy',
    query: {
      name: row.noticePolicyName  // 传递策略名称
    }
  });
  window.open(routeUrl.href, '_blank');
};
</script>
```

### 通知策略页面（参数接收方）

```vue
<script setup>
import { onMounted } from 'vue';
import { useRoute } from 'vue-router';

const route = useRoute();

onMounted(() => {
  // 检查是否有从其他页面传递过来的查询参数
  const query = route.query;
  for (const key in query) {
    if (key in queryParams.value) {
      queryParams.value[key] = query[key];
    }
  }
  getList();
});
</script>
```

## 注意事项

1. **参数安全**：使用 `in` 操作符检查字段是否存在，避免注入无关参数
2. **类型处理**：根据实际字段类型进行必要的类型转换
3. **默认值**：如需设置默认值，在遍历前处理
4. **新标签页**：使用 `window.open` 打开新标签页时，需使用 `router.resolve` 生成完整 URL
5. **Composition API**：推荐使用 `useRoute()` 和 `useRouter()`，更符合 Vue 3 风格

## 相关文件

- 通常修改目标页面的 `onMounted` 生命周期钩子
- 可能需要在类型定义中添加可选字段（如 `noticePolicyName`）
