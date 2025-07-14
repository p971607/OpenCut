# OpenCut API 实现开发指南

## 🎯 开发目标

在现有OpenCut项目基础上，添加完整的REST API接口，让外部项目可以通过API调用使用视频编辑功能。

## 📁 项目文件结构

```
apps/web/src/
├── app/api/v1/                     # 🆕 API v1 路由
│   ├── projects/
│   │   ├── route.ts                # 项目CRUD
│   │   └── [projectId]/
│   │       ├── route.ts
│   │       ├── media/
│   │       ├── timeline/
│   │       └── render/
│   ├── tasks/
│   └── batch/
├── lib/
│   ├── api-services/               # 🆕 API服务层
│   │   ├── project-api-service.ts
│   │   ├── media-api-service.ts
│   │   ├── timeline-api-service.ts
│   │   ├── render-api-service.ts
│   │   └── task-manager.ts
│   ├── api-middleware/             # 🆕 API中间件
│   │   ├── auth.ts
│   │   ├── rate-limit.ts
│   │   └── error-handler.ts
│   └── api-types/                  # 🆕 API类型定义
│       ├── project.ts
│       ├── media.ts
│       └── task.ts
└── stores/                         # 现有store需要适配API
    ├── project-store.ts            # 🔄 需要修改
    ├── timeline-store.ts           # 🔄 需要修改
    └── media-store.ts              # 🔄 需要修改
```

## 🔧 核心实现代码

### 1. 项目管理API

```typescript
// apps/web/src/app/api/v1/projects/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { ProjectAPIService } from '@/lib/api-services/project-api-service';
import { authenticateAPI } from '@/lib/api-middleware/auth';
import { apiRateLimit } from '@/lib/api-middleware/rate-limit';

export async function POST(request: NextRequest) {
  // 认证检查
  const authError = await authenticateAPI(request);
  if (authError) return authError;

  // 速率限制
  const rateLimitError = await apiRateLimit.check(request);
  if (rateLimitError) return rateLimitError;

  try {
    const body = await request.json();
    const projectService = new ProjectAPIService();
    const project = await projectService.createProject(body);
    
    return NextResponse.json(project, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to create project', message: error.message },
      { status: 500 }
    );
  }
}

export async function GET(request: NextRequest) {
  const authError = await authenticateAPI(request);
  if (authError) return authError;

  try {
    const projectService = new ProjectAPIService();
    const projects = await projectService.listProjects();
    
    return NextResponse.json({ projects });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch projects' },
      { status: 500 }
    );
  }
}
```

### 2. 项目服务类

```typescript
// apps/web/src/lib/api-services/project-api-service.ts
import { generateUUID } from '@/lib/utils';
import { db } from '@opencut/db';
import { apiProjects } from '@opencut/db/schema';

export interface CreateProjectRequest {
  name: string;
  width?: number;
  height?: number;
  fps?: number;
  duration?: number;
}

export interface APIProject {
  projectId: string;
  name: string;
  settings: {
    width: number;
    height: number;
    fps: number;
    duration: number;
  };
  status: 'active' | 'archived';
  createdAt: string;
  updatedAt: string;
}

export class ProjectAPIService {
  async createProject(data: CreateProjectRequest): Promise<APIProject> {
    const projectId = generateUUID();
    const settings = {
      width: data.width || 1920,
      height: data.height || 1080,
      fps: data.fps || 30,
      duration: data.duration || 60
    };

    const project = await db.insert(apiProjects).values({
      id: projectId,
      name: data.name,
      settings: JSON.stringify(settings),
      status: 'active'
    }).returning();

    return {
      projectId: project[0].id,
      name: project[0].name,
      settings,
      status: project[0].status,
      createdAt: project[0].createdAt.toISOString(),
      updatedAt: project[0].updatedAt.toISOString()
    };
  }

  async getProject(projectId: string): Promise<APIProject | null> {
    const project = await db.query.apiProjects.findFirst({
      where: (projects, { eq }) => eq(projects.id, projectId)
    });

    if (!project) return null;

    return {
      projectId: project.id,
      name: project.name,
      settings: JSON.parse(project.settings),
      status: project.status,
      createdAt: project.createdAt.toISOString(),
      updatedAt: project.updatedAt.toISOString()
    };
  }

  async listProjects(): Promise<APIProject[]> {
    const projects = await db.query.apiProjects.findMany({
      where: (projects, { eq }) => eq(projects.status, 'active'),
      orderBy: (projects, { desc }) => [desc(projects.createdAt)]
    });

    return projects.map(project => ({
      projectId: project.id,
      name: project.name,
      settings: JSON.parse(project.settings),
      status: project.status,
      createdAt: project.createdAt.toISOString(),
      updatedAt: project.updatedAt.toISOString()
    }));
  }

  async deleteProject(projectId: string): Promise<boolean> {
    const result = await db.update(apiProjects)
      .set({ status: 'archived' })
      .where(eq(apiProjects.id, projectId));
    
    return result.rowCount > 0;
  }
}
```

