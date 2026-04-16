---
name: websocket-realtime-communication
description: >
  本技能用于实现WebSocket实时通信，包括连接管理、心跳机制、重连策略、消息队列等。
  当需要实现实时通信、处理WebSocket连接、实现消息推送时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# WebSocket实时通信

## 一、概述

### 1.1 这是什么

WebSocket实时通信技能提供完整的WebSocket解决方案，涵盖连接管理、心跳机制、重连策略、消息队列等。

### 1.2 适用场景

- ✅ 实时消息推送
- ✅ 在线聊天
- ✅ 实时数据更新
- ✅ 协同编辑
- ✅ 实时通知
- ❌ 一次性数据获取（使用HTTP）
- ❌ 大文件传输

### 1.3 核心原则

1. **连接稳定** —— 自动重连机制
2. **心跳保活** —— 防止连接断开
3. **消息可靠** —— 消息确认和重发
4. **资源释放** —— 组件卸载时清理

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要实时双向通信
- 需要服务器推送消息
- 需要实现在线聊天
- 需要实时数据同步
- 需要处理WebSocket连接

### 2.2 输入

- WebSocket服务端地址
- 消息协议定义
- 重连策略需求
- 心跳间隔设置

### 2.3 输出

- WebSocket Hook/类
- 连接管理逻辑
- 消息处理机制
- 重连策略实现

---

## 三、WebSocket Hook

```typescript
import { useEffect, useRef, useState, useCallback } from 'react';

interface WebSocketOptions {
  url: string;
  protocols?: string | string[];
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
  heartbeatInterval?: number;
  onOpen?: (event: Event) => void;
  onClose?: (event: CloseEvent) => void;
  onError?: (event: Event) => void;
  onMessage?: (data: any) => void;
}

interface WebSocketState {
  readyState: number;
  isConnected: boolean;
  reconnectAttempts: number;
}

export function useWebSocket(options: WebSocketOptions) {
  const {
    url,
    protocols,
    reconnectInterval = 3000,
    maxReconnectAttempts = 5,
    heartbeatInterval = 30000,
    onOpen,
    onClose,
    onError,
    onMessage,
  } = options;

  const wsRef = useRef<WebSocket | null>(null);
  const reconnectTimerRef = useRef<NodeJS.Timeout>();
  const heartbeatTimerRef = useRef<NodeJS.Timeout>();
  const reconnectCountRef = useRef(0);

  const [state, setState] = useState<WebSocketState>({
    readyState: WebSocket.CONNECTING,
    isConnected: false,
    reconnectAttempts: 0,
  });

  // 连接WebSocket
  const connect = useCallback(() => {
    if (wsRef.current?.readyState === WebSocket.OPEN) return;

    try {
      const ws = new WebSocket(url, protocols);
      wsRef.current = ws;

      ws.onopen = (event) => {
        reconnectCountRef.current = 0;
        setState({
          readyState: WebSocket.OPEN,
          isConnected: true,
          reconnectAttempts: 0,
        });
        onOpen?.(event);
        startHeartbeat();
      };

      ws.onclose = (event) => {
        setState((prev) => ({
          ...prev,
          readyState: WebSocket.CLOSED,
          isConnected: false,
        }));
        onClose?.(event);
        stopHeartbeat();
        
        // 自动重连
        if (!event.wasClean && reconnectCountRef.current < maxReconnectAttempts) {
          reconnectTimerRef.current = setTimeout(() => {
            reconnectCountRef.current++;
            setState((prev) => ({
              ...prev,
              reconnectAttempts: reconnectCountRef.current,
            }));
            connect();
          }, reconnectInterval);
        }
      };

      ws.onerror = (event) => {
        onError?.(event);
      };

      ws.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data);
          onMessage?.(data);
        } catch {
          onMessage?.(event.data);
        }
      };
    } catch (error) {
      console.error('WebSocket连接失败:', error);
    }
  }, [url, protocols, onOpen, onClose, onError, onMessage]);

  // 断开连接
  const disconnect = useCallback(() => {
    clearTimeout(reconnectTimerRef.current);
    stopHeartbeat();
    
    if (wsRef.current) {
      wsRef.current.close();
      wsRef.current = null;
    }
  }, []);

  // 发送消息
  const send = useCallback((data: any) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      const message = typeof data === 'string' ? data : JSON.stringify(data);
      wsRef.current.send(message);
      return true;
    }
    return false;
  }, []);

  // 心跳
  const startHeartbeat = useCallback(() => {
    heartbeatTimerRef.current = setInterval(() => {
      send({ type: 'ping' });
    }, heartbeatInterval);
  }, [send, heartbeatInterval]);

  const stopHeartbeat = useCallback(() => {
    clearInterval(heartbeatTimerRef.current);
  }, []);

  // 组件挂载时连接
  useEffect(() => {
    connect();
    return disconnect;
  }, [connect, disconnect]);

  return {
    ...state,
    send,
    connect,
    disconnect,
  };
}
```

