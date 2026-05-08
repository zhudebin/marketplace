# 思维指南

> **目的**：在写代码前先把"想清楚"的步骤系统化，避免常见的"没想到那一步"导致的 bug。
>
> **核心理念**：30 分钟的思考，能省掉 3 小时的调试。

---

## 为什么需要思维指南？

**大多数 bug 和技术债来自"没想到"，而不是"不会写"**：

- 没想到层间数据格式 -> 跨层 bug
- 没想到代码模式重复 -> 处处复制粘贴
- 没想到边界情况 -> 运行时错误
- 没想到后人维护 -> 写出无法理解的代码

这些指南帮助你在动手前**先问对问题**。

---

## 现有指南

| 指南                                                                  | 用途                              | 何时使用                                     |
| --------------------------------------------------------------------- | --------------------------------- | -------------------------------------------- |
| [Cross-Layer Thinking](./cross-layer-thinking-guide.md)               | 跨层数据流梳理                    | 跨 3+ 层（Vue → axios → Controller → Service → Mapper）特性实现前 |
| [Pre-Implementation Checklist](./pre-implementation-checklist.md)     | 编码前的就绪度检查                | 任何新特性开始前                             |

---

## 何时用哪个？

### 跨层问题

[Cross-Layer Thinking](./cross-layer-thinking-guide.md)，当：

- [ ] 特性触及 3+ 层
- [ ] 数据格式在层间会变化（DO ↔ DTO ↔ VO ↔ TS 类型）
- [ ] 多个调用方需要同样的数据
- [ ] 不确定某段逻辑该写在哪一层
- [ ] 需要对接外部服务（第三方 API、消息队列）

### 编码前

[Pre-Implementation Checklist](./pre-implementation-checklist.md)，当：

- [ ] 准备添加一个常量或配置
- [ ] 准备实现新逻辑
- [ ] 准备定义 DTO / 类型
- [ ] 准备创建组件 / Composable
- [ ] 准备新增 Controller / Service / Mapper 方法
- [ ] 感觉"这段我以前好像写过"

---

## 修改前定律（最关键）

> **修改任何值之前，永远先搜！**

```bash
# 后端
rg "VALUE_TO_CHANGE" --type java
rg "value_to_change" backend/src

# 前端
rg "valueToChange" --type ts --type vue
```

这一个习惯能消除大半"忘了同步 X"的 bug。

---

## Vue + Spring Boot 项目的典型层次

```
Vue Component (template + <script setup>)
        |
        v
Composable (业务封装、副作用)
        |
        v
Pinia Store (跨页面共享状态)
        |
        v
axios 实例 (拦截器、错误统一)
        |
        v
Spring Controller (DTO 校验、HTTP 语义)
        |
        v
Service (业务逻辑、事务边界)
        |
        v
Mapper (MyBatis-Plus / 自定义 SQL)
        |
        v
MySQL
```

每一个边界都是潜在 bug 来源：

- **序列化**：Java `LocalDateTime` ↔ JSON ↔ TS `string`，时区与格式必须一致
- **类型不匹配**：后端 DTO 改字段，前端 TS 类型未重新生成
- **鉴权上下文**：Controller 拿到 `Authentication`，Service 内部拿不到（要么传参，要么 ThreadLocal）
- **事务边界**：Service 自调用导致 `@Transactional` 失效
- **缓存一致性**：Redis 缓存 / 浏览器缓存 / Pinia store 与数据库三方不同步

---

## 核心原则

1. **先搜后写** - 修改任何已有名词前，全局搜索
2. **先想后码** - 5 分钟清单胜过 50 分钟调试
3. **写出假设** - 隐式假设要显式化
4. **逐层验证** - 一处改动，多处可能要联动
5. **从 bug 学习** - 解决非平凡 bug 后回头补充指南

---

## 贡献

发现新的"我怎么没想到"瞬间？

1. 通用思维模式 -> 加入已有指南或新建指南
2. 引发了 bug -> 加入对应指南的"经验教训"段落
3. 项目专属 -> 创建独立的项目指南文件

---

**语言**：所有文档使用**简体中文**编写。
