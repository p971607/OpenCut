# OpenCut 视频编辑 API 接口方案

## 🎯 项目目标

为OpenCut视频编辑器添加完整的REST API接口，让外部项目可以通过API调用的方式使用OpenCut的视频编辑功能。

## 🏗️ API架构设计

### 核心理念
- **无状态API** - 每个请求独立处理
- **异步处理** - 视频处理任务异步执行
- **RESTful设计** - 标准的REST API规范
- **任务队列** - 支持并发处理多个视频任务

### 系统架构
```
外部项目 → OpenCut API → 任务队列 → 视频处理引擎 → 结果返回
```

## 📋 API接口设计

### 1. 项目管理接口

#### 创建项目
```http
POST /api/v1/projects
Content-Type: application/json

{
  "name": "我的视频项目",
  "width": 1920,
  "height": 1080,
  "fps": 30,
  "duration": 60
}

Response:
{
  "projectId": "proj_123456",
  "name": "我的视频项目",
  "settings": {
    "width": 1920,
    "height": 1080,
    "fps": 30,
    "duration": 60
  },
  "createdAt": "2024-01-01T00:00:00Z"
}
```

#### 获取项目信息
```http
GET /api/v1/projects/{projectId}

Response:
{
  "projectId": "proj_123456",
  "name": "我的视频项目",
  "settings": {...},
  "tracks": [...],
  "status": "ready",
  "createdAt": "2024-01-01T00:00:00Z",
  "updatedAt": "2024-01-01T00:00:00Z"
}
```

### 2. 媒体资源管理接口

#### 上传媒体文件
```http
POST /api/v1/projects/{projectId}/media
Content-Type: multipart/form-data

file: <video/image/audio file>
name: "素材名称"

Response:
{
  "mediaId": "media_789",
  "name": "素材名称",
  "type": "video",
  "duration": 30.5,
  "width": 1920,
  "height": 1080,
  "url": "https://cdn.example.com/media_789.mp4",
  "thumbnailUrl": "https://cdn.example.com/thumb_789.jpg"
}
```

#### 通过URL添加媒体
```http
POST /api/v1/projects/{projectId}/media/url
Content-Type: application/json

{
  "url": "https://example.com/video.mp4",
  "name": "远程视频"
}

Response:
{
  "mediaId": "media_790",
  "name": "远程视频",
  "type": "video",
  "url": "https://example.com/video.mp4",
  "status": "processing"
}
```

### 3. 时间轴编辑接口

#### 添加媒体到时间轴
```http
POST /api/v1/projects/{projectId}/timeline/elements
Content-Type: application/json

{
  "mediaId": "media_789",
  "trackType": "video",
  "startTime": 0,
  "duration": 10,
  "trimStart": 0,
  "trimEnd": 0,
  "position": {
    "x": 0,
    "y": 0,
    "scale": 1.0,
    "rotation": 0
  }
}

Response:
{
  "elementId": "elem_456",
  "mediaId": "media_789",
  "trackType": "video",
  "startTime": 0,
  "duration": 10,
  "properties": {...}
}
```

#### 添加文字元素
```http
POST /api/v1/projects/{projectId}/timeline/text
Content-Type: application/json

{
  "content": "这是标题文字",
  "startTime": 5,
  "duration": 3,
  "style": {
    "fontSize": 48,
    "fontFamily": "Arial",
    "color": "#FFFFFF",
    "backgroundColor": "#000000",
    "textAlign": "center"
  },
  "position": {
    "x": 0,
    "y": -100
  }
}

Response:
{
  "elementId": "text_789",
  "content": "这是标题文字",
  "startTime": 5,
  "duration": 3,
  "style": {...},
  "position": {...}
}
```

#### 修改时间轴元素
```http
PUT /api/v1/projects/{projectId}/timeline/elements/{elementId}
Content-Type: application/json

{
  "startTime": 2,
  "duration": 8,
  "position": {
    "x": 100,
    "y": 50,
    "scale": 1.2
  }
}

Response:
{
  "elementId": "elem_456",
  "startTime": 2,
  "duration": 8,
  "position": {...},
  "updatedAt": "2024-01-01T00:00:00Z"
}
```