---

## 四、消息队列

```typescript
interface QueuedMessage {
  id: string;
  data: any;
  timestamp: number;
  retries: number;
}

export function useMessageQueue(maxRetries = 3) {
  const queueRef = useRef<QueuedMessage[]>([]);
  const [pendingCount, setPendingCount] = useState(0);

  const enqueue = useCallback((data: any) => {
    const message: QueuedMessage = {
      id: `${Date.now()}-${Math.random()}`,
      data,
      timestamp: Date.now(),
      retries: 0,
    };
    
    queueRef.current.push(message);
    setPendingCount(queueRef.current.length);
    
    return message.id;
  }, []);

  const dequeue = useCallback(() => {
    const message = queueRef.current.shift();
    setPendingCount(queueRef.current.length);
    return message;
  }, []);

  const retry = useCallback((id: string) => {
    const message = queueRef.current.find((m) => m.id === id);
    if (message && message.retries < maxRetries) {
      message.retries++;
      return true;
    }
    return false;
  }, [maxRetries]);

  const clear = useCallback(() => {
    queueRef.current = [];
    setPendingCount(0);
  }, []);

  return {
    enqueue,
    dequeue,
    retry,
    clear,
    pendingCount,
    queue: queueRef.current,
  };
}
```

---

## 五、使用示例

```typescript
// 聊天组件
function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');

  const { isConnected, send } = useWebSocket({
    url: `wss://api.example.com/chat/${roomId}`,
    onMessage: (data) => {
      if (data.type === 'message') {
        setMessages((prev) => [...prev, data.payload]);
      }
    },
  });

  const handleSend = () => {
    if (!input.trim()) return;
    
    send({
      type: 'message',
      payload: {
        content: input,
        timestamp: Date.now(),
      },
    });
    
    setInput('');
  };

  return (
    <div className="chat-room">
      <div className="connection-status">
        {isConnected ? '🟢 已连接' : '🔴 未连接'}
      </div>
      
      <div className="messages">
        {messages.map((msg) => (
          <div key={msg.id} className="message">
            <span className="content">{msg.content}</span>
            <span className="time">
              {new Date(msg.timestamp).toLocaleTimeString()}
            </span>
          </div>
        ))}
      </div>
      
      <div className="input-area">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && handleSend()}
          placeholder="输入消息..."
        />
        <button onClick={handleSend} disabled={!isConnected}>
          发送
        </button>
      </div>
    </div>
  );
}
```

---

## 六、决策检查清单

- [ ] WebSocket连接自动重连
- [ ] 心跳机制防止断开
- [ ] 消息发送失败有重试
- [ ] 组件卸载时清理资源
- [ ] 连接状态显示清晰
- [ ] 消息队列管理完善
- [ ] 错误处理完善
- [ ] 支持消息确认机制

---

## 七、相关技能

- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 数据获取
- [状态管理实现](./state-management-implementation/SKILL.md) —— 消息状态管理
- [性能优化实战](./performance-optimization/SKILL.md) —— 实时数据优化

---

## 八、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含WebSocket Hook、消息队列、重连机制 |
