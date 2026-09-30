## 3. 生命周期：Fiber 状态机 ⭐

每个被加载的插件都拥有一个 **Fiber** 作用域：

```
PENDING → LOADING → ACTIVE
                 ↘ FAILED
ACTIVE → UNLOADING → DISPOSED
```

| 状态 | 含义 |
|---|---|
| **PENDING** | 已声明，但**所需依赖未就绪** ← 这就是面板显示"等待服务"的状态 |
| LOADING | 依赖就绪，正在执行 `apply` |
| ACTIVE | 插件运行中 |
| **FAILED** | **`apply` 抛出异常** |
| UNLOADING | 正在卸载并释放资源 |
| DISPOSED | 已完全卸载 |

**依赖驱动的加载**：声明了 `inject` 的插件等待所有必需服务就绪。**如果依赖的服务消失**（例如提供方被替换），插件会**自动卸载**（ACTIVE → DISPOSED），**待服务恢复后重新加载**。

### 自动清理

通过 `ctx` 做的任何注册，卸载时自动撤销 —— 以下都会被自动追踪：

- `ctx.on(event, handler)` — 事件监听
- `ctx.tools.register(tool)` — 工具注册
- `ctx.llm.registerAdapter(names, adapter)` — LLM 适配器注册
- `ctx.effect(() => cleanup)` — 自定义资源

```ts
export function apply(ctx: Context) {
  ctx.on('some-event', handler)

  ctx.effect(() => {
    const connection = createConnection()
    return () => connection.close()     // 卸载时执行
  })
}
```

⚠️ **处置顺序的坑**：卸载时处置器**按注册顺序的逆序开始调用**，但**多个异步处置器会并发执行，不保证逐个完成**。

> 存在顺序依赖的清理步骤，**必须放进同一个 `ctx.effect()` 返回的处置器中**，由该处置器负责串行等待。

### 嵌套上下文与 dispose

```ts
ctx.plugin(childPlugin)      // 子 Fiber：继承父上下文，独立生命周期，随父卸载

const fiber = ctx.plugin(myPlugin)
await fiber.dispose()        // 手动提前终止
```

`dispose` 保证：① 该插件所有注册被移除 ② 子树递归卸载 ③ Promise 在所有异步清理完成后兑现。

### HMR

加载 `@deepseek-ai/dsh-hmr` 后，改插件源码触发：卸载旧插件（清理所有注册）→ 重新加载新代码 → 执行新 `apply`。因注册自动清理，**热替换不会保留旧实例的注册**。

---