### 4. 视频渲染接口

#### 开始渲染
```http
POST /api/v1/projects/{projectId}/render
Content-Type: application/json

{
  "format": "mp4",
  "quality": "high",
  "resolution": "1920x1080",
  "fps": 30
}

Response:
{
  "taskId": "task_render_123",
  "status": "queued",
  "estimatedTime": 120,
  "createdAt": "2024-01-01T00:00:00Z"
}
```

#### 查询渲染状态
```http
GET /api/v1/tasks/{taskId}

Response:
{
  "taskId": "task_render_123",
  "status": "processing",
  "progress": 45,
  "estimatedTimeRemaining": 60,
  "result": null,
  "error": null
}
```

#### 渲染完成响应
```http
GET /api/v1/tasks/{taskId}

Response:
{
  "taskId": "task_render_123",
  "status": "completed",
  "progress": 100,
  "result": {
    "videoUrl": "https://cdn.example.com/rendered_video_123.mp4",
    "duration": 30.5,
    "fileSize": 15728640,
    "format": "mp4"
  },
  "completedAt": "2024-01-01T00:05:00Z"
}
```

### 5. 批量操作接口

#### 批量创建项目并渲染
```http
POST /api/v1/batch/create-and-render
Content-Type: application/json

{
  "projects": [
    {
      "name": "视频1",
      "elements": [
        {
          "type": "media",
          "mediaUrl": "https://example.com/video1.mp4",
          "startTime": 0,
          "duration": 10
        },
        {
          "type": "text",
          "content": "标题1",
          "startTime": 2,
          "duration": 3
        }
      ]
    },
    {
      "name": "视频2",
      "elements": [...]
    }
  ],
  "renderSettings": {
    "format": "mp4",
    "quality": "high"
  }
}

Response:
{
  "batchId": "batch_456",
  "tasks": [
    {
      "projectId": "proj_001",
      "taskId": "task_001",
      "status": "queued"
    },
    {
      "projectId": "proj_002", 
      "taskId": "task_002",
      "status": "queued"
    }
  ]
}
```

## 🔧 技术实现方案

### 1. API层架构

```typescript
// apps/web/src/app/api/v1/
├── projects/
│   ├── route.ts                    # POST /projects
│   └── [projectId]/
│       ├── route.ts                # GET /projects/{id}
│       ├── media/
│       │   ├── route.ts            # POST /projects/{id}/media
│       │   └── url/
│       │       └── route.ts        # POST /projects/{id}/media/url
│       ├── timeline/
│       │   ├── elements/
│       │   │   ├── route.ts        # POST /timeline/elements
│       │   │   └── [elementId]/
│       │   │       └── route.ts    # PUT /timeline/elements/{id}
│       │   └── text/
│       │       └── route.ts        # POST /timeline/text
│       └── render/
│           └── route.ts            # POST /projects/{id}/render
├── tasks/
│   └── [taskId]/
│       └── route.ts                # GET /tasks/{id}
└── batch/
    └── create-and-render/
        └── route.ts                # POST /batch/create-and-render
```

### 2. 核心服务类

```typescript
// apps/web/src/lib/api-services/
├── project-service.ts              # 项目管理服务
├── media-service.ts                # 媒体处理服务  
├── timeline-service.ts             # 时间轴编辑服务
├── render-service.ts               # 视频渲染服务
├── task-manager.ts                 # 任务队列管理
└── batch-processor.ts              # 批量处理服务
```

### 3. 数据库设计

