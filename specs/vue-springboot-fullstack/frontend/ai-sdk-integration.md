# AI SDK 集成（前端）

前端消费后端 SSE 流式接口、累积消息、渲染工具调用结果。后端实现见 [backend/ai-sdk-integration.md](../backend/ai-sdk-integration.md)。

## 流式协议约定

后端使用 SSE（Server-Sent Events）输出，每条事件格式：

```
event: delta
data: {"type":"text","content":"你好"}

event: tool_call
data: {"id":"call_1","name":"search","args":{"q":"vue"}}

event: tool_result
data: {"id":"call_1","result":"..."}

event: done
data: {"finishReason":"stop","usage":{"promptTokens":12,"completionTokens":34}}

event: error
data: {"code":"RATE_LIMITED","message":"..."}
```

> 所有事件 data 字段都是 JSON 字符串。前端解析后累积成完整消息。

## 推荐方式：fetch + ReadableStream

`EventSource` 不支持自定义 Header（无法带 JWT），生产环境推荐用 `fetch` + 手动解析 SSE。

```typescript
// src/composables/useChatStream.ts
import { ref } from 'vue';
import { useUserStore } from '@/stores/user';

export interface StreamMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  toolCalls: Array<{ id: string; name: string; args: unknown; result?: unknown }>;
  status: 'pending' | 'streaming' | 'done' | 'error';
  error?: string;
}

export function useChatStream() {
  const messages = ref<StreamMessage[]>([]);
  const streaming = ref(false);
  let abortController: AbortController | null = null;

  async function send(prompt: string) {
    if (streaming.value) return;

    const userStore = useUserStore();
    const userMsg: StreamMessage = {
      id: crypto.randomUUID(),
      role: 'user',
      content: prompt,
      toolCalls: [],
      status: 'done',
    };
    const assistantMsg: StreamMessage = {
      id: crypto.randomUUID(),
      role: 'assistant',
      content: '',
      toolCalls: [],
      status: 'streaming',
    };
    messages.value.push(userMsg, assistantMsg);

    streaming.value = true;
    abortController = new AbortController();

    try {
      const res = await fetch(`${import.meta.env.VITE_API_BASE_URL}/api/ai/chat`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${userStore.token}`,
        },
        body: JSON.stringify({ prompt, history: messages.value.slice(0, -1) }),
        signal: abortController.signal,
      });

      if (!res.ok || !res.body) {
        throw new Error(`HTTP ${res.status}`);
      }

      const reader = res.body.getReader();
      const decoder = new TextDecoder();
      let buffer = '';

      while (true) {
        const { value, done } = await reader.read();
        if (done) break;
        buffer += decoder.decode(value, { stream: true });

        // SSE 以双换行分隔事件
        const events = buffer.split('\n\n');
        buffer = events.pop() ?? '';

        for (const evt of events) {
          handleSseEvent(evt, assistantMsg);
        }
      }

      assistantMsg.status = 'done';
    } catch (e) {
      if ((e as Error).name === 'AbortError') {
        assistantMsg.status = 'done';
      } else {
        assistantMsg.status = 'error';
        assistantMsg.error = (e as Error).message;
      }
    } finally {
      streaming.value = false;
      abortController = null;
    }
  }

  function abort() {
    abortController?.abort();
  }

  function handleSseEvent(raw: string, target: StreamMessage) {
    const lines = raw.split('\n');
    let event = 'message';
    let data = '';
    for (const line of lines) {
      if (line.startsWith('event:')) event = line.slice(6).trim();
      else if (line.startsWith('data:')) data += line.slice(5).trim();
    }
    if (!data) return;

    try {
      const payload = JSON.parse(data);
      switch (event) {
        case 'delta':
          if (payload.type === 'text') target.content += payload.content;
          break;
        case 'tool_call':
          target.toolCalls.push({ id: payload.id, name: payload.name, args: payload.args });
          break;
        case 'tool_result': {
          const tc = target.toolCalls.find((c) => c.id === payload.id);
          if (tc) tc.result = payload.result;
          break;
        }
        case 'error':
          target.status = 'error';
          target.error = payload.message;
          break;
      }
    } catch {
      // 静默忽略损坏帧
    }
  }

  return { messages, streaming, send, abort };
}
```

## 组件使用

```vue
<script setup lang="ts">
import { ref, nextTick, watch } from 'vue';
import { useChatStream } from '@/composables/useChatStream';

