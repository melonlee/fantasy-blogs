# Vibe Coding 系列 11：全栈 Spring Boot + React 实战

前端后端一起写，这大概是全栈开发者最常见的愿望。Cursor 在这方面其实有天然优势——它能同时理解 Java 和 TypeScript/React，在两个世界之间跳转时不会丢上下文。这篇来实打实地做一个完整功能：从 React 组件写起，到 Spring Boot 接口，再到手把手串起来。

## 项目结构设计

在说代码之前，先把结构理清楚。我见过太多全栈项目前端一个目录、后端一个目录、互相不知道对方长什么样。Cursor 的做法是让整个项目在一个 workspace 里，前端后端平级存在，通过 REST API 通信。

```
my-fullstack-app/
├── frontend/               # Vite + React
│   ├── src/
│   │   ├── components/    # React 组件
│   │   ├── hooks/         # 自定义 Hooks
│   │   ├── api/           # API 调用封装
│   │   └── App.tsx
│   ├── package.json
│   └── vite.config.ts
├── backend/               # Spring Boot
│   ├── src/main/java/com/example/
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   └── model/
│   ├── src/main/resources/
│   │   └── application.yml
│   └── pom.xml
└── .cursor/               # 共享的规则
```

这个结构的好处是：Cursor 在索引的时候会把前后端代码一起扫，遇到问题能跨越边界给出建议。比如你在 React 里调用了一个后端接口，Cursor 能直接跳到对应的 Controller 方法，告诉你这个接口的参数校验逻辑是什么样的。

## React 组件：任务列表

来做个实际功能——任务列表。这是全栈项目的经典起点，但我要把它做完整，不只是显示数据，还要有增删改查、loading 状态、错误处理。

```tsx
// frontend/src/components/TaskList.tsx
import React, { useState, useEffect, useCallback } from 'react';
import { Task } from '../types/Task';
import { taskApi } from '../api/tasks';
import { useTaskStore } from '../store/taskStore';
import './TaskList.css';

interface TaskListProps {
  projectId: string;
}

export const TaskList: React.FC<TaskListProps> = ({ projectId }) => {
  const [tasks, setTasks] = useState<Task[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');
  const { selectedTaskId, setSelectedTaskId } = useTaskStore();

  const fetchTasks = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await taskApi.getTasks(projectId);
      setTasks(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : '加载任务失败');
    } finally {
      setLoading(false);
    }
  }, [projectId]);

  useEffect(() => {
    fetchTasks();
  }, [fetchTasks]);

  const handleDelete = async (taskId: string) => {
    try {
      await taskApi.deleteTask(taskId);
      setTasks(prev => prev.filter(t => t.id !== taskId));
      if (selectedTaskId === taskId) {
        setSelectedTaskId(null);
      }
    } catch (err) {
      setError('删除失败，请重试');
    }
  };

  const handleToggleComplete = async (task: Task) => {
    try {
      const updated = await taskApi.updateTask(task.id, {
        completed: !task.completed
      });
      setTasks(prev => prev.map(t => t.id === task.id ? updated : t));
    } catch (err) {
      setError('更新失败，请重试');
    }
  };

  const filteredTasks = tasks.filter(task => {
    if (filter === 'active') return !task.completed;
    if (filter === 'completed') return task.completed;
    return true;
  });

  if (loading) {
    return (
      <div className="task-list-loading">
        <div className="spinner" />
        <span>加载中...</span>
      </div>
    );
  }

  if (error) {
    return (
      <div className="task-list-error">
        <span className="error-icon">⚠️</span>
        <span>{error}</span>
        <button onClick={fetchTasks}>重试</button>
      </div>
    );
  }

  return (
    <div className="task-list">
      <div className="task-list-header">
        <h2>任务列表</h2>
        <div className="task-filters">
          {(['all', 'active', 'completed'] as const).map(f => (
            <button
              key={f}
              className={filter === f ? 'active' : ''}
              onClick={() => setFilter(f)}
            >
              {f === 'all' ? '全部' : f === 'active' ? '进行中' : '已完成'}
            </button>
          ))}
        </div>
      </div>

      <ul className="task-items">
        {filteredTasks.length === 0 ? (
          <li className="task-empty">暂无任务</li>
        ) : (
          filteredTasks.map(task => (
            <li
              key={task.id}
              className={`task-item ${task.id === selectedTaskId ? 'selected' : ''}`}
              onClick={() => setSelectedTaskId(task.id)}
            >
              <input
                type="checkbox"
                checked={task.completed}
                onChange={() => handleToggleComplete(task)}
                onClick={e => e.stopPropagation()}
              />
              <span className={task.completed ? 'completed' : ''}>
                {task.title}
              </span>
              <button
                className="delete-btn"
                onClick={(e) => {
                  e.stopPropagation();
                  handleDelete(task.id);
                }}
              >
                ×
              </button>
            </li>
          ))
        )}
      </ul>
    </div>
  );
};
```