### 3. 媒体管理API

```typescript
// apps/web/src/app/api/v1/projects/[projectId]/media/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { MediaAPIService } from '@/lib/api-services/media-api-service';

export async function POST(
  request: NextRequest,
  { params }: { params: { projectId: string } }
) {
  try {
    const projectId = params.projectId;
    const formData = await request.formData();
    const file = formData.get('file') as File;
    const name = formData.get('name') as string;

    if (!file) {
      return NextResponse.json(
        { error: 'No file provided' },
        { status: 400 }
      );
    }

    const mediaService = new MediaAPIService();
    const media = await mediaService.uploadMedia(projectId, file, name);
    
    return NextResponse.json(media, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to upload media' },
      { status: 500 }
    );
  }
}
```

### 4. 时间轴编辑API

```typescript
// apps/web/src/app/api/v1/projects/[projectId]/timeline/elements/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { TimelineAPIService } from '@/lib/api-services/timeline-api-service';

export async function POST(
  request: NextRequest,
  { params }: { params: { projectId: string } }
) {
  try {
    const projectId = params.projectId;
    const body = await request.json();
    
    const timelineService = new TimelineAPIService();
    const element = await timelineService.addElement(projectId, body);
    
    return NextResponse.json(element, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to add timeline element' },
      { status: 500 }
    );
  }
}
```

### 5. 视频渲染API

```typescript
// apps/web/src/app/api/v1/projects/[projectId]/render/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { RenderAPIService } from '@/lib/api-services/render-api-service';

export async function POST(
  request: NextRequest,
  { params }: { params: { projectId: string } }
) {
  try {
    const projectId = params.projectId;
    const body = await request.json();
    
    const renderService = new RenderAPIService();
    const task = await renderService.startRender(projectId, body);
    
    return NextResponse.json(task, { status: 202 }); // 202 Accepted
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to start render' },
      { status: 500 }
    );
  }
}
```

### 6. 渲染服务实现

```typescript
// apps/web/src/lib/api-services/render-api-service.ts
import { TaskManager } from './task-manager';
import { generateUUID } from '@/lib/utils';

export interface RenderRequest {
  format?: 'mp4' | 'webm';
  quality?: 'low' | 'medium' | 'high';
  resolution?: string;
  fps?: number;
}

export interface RenderTask {
  taskId: string;
  status: 'queued' | 'processing' | 'completed' | 'failed';
  progress: number;
  estimatedTime?: number;
  result?: {
    videoUrl: string;
    duration: number;
    fileSize: number;
    format: string;
  };
  error?: string;
  createdAt: string;
  completedAt?: string;
}

export class RenderAPIService {
  private taskManager = new TaskManager();

  async startRender(projectId: string, options: RenderRequest): Promise<RenderTask> {
    const taskId = generateUUID();
    
    // 创建渲染任务
    const task: RenderTask = {
      taskId,
      status: 'queued',
      progress: 0,
      estimatedTime: 120, // 预估2分钟
      createdAt: new Date().toISOString()
    };

    // 添加到任务队列
    await this.taskManager.addTask(taskId, {
      type: 'render',
      projectId,
      options
    });

    // 异步开始处理
    this.processRenderTask(taskId, projectId, options);

    return task;
  }

  private async processRenderTask(
    taskId: string, 
    projectId: string, 
    options: RenderRequest
  ) {
    try {
      // 更新状态为处理中
      await this.taskManager.updateTaskStatus(taskId, 'processing', 0);

      // 1. 获取项目数据
      const projectData = await this.getProjectData(projectId);
      await this.taskManager.updateTaskStatus(taskId, 'processing', 20);

      // 2. 准备渲染素材
      const assets = await this.prepareAssets(projectData);
      await this.taskManager.updateTaskStatus(taskId, 'processing', 40);

      // 3. 使用FFmpeg渲染视频
      const videoBlob = await this.renderVideo(projectData, assets, options);
      await this.taskManager.updateTaskStatus(taskId, 'processing', 80);

      // 4. 上传到CDN
      const videoUrl = await this.uploadVideo(videoBlob, taskId);
      await this.taskManager.updateTaskStatus(taskId, 'processing', 100);

      // 5. 完成任务
      const result = {
        videoUrl,
        duration: projectData.duration,
        fileSize: videoBlob.size,
        format: options.format || 'mp4'
      };

      await this.taskManager.completeTask(taskId, result);

    } catch (error) {
      await this.taskManager.failTask(taskId, error.message);
    }
  }

  async getTaskStatus(taskId: string): Promise<RenderTask | null> {
    return await this.taskManager.getTask(taskId);
  }
}
```