const { messages, streaming, send, abort } = useChatStream();
const input = ref('');
const scrollEl = ref<HTMLElement>();

async function onSend() {
  if (!input.value.trim() || streaming.value) return;
  const prompt = input.value;
  input.value = '';
  await send(prompt);
}

watch(
  () => messages.value.at(-1)?.content,
  () => nextTick(() => scrollEl.value?.scrollTo({ top: scrollEl.value.scrollHeight })),
);
</script>

<template>
  <div class="chat">
    <div ref="scrollEl" class="messages">
      <div
        v-for="msg in messages"
        :key="msg.id"
        :class="['msg', msg.role]"
      >
        <div class="content" v-text="msg.content || (msg.status === 'streaming' ? '思考中…' : '')" />
        <div v-if="msg.toolCalls.length" class="tools">
          <el-tag v-for="tc in msg.toolCalls" :key="tc.id" size="small">
            🔧 {{ tc.name }}{{ tc.result ? ' ✓' : '…' }}
          </el-tag>
        </div>
        <el-alert v-if="msg.status === 'error'" :title="msg.error" type="error" :closable="false" />
      </div>
    </div>
    <div class="input">
      <el-input
        v-model="input"
        type="textarea"
        :rows="2"
        :disabled="streaming"
        @keydown.enter.exact.prevent="onSend"
      />
      <el-button v-if="!streaming" type="primary" @click="onSend">发送</el-button>
      <el-button v-else type="danger" @click="abort">中断</el-button>
    </div>
  </div>
</template>
```

## 关键约定

### 1. 必须支持中断

用户切页面、关闭弹窗时调 `abort()`，否则连接挂着浪费 token + 后端线程。

### 2. 缓冲区拼接

SSE 帧可能被 TCP 切割，必须维护 `buffer` 直到遇到 `\n\n` 才解析。

### 3. 错误隔离

单条事件 `JSON.parse` 失败不要中断整个流，记录后跳过。

### 4. 工具调用渲染

工具调用应当显式可见（用户感知 AI 在做什么），而不是只显示最终文本。

### 5. 历史消息发回后端

每次 `send` 都把上下文 `history` 传回后端（除非后端用 session 保存），便于多轮对话。

## 长任务进度

如果工具调用本身是长流程（如代码生成），后端应在执行过程中持续发 `event: tool_progress`：

```typescript
case 'tool_progress': {
  const tc = target.toolCalls.find((c) => c.id === payload.id);
  if (tc) (tc as any).progress = payload.progress;
  break;
}
```

## Markdown 渲染

AI 输出常带 Markdown，使用 `markdown-it` 或 `@vueuse/integrations` + DOMPurify：

```typescript
import MarkdownIt from 'markdown-it';
import DOMPurify from 'dompurify';

const md = new MarkdownIt({ html: false, breaks: true, linkify: true });

const renderMarkdown = (text: string) => DOMPurify.sanitize(md.render(text));
```

```vue
<div class="content" v-html="renderMarkdown(msg.content)" />
```

> **必须** 用 DOMPurify 过滤 XSS。AI 输出不可信。

## 最佳实践

1. **fetch + ReadableStream** 而非 EventSource（要带 token）
2. **必须支持 abort**
3. **缓冲到完整事件再解析**
4. **错误隔离**，单帧错误不中断流
5. **Markdown 输出必须消毒**

## 反模式

- 用 `EventSource` 但 token 放 query string（日志泄漏）
- 不维护 buffer，直接 `split('\n')` 解析（半帧丢失）
- 不实现中断（用户离开页面后请求继续跑）
- 直接 `v-html="msg.content"` 不过滤（XSS）
- 后端返回完整 Markdown JSON 数组而不是流式（体验差）