这段组件代码涵盖了：状态管理（useState）、副作用处理（useEffect）、回调缓存（useCallback）、条件渲染、事件处理。看起来挺多，但每个部分都有明确的职责，不是为了"展示技术"堆出来的。

## API 调用封装

组件里用到了 `taskApi`，这个封装层的作用是把 fetch 逻辑和 React 组件解耦。好处是：接口变了只需要改一个地方，组件不需要关心 HTTP 细节。

```typescript
// frontend/src/api/tasks.ts
import { Task, CreateTaskRequest, UpdateTaskRequest } from '../types/Task';

const API_BASE = import.meta.env.VITE_API_BASE || 'http://localhost:8080/api';

class TaskApiError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public response?: unknown
  ) {
    super(message);
    this.name = 'TaskApiError';
  }
}

async function request<T>(
  path: string,
  options?: RequestInit
): Promise<T> {
  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
    credentials: 'include',  // 携带 cookie，支持 session
  });

  if (!response.ok) {
    const errorBody = await response.text().catch(() => '');
    throw new TaskApiError(
      `请求失败: ${response.status} ${response.statusText}`,
      response.status,
      errorBody
    );
  }

  if (response.status === 204) {
    return undefined as T;
  }

  return response.json();
}

export const taskApi = {
  getTasks: (projectId: string): Promise<Task[]> =>
    request<Task[]>(`/projects/${projectId}/tasks`),

  getTask: (taskId: string): Promise<Task> =>
    request<Task>(`/tasks/${taskId}`),

  createTask: (projectId: string, data: CreateTaskRequest): Promise<Task> =>
    request<Task>(`/projects/${projectId}/tasks`, {
      method: 'POST',
      body: JSON.stringify(data),
    }),

  updateTask: (taskId: string, data: UpdateTaskRequest): Promise<Task> =>
    request<Task>(`/tasks/${taskId}`, {
      method: 'PATCH',
      body: JSON.stringify(data),
    }),

  deleteTask: (taskId: string): Promise<void> =>
    request<void>(`/tasks/${taskId}`, {
      method: 'DELETE',
    }),
};
```

这里用了 `credentials: 'include'` 来支持带 cookie 的请求，这样如果后端用 Session 认证就不需要额外处理 token。环境变量 `VITE_API_BASE` 允许在不同环境指向不同的后端地址，开发用 localhost，生产用实际域名。

## Spring Boot 后端：REST API

后端部分用 Spring Boot 3.x，配合 JPA 来操作数据库。先看 Controller：