## 🔒 认证中间件

```typescript
// apps/web/src/lib/api-middleware/auth.ts
import { NextRequest, NextResponse } from 'next/server';
import { db } from '@opencut/db';

export async function authenticateAPI(request: NextRequest) {
  const authHeader = request.headers.get('Authorization');
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return NextResponse.json(
      { error: 'Missing or invalid authorization header' },
      { status: 401 }
    );
  }

  const apiKey = authHeader.replace('Bearer ', '');
  
  // 验证API密钥
  const isValid = await validateApiKey(apiKey);
  if (!isValid) {
    return NextResponse.json(
      { error: 'Invalid API key' },
      { status: 401 }
    );
  }

  return null; // 认证通过
}

async function validateApiKey(apiKey: string): Promise<boolean> {
  // 这里实现API密钥验证逻辑
  // 可以从数据库查询或使用JWT验证
  return apiKey === process.env.OPENCUT_API_KEY;
}
```

## 📊 数据库Schema扩展

```typescript
// packages/db/src/schema.ts - 添加API相关表

export const apiProjects = pgTable("api_projects", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  settings: text("settings").notNull(), // JSON string
  status: text("status").$type<'active' | 'archived'>().default('active'),
  createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
  updatedAt: timestamp("updated_at").$defaultFn(() => new Date()).notNull(),
});

export const apiTasks = pgTable("api_tasks", {
  id: text("id").primaryKey(),
  projectId: text("project_id").references(() => apiProjects.id),
  type: text("type").$type<'render' | 'process' | 'batch'>().notNull(),
  status: text("status").$type<'queued' | 'processing' | 'completed' | 'failed'>().default('queued'),
  progress: integer("progress").default(0),
  result: text("result"), // JSON string
  errorMessage: text("error_message"),
  createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
  completedAt: timestamp("completed_at"),
});

export const apiMedia = pgTable("api_media", {
  id: text("id").primaryKey(),
  projectId: text("project_id").references(() => apiProjects.id).notNull(),
  name: text("name").notNull(),
  type: text("type").$type<'video' | 'image' | 'audio'>().notNull(),
  url: text("url").notNull(),
  thumbnailUrl: text("thumbnail_url"),
  metadata: text("metadata"), // JSON string
  createdAt: timestamp("created_at").$defaultFn(() => new Date()).notNull(),
});
```

## 🚀 部署配置

### 环境变量
```bash
# .env.local 添加
OPENCUT_API_KEY=your-secret-api-key
API_RATE_LIMIT_PER_HOUR=100
MAX_CONCURRENT_RENDERS=3
VIDEO_STORAGE_PATH=/path/to/video/storage
CDN_BASE_URL=https://your-cdn.com
```

### Docker配置
```dockerfile
# 如果需要服务端FFmpeg处理
FROM node:18-alpine
RUN apk add --no-cache ffmpeg
# ... 其他配置
```

---

## 🎯 使用示例

```bash
# 创建项目
curl -X POST https://your-opencut-api.com/api/v1/projects \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "我的视频项目",
    "width": 1920,
    "height": 1080,
    "fps": 30
  }'

# 上传媒体
curl -X POST https://your-opencut-api.com/api/v1/projects/proj_123/media \
  -H "Authorization: Bearer your-api-key" \
  -F "file=@video.mp4" \
  -F "name=背景视频"

# 开始渲染
curl -X POST https://your-opencut-api.com/api/v1/projects/proj_123/render \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "format": "mp4",
    "quality": "high"
  }'
```

这个实现方案让OpenCut成为一个完整的视频编辑API服务，你的小说项目可以轻松调用这些接口来生成视频！