```sql
-- API项目表
CREATE TABLE api_projects (
  id VARCHAR(255) PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  settings JSON NOT NULL,
  status ENUM('active', 'archived') DEFAULT 'active',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- API任务表
CREATE TABLE api_tasks (
  id VARCHAR(255) PRIMARY KEY,
  project_id VARCHAR(255),
  type ENUM('render', 'process', 'batch') NOT NULL,
  status ENUM('queued', 'processing', 'completed', 'failed') DEFAULT 'queued',
  progress INTEGER DEFAULT 0,
  result JSON,
  error_message TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP,
  FOREIGN KEY (project_id) REFERENCES api_projects(id) ON DELETE CASCADE
);

-- API媒体文件表
CREATE TABLE api_media (
  id VARCHAR(255) PRIMARY KEY,
  project_id VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  type ENUM('video', 'image', 'audio') NOT NULL,
  url VARCHAR(500) NOT NULL,
  thumbnail_url VARCHAR(500),
  metadata JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (project_id) REFERENCES api_projects(id) ON DELETE CASCADE
);
```

## 🚀 使用示例

### 外部项目调用示例

```typescript
// 你的小说项目中的API调用
class OpenCutVideoAPI {
  private baseUrl = 'https://your-opencut-api.com/api/v1';
  private apiKey = 'your-api-key';

  // 创建视频项目
  async createVideoProject(novelChapter: string, images: string[]): Promise<string> {
    // 1. 创建项目
    const project = await this.createProject({
      name: `小说章节视频_${Date.now()}`,
      width: 1920,
      height: 1080,
      fps: 30
    });

    // 2. 添加图片素材
    for (const imageUrl of images) {
      await this.addMediaFromUrl(project.projectId, imageUrl);
    }

    // 3. 添加文字内容
    await this.addTextElement(project.projectId, {
      content: novelChapter,
      startTime: 0,
      duration: 10
    });

    // 4. 开始渲染
    const renderTask = await this.startRender(project.projectId);
    
    return renderTask.taskId;
  }

  // 检查渲染状态
  async checkRenderStatus(taskId: string): Promise<RenderStatus> {
    const response = await fetch(`${this.baseUrl}/tasks/${taskId}`, {
      headers: { 'Authorization': `Bearer ${this.apiKey}` }
    });
    return response.json();
  }

  // 批量生成多个视频
  async batchCreateVideos(chapters: ChapterData[]): Promise<string> {
    const batchRequest = {
      projects: chapters.map(chapter => ({
        name: chapter.title,
        elements: [
          {
            type: 'media',
            mediaUrl: chapter.backgroundImage,
            startTime: 0,
            duration: chapter.estimatedDuration
          },
          {
            type: 'text', 
            content: chapter.content,
            startTime: 1,
            duration: chapter.estimatedDuration - 2
          }
        ]
      }))
    };

    const response = await fetch(`${this.baseUrl}/batch/create-and-render`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.apiKey}`
      },
      body: JSON.stringify(batchRequest)
    });

    const result = await response.json();
    return result.batchId;
  }
}
```

## 🔒 安全和认证

### API密钥认证
```typescript
// 中间件验证API密钥
export async function authenticateAPI(request: NextRequest) {
  const apiKey = request.headers.get('Authorization')?.replace('Bearer ', '');
  
  if (!apiKey || !await validateApiKey(apiKey)) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  return null; // 认证通过
}
```

### 速率限制
```typescript
// 基于Redis的速率限制
export const apiRateLimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(100, "1 h"), // 每小时100次请求
  analytics: true,
});
```

## 📊 监控和日志

### API调用统计
```sql
CREATE TABLE api_usage_stats (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  api_key_id VARCHAR(255),
  endpoint VARCHAR(255),
  method VARCHAR(10),
  status_code INTEGER,
  response_time_ms INTEGER,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_api_key_date (api_key_id, created_at),
  INDEX idx_endpoint (endpoint)
);
```

### 错误处理
```typescript
// 统一错误响应格式
interface APIError {
  error: string;
  code: string;
  message: string;
  details?: any;
  timestamp: string;
}
```

---

## 🎯 总结

这个API方案让OpenCut成为一个强大的视频编辑服务，外部项目可以通过简单的HTTP请求来：

1. **创建视频项目** - 设置画布尺寸、帧率等
2. **管理媒体素材** - 上传或引用外部媒体文件  
3. **编辑时间轴** - 添加视频、图片、文字元素
4. **渲染输出** - 异步渲染并获取结果视频

你的小说项目只需要调用这些API，就能自动生成专业的视频内容，无需关心底层的视频处理细节！