```java
// backend/src/main/java/com/example/controller/TaskController.java
package com.example.controller;

import com.example.model.Task;
import com.example.service.TaskService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api")
@CrossOrigin(origins = "http://localhost:5173", allowCredentials = "true")
public class TaskController {

    private final TaskService taskService;

    public TaskController(TaskService taskService) {
        this.taskService = taskService;
    }

    @GetMapping("/projects/{projectId}/tasks")
    public ResponseEntity<List<Task>> getTasks(
        @PathVariable String projectId,
        @RequestParam(required = false) Boolean completed
    ) {
        List<Task> tasks = taskService.findByProject(projectId, completed);
        return ResponseEntity.ok(tasks);
    }

    @GetMapping("/tasks/{taskId}")
    public ResponseEntity<Task> getTask(@PathVariable String taskId) {
        return taskService.findById(taskId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping("/projects/{projectId}/tasks")
    public ResponseEntity<Task> createTask(
        @PathVariable String projectId,
        @Valid @RequestBody CreateTaskRequest request
    ) {
        Task task = taskService.create(projectId, request);
        return ResponseEntity.status(HttpStatus.CREATED).body(task);
    }

    @PatchMapping("/tasks/{taskId}")
    public ResponseEntity<Task> updateTask(
        @PathVariable String taskId,
        @RequestBody UpdateTaskRequest request
    ) {
        return taskService.update(taskId, request)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/tasks/{taskId}")
    public ResponseEntity<Void> deleteTask(@PathVariable String taskId) {
        taskService.delete(taskId);
        return ResponseEntity.noContent().build();
    }
}
```

Controller 层只负责 HTTP 层面的事情：接收请求、参数校验、返回响应。具体的业务逻辑委托给 Service 层。

## CORS 配置

`@CrossOrigin` 注解处理的是单个接口级别的跨域。如果项目里接口多，通常会在配置类里统一处理：

```java
// backend/src/main/java/com/example/config/WebConfig.java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.filter.CorsFilter;

@Configuration
public class WebConfig {

    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        
        // 允许的前端地址
        config.addAllowedOrigin("http://localhost:5173");
        config.addAllowedOrigin("http://localhost:3000");
        
        // 允许携带凭证（cookie）
        config.setAllowCredentials(true);
        
        // 允许的请求头
        config.addAllowedHeader("*");
        
        // 允许的 HTTP 方法
        config.addAllowedMethod("GET");
        config.addAllowedMethod("POST");
        config.addAllowedMethod("PATCH");
        config.addAllowedMethod("DELETE");
        config.addAllowedMethod("OPTIONS");
        
        // 缓存时间
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        
        return new CorsFilter(source);
    }
}
```

关键点在于 `setAllowCredentials(true)` 配合 `addAllowedOrigin` 不能用通配符 `*`，必须指定具体域名。踩过这个坑的人不少——前端带 cookie 发请求，结果被 CORS 拦截，问题就出在这里。

## Spring Boot Service 层

```java
// backend/src/main/java/com/example/service/TaskService.java
package com.example.service;

import com.example.model.Task;
import com.example.repository.TaskRepository;
import com.example.controller.CreateTaskRequest;
import com.example.controller.UpdateTaskRequest;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

@Service
@Transactional
public class TaskService {

    private final TaskRepository taskRepository;

    public TaskService(TaskRepository taskRepository) {
        this.taskRepository = taskRepository;
    }

    public List<Task> findByProject(String projectId, Boolean completed) {
        if (completed != null) {
            return taskRepository.findByProjectIdAndCompleted(projectId, completed);
        }
        return taskRepository.findByProjectId(projectId);
    }

    public Optional<Task> findById(String taskId) {
        return taskRepository.findById(taskId);
    }

    public Task create(String projectId, CreateTaskRequest request) {
        Task task = new Task();
        task.setProjectId(projectId);
        task.setTitle(request.title());
        task.setDescription(request.description());
        task.setCompleted(false);
        task.setCreatedAt(Instant.now());
        task.setUpdatedAt(Instant.now());
        return taskRepository.save(task);
    }

    public Optional<Task> update(String taskId, UpdateTaskRequest request) {
        return taskRepository.findById(taskId)
            .map(task -> {
                request.title().ifPresent(task::setTitle);
                request.description().ifPresent(task::setDescription);
                request.completed().ifPresent(task::setCompleted);
                task.setUpdatedAt(Instant.now());
                return taskRepository.save(task);
            });
    }

    public void delete(String taskId) {
        taskRepository.deleteById(taskId);
    }
}
```

用了 `@Transactional` 确保数据库操作的一致性。`Optional` 的使用让更新操作可以优雅地处理"找不到记录"的情况，不需要抛异常。

## 请求 DTO 定义

```java
// backend/src/main/java/com/example/controller/CreateTaskRequest.java
package com.example.controller;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record CreateTaskRequest(
    @NotBlank(message = "标题不能为空")
    @Size(min = 1, max = 200, message = "标题长度必须在1-200之间")
    String title,
    
    @Size(max = 2000, message = "描述不能超过2000字")
    String description
) {}

// backend/src/main/java/com/example/controller/UpdateTaskRequest.java
package com.example.controller;

import jakarta.validation.constraints.Size;
import java.util.Optional;

public record UpdateTaskRequest(
    Optional<String> title,
    Optional<String> description,
    Optional<Boolean> completed
) {}
```

用了 Java 的 `record` 来定义 DTO，比传统的 class 更简洁。`Optional` 字段在 PATCH 请求里特别有用——字段不存在表示不更新，`null` 和"不传"能区分开。

## Cursor 里的前后端联动

Cursor 的一个强大之处在于跨文件上下文感知。当你在 `TaskList.tsx` 里写 `taskApi.getTasks(projectId)` 的时候，按住 Cmd 点击这个方法名，Cursor 能直接跳转到 `tasks.ts` 里的实现，再按一次跳到 `TaskController.java`。

这种跨语言跳转的能力，在调试全栈问题时特别有用。前端报 bug，Cursor 可以顺着调用链一路找到后端，告诉你数据从哪来、经过了哪些处理、最终返回了什么。整个链路不需要切换工具，不需要手动搜索。

![Spring Boot 小熊和 React 小人协作编程](/tmp/fantasy-blogs/vibe-coding-series/images/11-fullstack-spring-react.jpg)

另外，Cursor 的 Composer 模式也适合全栈开发。在 Composer 里同时打开前后端的多个文件，AI 在生成代码时能看到完整的上下文，不会出现前端生成了一套接口、后端发现参数对不上的情况。

## 状态管理：Zustand 轻量方案

任务列表组件里用到了 `useTaskStore`，这是 Zustand 提供的全局状态管理。相比 Redux，Zustand 的 API 简单很多，适合中小型项目：

```typescript
// frontend/src/store/taskStore.ts
import { create } from 'zustand';

interface TaskStore {
  selectedTaskId: string | null;
  setSelectedTaskId: (id: string | null) => void;
  searchQuery: string;
  setSearchQuery: (query: string) => void;
}

export const useTaskStore = create<TaskStore>((set) => ({
  selectedTaskId: null,
  setSelectedTaskId: (id) => set({ selectedTaskId: id }),
  searchQuery: '',
  setSearchQuery: (query) => set({ searchQuery: query }),
}));
```

Zustand 的 store 就是一个 React Hook，返回的状态在组件间共享，修改状态不需要 Provider 包裹。这种轻量方案对于大多数全栈应用来说足够了，不需要引入 Redux 那一套复杂的模板。

## 结语

Spring Boot 和 React 的组合在全栈领域久经考验，但把它们真正串起来跑通需要处理不少细节：CORS、请求封装、状态管理、类型对应。Cursor 的价值在于让这些琐碎工作变得不那么枯燥——它能同时理解两个世界的代码，在切换上下文时保持连贯，减少"mental overhead"。

下篇我们来聊部署：怎么把写好的前后端代码打包成 Docker 镜像，用 GitHub Actions 做 CI/CD，最终推到生产环境。
