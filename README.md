Hệ thống Version Control cho phép bạn ghi lại nhiều phiên bản của một project hoặc file, có thể quay lại bất kỳ thời điểm nào. Git là một ví dụ về hệ thống version control.

Trong tutorial này, bạn sẽ xây dựng một hệ thống version control được hỗ trợ bởi Cloudflare để track các thay đổi đối với content như file text hoặc code. Cloudflare Workers sẽ được sử dụng để handle các function như tạo version cho content, và Durable Objects sẽ được sử dụng để ghi lại các version state.

Các function chúng ta sẽ đề cập trong tutorial này gồm:

- Create content mới
- Update content
- Get content (version cụ thể và list tất cả version)
- Revert về một version cụ thể
- Publish hoặc unpublish một version
- Lịch sử chi tiết về các modification
- Tag một version
- Delete một version (trong trường hợp không cần thiết nữa)

# Prerequisites:

- Cần có tài khoản Cloudflare với Workers Paid plan để sử dụng Durable Objects

# 1. Thiết lập Worker project và Durable Objects

Để tạo Worker project có thể sử dụng Durable Objects, chạy lệnh C3 (create-cloudflare-cli) trong CLI của bạn.

```yaml
mkdir content-version-system
cd content-version-system

npm create cloudflare@latest
```

Sau khi chạy lệnh trên, bạn sẽ được yêu cầu cung cấp thông tin về loại project bạn muốn tạo. Sử dụng các giá trị sau:

```yaml
Ok to proceed? (y) y
In which directory do you want to create your application?
│ ./content-version-system
├ What would you like to start with?
│ Hello World example
│
├ Which template would you like to use?
│ Hello World Worker Using Durable Objects
│
├ Which language do you want to use?
│ TypeScript
├ Do you want to use git for version control?
│ yes
├ Do you want to deploy your application?
│ no
```

## Cấu hình Worker và Durable Object binding

Sau khi `C3` hoàn thành, bạn sẽ có một Workers project với tên `content-version-system` chứa công cụ phát triển Wrangler của Cloudflare cùng các file và thư mục sau:

```yaml
~/content-version-system/content-version-system# ls

node_modules  package.json  package-lock.json  src  tsconfig.json  worker-configuration.d.ts  wrangler.toml

```

Cập nhật các giá trị trong tệp cấu hình `wrangler.toml` của bạn để khớp với những configuration sau:

```yaml
name = "content-version-system"
main = "src/index.ts"
compatibility_date = "2024-11-06"

[observability]
enabled = true

[placement]
mode = "smart"

[[durable_objects.bindings]]
name = "CONTENT"
class_name = "ContentDO"

[[migrations]]
tag = "v1"
new_classes = ["ContentDO"]
```

## Cài đặt các package cần thiết

Cài đặt các gói bổ sung mà chúng ta sẽ cần cho dự án này:

```yaml
# Install runtime dependencies
npm install diff

# Install development dependencies for linting, formatting and testing
npm install --save-dev @types/diff @typescript-eslint/eslint-plugin @typescript-eslint/parser eslint prettier vitest
```

Sau đó, thêm các scripts sau vào `package.json` :

```yaml
{
  "scripts": {
    "lint": "npx eslint . --ext .ts --fix",
    "format": "prettier --write .",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

Những bổ sung này cho phép:

- Runtime features:
    - `diff`: Triển khai so sánh văn bản cho JavaScript
- Development tools:
    - Linting and formatting với ESLint và Prettier
    - Testing với Vitest
    - TypeScript type definitions

# 2. Định nghĩa cấu trúc dữ liệu cho version control system

Tạo các file 'types.ts' và 'contentDO.ts' trong thư mục 'src'. Cấu trúc file sẽ như sau:

```yaml
.src
    └───types.ts     <--- Interface definitions for data structures
    └───index.ts     <--- Entry point for request routing and presentation
    └───contentDO.ts <--- Core business logic
```

Các cấu trúc dữ liệu sẽ được sử dụng bởi Durable Object và Worker để handle state management cần được định nghĩa trước. Trong bước này, những dữ liệu sau sẽ được định nghĩa:

- Các state khác nhau mà một content version có thể có (`draft`, `published` và `archived`)
- Cấu trúc dữ liệu cốt lõi cho version management, như version id number, content chứa trong đó và version message đính kèm
- Cấu trúc dữ liệu để track các version event và history
- Cấu trúc API response

Thêm đoạn code sau vào file `src/types.ts`:

```yaml
// Core version statuses - Define possible states of a content version
export enum VersionStatus {
    DRAFT = 'draft',
    PUBLISHED = 'published',
    ARCHIVED = 'archived'
  }
  
  // Version & tag structures - Define core data structures for version management
  export interface Version {
    id: number;
    content: string;
    timestamp: string;
    message: string;
    diff?: ContentDiff;
    status: VersionStatus;
  }
  
  export interface Tag {
    name: string;
    versionId: number;
    createdAt: string;
    updatedAt?: string;
  }
  
  // State management - Define how content versions and states are tracked
  export interface ContentState {
    currentVersion: number;
    versions: Version[];
    tags: { [key: string]: Tag };
    content: string | null;
    publishHistory: PublishRecord[];
  }
  
  // Change tracking - tracking differences between versions
  export interface ContentDiff {
    from: string;
    to: string;
    changes: {
      additions: number;
      deletions: number;
      totalChanges: number;
      timestamp: string;
    };
    patch: string;
    hunks: Array<{
      oldStart: number;
      oldLines: number;
      newStart: number;
      newLines: number;
      lines: string[];
    }>;
  }
  
  // Publishing system - Track publishing events and history
  export interface PublishRecord {
    versionId: number;
    publishedAt: string;
    publishedBy: string;
  }
  
  // API response types - Standardized response structures for the API
  export interface ApiResponse<T> {
    success: boolean;
    data?: T;
    error?: string;
  }
  
  export interface DeleteVersionResponse {
    success: boolean;
    message: string;
  }
  
  export interface TagResponse {
    tagName: string;
    versionId: number;
  }
  
  export interface CurrentVersionResponse {
    version: number;
    content: string | null;
  }

  // Interface for API requests
  export interface CreateVersionRequest {
    content: string;
    message: string;
  }
  
  export interface PublishVersionRequest {
    publishedBy: string;
  }
  
  export interface CreateTagRequest {
    versionId: number;
    name: string;
  }
  
  export interface UpdateTagRequest {
    newName: string;
  }
  
  export interface RevertVersionRequest {
    versionId: number;
  }
  
  export type VersionListItem = Omit<Version, 'content' | 'diff'>;
```

**Key Components:**

1. **Version Status (VersionStatus)**
    - Xác định ba trạng thái có thể có cho các phiên bản nội dung:
    - `draft`: Trạng thái ban đầu cho các phiên bản mới
    - `published`: Phiên bản hiện đang hoạt động/trực tiếp
    - `archived`: Phiên bản lịch sử không còn được sử dụng
2. **Version Management (Version & Tag)**
    - `Version`: Cấu trúc cốt lõi chứa:
        - Unique ID, content, timestamp
        - Version message (như thông báo commit của Git)
        - Optional diff information
        - Trạng thái hiện tại
    - `Tag`: Tham chiếu được đặt tên cho các phiên bản cụ thể (tương tự như tag của Git)
3. **State Management (ContentState)**
    - Theo dõi trạng thái tổng thể của hệ thống:
        - Phiên bản hoạt động hiện tại
        - Danh sách tất cả các phiên bản
        - Ánh xạ tag
        - Nội dung hiện tại
        - Lịch sử xuất bản
4. **Change Tracking (ContentDiff)**
    - Thông tin khác biệt chi tiết giữa các phiên bản
    - Theo dõi các bổ sung, xóa và tổng số thay đổi
    - Bao gồm thông tin vá lỗi để hiển thị sự khác biệt
5. **Publishing System (PublishRecord)**
    - Ghi lại khi nào và ai đã xuất bản các phiên bản
    - Duy trì dấu vết kiểm toán các sự kiện xuất bản
6. **API Responses**
    - Cấu trúc phản hồi được tiêu chuẩn hóa
    - Bao gồm thông tin thành công/lỗi
    - Dữ liệu phản hồi an toàn kiểu

# 3. Xử lý requests gửi đến Durable Object

Cloudflare Workers sẽ định tuyến các incoming requests đến instance của Durable Objects. Instance của Durable Objects sau đó xử lý request và cung cấp response phù hợp cho Worker.

Đầu tiên, tạo kiểu TypeScript cho các liên kết Worker của bạn:

```
npm run cf-typegen
```

Lệnh này sẽ cập nhật tệp `worker-configuration.d.ts` của bạn với các định nghĩa kiểu chính xác cho các liên kết Durable Object của bạn.

Để xử lý các yêu cầu đến, hãy thay thế code trong `src/index.ts` bằng code sau:

## **Initial setup & HTML template**

```yaml
import { ContentDO } from './contentDO';
import { DurableObjectNamespace, DurableObjectStub } from '@cloudflare/workers-types';
import { Version } from './types';

type Env = {
  CONTENT: DurableObjectNamespace;
};

// Error handling types and helpers
interface ErrorWithMessage {
  message: string;
}

function isErrorWithMessage(error: unknown): error is ErrorWithMessage {
  return (
    typeof error === 'object' &&
    error !== null &&
    'message' in error &&
    typeof (error as Record<string, unknown>).message === 'string'
  );
}

function getErrorMessage(error: unknown): string {
  if (isErrorWithMessage(error)) {
    return error.message;
  }
  return 'Unknown error occurred';
}

// HTML Template & Styling
const getHtmlTemplate = (content: string, message: string = '', timestamp: string = '') => `
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Content Version System</title>
    <style>
        body {
            font-family: system-ui, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
        }
        .content {
            background: #f5f5f5;
            padding: 20px;
            border-radius: 8px;
            margin: 20px 0;
            white-space: pre-wrap;
        }
        .button {
            background: #0070f3;
            color: white;
            padding: 12px 24px;
            border-radius: 5px;
            text-decoration: none;
            display: inline-block;
            margin-top: 20px;
        }
        .button:hover {
            background: #0051a2;
        }
        pre {
            white-space: pre-wrap;
            word-wrap: break-word;
        }
        .meta {
            color: #666;
            font-size: 0.9em;
            margin-bottom: 10px;
        }
    </style>
</head>
<body>
    <h1>Content Version System</h1>
    <div class="content">
        <div class="meta">
            <strong>Message:</strong> ${message}<br>
            <strong>Last Updated:</strong> ${new Date(timestamp).toLocaleString()}
        </div>
        <h2>Current Content:</h2>
        <pre>${content}</pre>
    </div>
</body>
</html>
`;
// TODO: CORS and helper functions
```

Phần này cung cấp:
- Giao diện HTML rõ ràng để xem nội dung đã published
- Kiểu dáng cơ bản để dễ đọc hơn

## **CORS & helper functions**

Thiết lập CORS headers để cho phép các cross-origin requests, sau đó thêm một function để lấy published version mới nhất của content từ durable objects. Thêm đoạn code sau vào `src/index.ts`:

```yaml
const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'GET, HEAD, POST, PUT, OPTIONS, DELETE',
  'Access-Control-Allow-Headers': 'Content-Type',
};

// Helper function to get latest published version from Durable Objects
async function getLatestPublishedVersion(contentDO: DurableObjectStub, origin: string): Promise<Version | null> {
  const versionsResponse = await contentDO.fetch(`${origin}/content/default/versions`);
  const versions: Version[] = await versionsResponse.json(); // Xác định rõ ràng kiểu phản hồi

  const publishedVersions = versions.filter(v => v.status === 'published');
  if (publishedVersions.length === 0) {
    return null;
  }

  return publishedVersions.reduce((latest, current) => 
    latest.id > current.id ? latest : current
  );
}

// TODO: Request handler
```

Các tính năng chính:

- Thiết lập tiêu đề CORS cho các yêu cầu cross-origin

- Bao gồm hàm trợ giúp để fetch phiên bản mới nhất đã published

- Xử lý việc truy xuất phiên bản và các trường hợp lỗi

## **Request handler**

Thêm code sau vào `src/index.ts` để xử lý các yêu cầu cho Durable Object:

```yaml
export { ContentDO };

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    try {
      const url = new URL(request.url);

      // Handle CORS preflight
      if (request.method === 'OPTIONS') {
        return new Response(null, { headers: corsHeaders });
      }

      // Get Durable Objects instance
      const doId = env.CONTENT.idFromName('default');
      const contentDO = env.CONTENT.get(doId);

      // Handle root path - show HTML view
      if (url.pathname === '/') {
        try {
          const latestPublished = await getLatestPublishedVersion(contentDO, url.origin);
          // If there is published content, fetch it and display it
          if (latestPublished) {
            const contentResponse = await contentDO.fetch(
              `${url.origin}/content/default/versions/${latestPublished.id}`
            );
            const contentData: Version = await contentResponse.json();
            
            return new Response(
              getHtmlTemplate(
                contentData.content || 'No content available',
                contentData.message,
                contentData.timestamp
              ), {
                headers: { 'Content-Type': 'text/html' }
              }
            );
          } else {
            return new Response(
              getHtmlTemplate('No published content available', 'No published versions', ''), {
                headers: { 'Content-Type': 'text/html' }
              }
            );
          }
        } catch (error) {
          console.error('Root error:', error);
          return new Response(
            getHtmlTemplate('Error loading content', 'Error occurred', ''), {
              headers: { 'Content-Type': 'text/html' }
        ```typescript
    }
      );
        }
      }

      // Special handling for /content/default
      if (url.pathname === '/content/default') {
        try {
          const latestPublished = await getLatestPublishedVersion(contentDO, url.origin);
          
          if (latestPublished) {
            const contentResponse = await contentDO.fetch(
              `${url.origin}/content/default/versions/${latestPublished.id}`
            );
            const contentData: Version = await contentResponse.json();

            return new Response(JSON.stringify(contentData), {
              headers: {
                'Content-Type': 'application/json',
                ...corsHeaders
              }
            });
          } else {
            return new Response(JSON.stringify({ error: 'No published content available' }), {
              status: 404,
              headers: {
                'Content-Type': 'application/json',
                ...corsHeaders
              }
            });
          }
        } catch (error) {
          console.error('Content default error:', error);
          return new Response(JSON.stringify({ error: getErrorMessage(error) }), {
            status: 500,
            headers: {
              'Content-Type': 'application/json',
              ...corsHeaders
            }
          });
        }
      }

      // Forward all other requests to Durable Objects
      const response = await contentDO.fetch(request.url, {
        method: request.method,
        headers: request.headers,
        body: request.body
      });

      // Add CORS headers
      const newResponse = new Response(response.body, response);
      Object.entries(corsHeaders).forEach(([key, value]) => {
        newResponse.headers.set(key, value);
      });
      
      return newResponse;

    } catch (error) {
      console.error('Worker error:', error);
      return new Response(JSON.stringify({ 
        error: 'Internal Server Error',
        message: getErrorMessage(error)
      }), { 
        status: 500,
        headers: {
          'Content-Type': 'application/json',
          ...corsHeaders 
        }
      });
    }
  }
};
```

Trình xử lý yêu cầu (Request handler):

1. Xử lý các yêu cầu đến và định tuyến chúng một cách thích hợp
2. Xử lý các trường hợp đặc biệt cho đường dẫn gốc và nội dung mặc định
3. Chuyển tiếp các yêu cầu khác đến Durable Objects
4. Thêm tiêu đề CORS vào phản hồi
5. Cung cấp xử lý lỗi cho các yêu cầu không thành công

# 4. Xây dựng logic cốt lõi của hệ thống version control

Để xử lý các chức năng version control, thêm code sau vào `src/contentDO.ts`:

File này cho phép thực hiện các thao tác Create - Update - Delete - Read (CRUD) trên các version của content và version tags. Ngoài ra, có thể publish hoặc unpublish một version, lấy thông tin chi tiết về lịch sử thay đổi, so sánh các version (tương tự như lệnh `diff` của git), và quay lại một version cụ thể.

- Tất cả states được lưu trong Durable Object storage
- Các thao tác diễn ra theo thứ tự: đọc state → xử lý thay đổi → ghi thay đổi
- Tất cả thao tác đều là bất đồng bộ do giao tiếp với storage
- Có 2 điều quan trọng cần chú ý:
    1. Cần thiết lập CORS headers để cho phép cross-origin requests
    2. Tên class của Durable Object phải khớp với tên đã định nghĩa trong file wrangler.toml (nếu không sẽ gặp lỗi khi chạy lệnh `wrangler deploy`)

## Initial setup và class definition:

```yaml
import { createPatch } from 'diff';
import { 
  ContentDiff, 
  ContentState, 
  Version, 
  PublishRecord, 
  VersionStatus, 
  Tag, 
  CreateVersionRequest,
  PublishVersionRequest,
  CreateTagRequest,
  UpdateTagRequest,
  RevertVersionRequest } from './types';

// Lưu ý rằng tên của class phải khớp với tên của Durable Object được xác định trong tệp wrangler.toml.
export class ContentDO {
  private state: DurableObjectState;
  private env: any;

  constructor(state: DurableObjectState, env: any) {
    this.state = state;
    this.env = env;
  }
  // TODO: Core Version Control Functions
}
```

Phần này:

- Thiết lập các import và tiêu đề CORS cần thiết
- Xác định class Durable Object chính
- Khởi tạo quản lý trạng thái

**Lưu ý rằng tất cả code snippets trong các phần sau đều được triển khai bên trong class ContentDO.**

## **Core version control functions**

Sau đây là các thao tác kiểm soát phiên bản cơ bản:

```yaml
  // Khởi tạo hoặc lấy trạng thái hiện có từ Durable Objects storage
  private async initialize(): Promise<ContentState> {
    // Lấy trạng thái hiện có từ Durable Objects storage
    const stored = await this.state.storage.get<ContentState>("content");
    if (!stored) {
      // Khởi tạo trạng thái nếu chưa tồn tại
      const initialData: ContentState = {
        currentVersion: 0,
        versions: [],
        tags: {},
        content: null,
        publishHistory: []
      };
      await this.state.storage.put("content", initialData);
      return initialData;
    }
    return stored;
  }

  // Tính toán ID phiên bản tiếp theo dựa trên các phiên bản hiện có
  private getNextVersionId(data: ContentState): number {
    return data.versions.length > 0 
      ? Math.max(...data.versions.map(v => v.id)) + 1 
      : 1;
  }

  // Tạo một phiên bản mới với nội dung và thông báo được cung cấp
  async createVersion(content: string, message: string = ""): Promise<Version> {
    const data = await this.initialize();
      
    const newVersion: Version = {
      id: this.getNextVersionId(data),
      content,
      timestamp: new Date().toISOString(),
      message,
      status: VersionStatus.DRAFT,
      diff: data.content ? this.calculateDetailedDiff(data.content, content) : undefined
    };
  
    data.versions.push(newVersion);
    data.currentVersion = newVersion.id;
    data.content = content;
  
    await this.state.storage.put("content", data);
    return newVersion;
  }

  // Lấy tất cả các phiên bản
  async getVersions(): Promise<Version[]> {
    const data = await this.initialize();
    return data.versions;
  }

  // Lấy một phiên bản cụ thể theo ID
  async getVersion(id: number): Promise<Version | null> {
    const data = await this.initialize();
    return data.versions.find(v => v.id === id) || null;
  }

  // Xóa một phiên bản cụ thể theo ID
  async deleteVersion(id: number): Promise<{ success: boolean }> {
    const data = await this.initialize();
      
    const versionIndex = data.versions.findIndex(v => v.id === id);
    if (versionIndex === -1) {
      throw new Error("Version not found");
    }
  
    const version = data.versions[versionIndex];
    if (version.status === VersionStatus.PUBLISHED) {
      throw new Error("Cannot delete published version");
    }
  
    data.versions.splice(versionIndex, 1);
  
    if (data.currentVersion === id) {
      data.currentVersion = 0;
      data.content = null;
    }
  
    // Xóa bất kỳ tag nào được liên kết với phiên bản này
    for (const tagName in data.tags) {
      if (data.tags[tagName].versionId === id) {
        delete data.tags[tagName];
      }
    }
  
    await this.state.storage.put("content", data);
    return {
      success: true
    };
  }

  // TODO: Tag Management Functions
```

Các hàm này cung cấp:

- Khởi tạo và quản lý trạng thái với type safety
- Tạo phiên bản với tự động tạo ID
- Truy xuất phiên bản với phản hồi an toàn kiểu
- Xóa phiên bản với xử lý lỗi và dọn dẹp thích hợp
- Tự động lưu trạng thái bằng Durable Objects storage

## Tag management functions

```yaml
  async getTags(): Promise<Tag[]> {
    const data = await this.initialize();
    return Object.entries(data.tags).map(([_, tag]) => ({
      ...tag
    }));
  }

  async getVersionTags(versionId: number): Promise<Tag[]> {
    const data = await this.initialize();
    return Object.entries(data.tags)
      .filter(([_, tag]) => tag.versionId === versionId)
      .map(([_, tag]) => ({
        ...tag
      }));
  }
  
  async createTag(versionId: number, name: string): Promise<Tag> {
    const data = await this.initialize();
    if (data.tags[name]) {
      throw new Error("Tag already exists");
    }
  
    const version = data.versions.find(v => v.id === versionId);
    if (!version) {
      throw new Error("Version not found");
    }
  
    const tag: Tag = {
      name,
      versionId,
      createdAt: new Date().toISOString()
    };
    data.tags[name] = tag;
    await this.state.storage.put("content", data);
    return tag;
  }
  
  async updateTag(name: string, newName: string): Promise<Tag> {
    const data = await this.initialize();
    
    if (!data.tags[name]) {
      throw new Error("Tag not found");
    }
  
    if (data.tags[newName]) {
      throw new Error("New tag name already exists");
    }
  
    const tag = data.tags[name];
    delete data.tags[name];
    data.tags[newName] = {
      ...tag,
      name: newName
    };
  
    await this.state.storage.put("content", data);
    return {
      ...data.tags[newName],
      name: newName
    };
  }
  
  async deleteTag(name: string): Promise<{ success: boolean; message: string }> {
    const data = await this.initialize();
    
    if (!data.tags[name]) {
      throw new Error("Tag not found");
    }
  
    delete data.tags[name];
    await this.state.storage.put("content", data);
  
    return {
      success: true,
      message: `Tag ${name} deleted successfully`
    };
  }

  // TODO: Publishing and unpublishing functions
```

Quản lý tag bao gồm:

- Tạo và liệt kê tag
- Cập nhật tên tag
- Xóa tag
- Truy xuất tag cho các phiên bản cụ thể

## Publishing and unpublishing functions

```yaml
  async publishVersion(versionId: number, publishedBy: string): Promise<PublishRecord> {
    const data = await this.initialize();
    const version = data.versions.find(v => v.id === versionId);
    if (!version) {
      throw new Error("Version not found");
    }
  
    data.versions = data.versions.map(v => ({
      ...v,
      status: v.id === versionId ? VersionStatus.PUBLISHED : VersionStatus.DRAFT
    }));
  
    const publishRecord: PublishRecord = {
      versionId,
      publishedAt: new Date().toISOString(),
      publishedBy
    };
  
    if (!data.publishHistory) {
      data.publishHistory = [];
    }
    data.publishHistory.push(publishRecord);
  
    data.currentVersion = versionId;
    data.content = version.content;
  
    await this.state.storage.put("content", data);
    return publishRecord;
  }
  
  // Unpublish a version
  async unpublishVersion(versionId: number): Promise<Version> {
    const data = await this.initialize();
    const version = data.versions.find(v => v.id === versionId);
    if (!version) {
      throw new Error("Version not found");
    }
    data.versions = data.versions.map(v => ({
      ...v,
      status: v.id === versionId ? VersionStatus.DRAFT : v.status
    }));
    if (data.publishHistory) {
      data.publishHistory = data.publishHistory.filter(
        record => record.versionId !== versionId
      );
    }
    if (data.currentVersion === versionId) {
      data.currentVersion = 0;
      data.content = null;
    }
    await this.state.storage.put("content", data);
    const updatedVersion = data.versions.find(v => v.id === versionId);
    if (!updatedVersion) {
      throw new Error("Failed to get updated version");
    }
    return updatedVersion;
  }
  
  async getPublishHistory(): Promise<PublishRecord[]> {
    const data = await this.initialize();
    return data.publishHistory || [];
  }
  // TODO: Diff and version control operations
```

Các hàm này xử lý:

- Publish phiên bản với audit trail
- Unpublish phiên bản
- Theo dõi lịch sử xuất bản
- Quản lý trạng thái

## Diff and version control operations

```yaml
  async compareVersions(fromId: number, toId: number): Promise<ContentDiff> {
    const data = await this.initialize();
    const fromVersion = data.versions.find(v => v.id === fromId);
    const toVersion = data.versions.find(v => v.id === toId);
    if (!fromVersion || !toVersion) {
      throw new Error("Version not found");
    }
    return this.calculateDetailedDiff(fromVersion.content, toVersion.content);
  }
  
  private calculateDetailedDiff(oldContent: string, newContent: string): ContentDiff {
    const patch = createPatch('content', 
      oldContent,
      newContent,
      'old version',
      'new version'
    );
  
    const oldLines = oldContent.split('\n');
    const newLines = newContent.split('\n');
  
    return {
      from: oldContent,
      to: newContent,
      changes: {
        additions: newLines.length - oldLines.length,
        deletions: Math.max(0, oldLines.length - newLines.length),
        totalChanges: Math.abs(newLines.length - oldLines.length),
        timestamp: new Date().toISOString()
      },
      patch: patch,
      hunks: []
    };
  }
  
  async getDiff(fromVersionId: number, toVersionId: number): Promise<Response> {
    const data = await this.initialize();
    const fromVersion = data.versions.find(v => v.id === fromVersionId);
    const toVersion = data.versions.find(v => v.id === toVersionId);
    
    if (!fromVersion || !toVersion) {
      throw new Error("Version not found");
    }
  
    const formattedDiff = [
      `Comparing Version ${fromVersion.id} -> Version ${toVersion.id}`,
      ```typescript
      fromVersion.content,
      '\nContent in Version ' + toVersion.id + ':',
      toVersion.content,
      '\nDifferences:',
      '===================================================================',
      createPatch('content.txt',
        fromVersion.content || '',
        toVersion.content || '',
        `Version ${fromVersion.id} (${fromVersion.message})`,
        `Version ${toVersion.id} (${toVersion.message})`
      )
    ].join('\n');
  
    return new Response(formattedDiff, {
      headers: {
        'Content-Type': 'text/plain',
        'Access-Control-Allow-Origin': '*',
      }
    });
  }
  
  async revertTo(versionId: number): Promise<Version> {
    const data = await this.initialize();
    const targetVersion = data.versions.find(v => v.id === versionId);
    if (!targetVersion) {
      throw new Error("Version not found");
    }
  
    const newVersion: Version = {
      id: this.getNextVersionId(data),
      content: targetVersion.content,
      timestamp: new Date().toISOString(),
      message: `Reverted to version ${versionId}`,
      status: targetVersion.status,
      diff: this.calculateDetailedDiff(
        data.versions[data.versions.length - 1]?.content || '',
        targetVersion.content
      )
    };
  
    data.versions.push(newVersion);
    await this.state.storage.put("content", data);
    return newVersion;
  }

  // TODO: Request handling and routing
```

Cung cấp:

- Chức năng so sánh phiên bản
- Tính toán diff chi tiết
- Khả năng revert phiên bản
- Tạo patch cho các thay đổi

## Request handling and routing

Thêm các phương thức xử lý yêu cầu vào src/contentDO.ts:

```yaml
  // Entry point for requests to the Durable Object
  async fetch(request: Request): Promise<Response> {
      if (request.method === 'OPTIONS') {
        return new Response(null);
      }
    
      try {
        const url = new URL(request.url);
        const parts = url.pathname.split('/').filter(Boolean);
    
        console.log('ContentDO handling request:', request.method, url.pathname);
        console.log('Parts:', parts);
        // Add detailed logging
        console.log('Request details:');
        console.log('- Method:', request.method);
        console.log('- URL:', request.url);
        console.log('- Path:', url.pathname);
        console.log('- Parts:', parts);
        console.log('- Headers:', Object.fromEntries(request.headers));

        // Log the body if it exists
        if (request.body) {
          const clonedRequest = request.clone();
          const body = await clonedRequest.text();
          console.log('- Body:', body);
        }
    
        const response = await this.handleRequest(request, parts);
        return response;

      } catch (err) {
        const error = err as Error;
        console.error('Error:', error);
        return new Response(error.message, { 
          status: 500
        });
      }
    }
    
    private async handleRequest(request: Request, parts: string[]): Promise<Response> {
      const path = parts.join('/');
      console.log('Handling request:');
      console.log('- Method + Path:', `${request.method} ${path}`);
      console.log('- Looking for match:', `POST content/default/versions/${parts[3]}/publish`);
    
      switch (`${request.method} ${path}`) {
        
        case 'POST content': {
          const body = await request.json() as CreateVersionRequest;
          const version = await this.createVersion(body.content, body.message);
          return Response.json(version);
      }
    
        case 'GET content/default': {
          const data = await this.initialize();
          if (!data.currentVersion) {
            return Response.json(null);
          }
          const version = await this.getVersion(data.currentVersion);
          return Response.json(version);
        }
    
        case 'GET content/default/versions': {
          const versions = await this.getVersions();
          return Response.json(versions);
        }
    
        case `GET content/default/versions/${parts[3]}`: {
          const version = await this.getVersion(parseInt(parts[3]));
          return Response.json(version);
        }
    
        case `DELETE content/default/versions/${parts[3]}`: {
          const versionId = parseInt(parts[3]);
          const result = await this.deleteVersion(versionId);
          return Response.json(result);
        }
    
        case 'GET content/versions/tags': {
          const tags = await this.getTags();
          return Response.json(tags);
        }
    
        case `GET content/versions/${parts[2]}/tags`: {
          const versionId = parseInt(parts[2]);
          const tags = await this.getVersionTags(versionId);
          return Response.json(tags);
        }
    
        case 'POST content/versions/tags': {
          const { versionId, name } = await request.json() as CreateTagRequest;
          const tag = await this.createTag(versionId, name);
          return Response.json(tag);
        }
    
        case `PUT content/versions/tags/${parts[3]}`: {
          const body = await request.json() as UpdateTagRequest;
          const tag = await this.updateTag(parts[3], body.newName);
          return Response.json(tag);
        }
    
        case `DELETE content/versions/tags/${parts[3]}`: {
          const result = await this.deleteTag(parts[3]);
          return Response.json(result);
        }
    
        case `POST content/default/versions/${parts[3]}/publish`: {
          try {
            console.log('Attempting to publish version');
            console.log('Raw request body:', await request.clone().text());
            const { publishedBy } = await request.json() as PublishVersionRequest;
            console.log('Parsed publishedBy:', publishedBy);
            const result = await this.publishVersion(parseInt(parts[3]), publishedBy);
            return Response.json(result);
          } catch (err) {
            const error = err as Error;
            console.error('Error in publish handler:', error);
            return new Response(error.message, { status: 500 });
          }
        }
    
        case `POST content/default/versions/${parts[3]}/unpublish`: {
          const result = await this.unpublishVersion(parseInt(parts[3]));
          return Response.json(result);
        }
    
        case 'GET content/default/publish-history': {
          const history = await this.getPublishHistory();
          return Response.json(history);
        }
    
        case `GET content/default/versions/${parts[3]}/diff`: {
          const compareToId = parseInt(new URL(request.url).searchParams.get('compare') || '0');
          if (compareToId) {
            return await this.getDiff(parseInt(parts[3]), compareToId);
          }
          const diff = await this.compareVersions(parseInt(parts[3]), parseInt(parts[3]) - 1);
          return Response.json(diff);
        }
    
        case `POST content/default/revert`: {
          const body = await request.json() as RevertVersionRequest;
          const version = await this.revertTo(body.versionId);
          return Response.json(version);
        }
    
        default:
          return new Response('No route matched: ' + request.method + ' ' + path, { status: 404 });
      }
    }
```

Xử lý yêu cầu và định tuyến tạo thành cốt lõi của hệ thống, quản lý tất cả các API endpoints. Hãy chia nhỏ các thành phần chính:

1. **Main Request Handler (`fetch`)**
    - Hoạt động như điểm vào chính cho tất cả các yêu cầu đến Durable Object
    - Xử lý các yêu cầu CORS preflight
    - Phân tích cú pháp URL và định tuyến các yêu cầu đến các trình xử lý thích hợp
    - Đảm bảo tất cả các phản hồi bao gồm tiêu đề CORS
    - Cung cấp xử lý lỗi tập trung
2. **Route Handler (`handleRequest`)**
Quản lý các endpoints chính sau:
    
    **Version Management:**
    
    - `POST /content` - Tạo phiên bản mới với nội dung và thông báo
    - `GET /content/default` - Truy xuất phiên bản hiện tại
    - `GET /content/default/versions` - Liệt kê tất cả các phiên bản
    - `GET /content/default/versions/{id}` - Lấy phiên bản cụ thể theo ID
    - `DELETE /content/default/versions/{id}` - Xóa phiên bản (không thể xóa phiên bản đã published)
    
    **Tag Operations:**
    
    - `GET /content/versions/tags` - Liệt kê tất cả các tag
    - `GET /content/versions/{id}/tags` - Lấy tag cho phiên bản cụ thể
    - `POST /content/versions/tags` - Tạo tag mới
    - `PUT /content/versions/tags/{name}` - Cập nhật tên tag
    - `DELETE /content/versions/tags/{name}` - Xóa tag
    
    **Publishing Operations:**
    
    - `POST /content/default/versions/{id}/publish` - Publish phiên bản với audit trail
    - `POST /content/default/versions/{id}/unpublish` - Unpublish phiên bản
    - `GET /content/default/publish-history` - Xem toàn bộ lịch sử xuất bản
    
    **Version Control Operations:**
    
    - `GET /content/default/versions/{id}/diff` - So sánh các phiên bản với diff chi tiết
    - `POST /content/default/revert` - Revert về phiên bản cụ thể
3. **Error Handling**
    - Mỗi route được bao bọc trong các khối try-catch
    - Trả về thông báo lỗi có ý nghĩa với mã trạng thái thích hợp
    - Ghi lại lỗi cho mục đích debug
4. **Response Format**
    - Tất cả các phản hồi tuân theo một định dạng nhất quán
    - Tự động bao gồm tiêu đề CORS
    - Phản hồi JSON cho API endpoints
    - Phản hồi văn bản thuần túy cho các hoạt động diff

# **5. Testing & Deployment**

Nhấp vào liên kết được hiển thị trong terminal của bạn: `http://localhost:8787` hoặc sao chép và dán URL vào trình duyệt của bạn. Bạn cũng có thể sử dụng các lệnh cURL để test các API endpoints.

Khi bạn truy cập URL lần đầu tiên, bạn sẽ thấy một thông báo cho biết "No published content available" vì chúng ta chưa publish bất kỳ phiên bản nào.

Hãy tạo và publish một phiên bản để test:

```
# Create a new version
curl -X POST \\
  -H "Content-Type: application/json" \\
  -d '{"content": "Hello World!", "message": "First version"}' \\
  <http://localhost:8787/content>

# Publish a version
curl -X POST \\
  -H "Content-Type: application/json" \\
  -d '{"publishedBy": "test-user"}' \\
  <http://localhost:8787/content/default/versions/1/publish>

# List all versions
curl -X GET \\
  <http://localhost:8787/content/default/versions>

```

Bây giờ hãy refresh trình duyệt của bạn tại `http://localhost:8787` để xem nội dung đã published của bạn.

Khi bạn hài lòng với kết quả, hãy deploy dự án của bạn:

```
npx wrangler deploy
```

Một số lệnh để test project sau khi deploy:

## Version Management Operations:

```
# Create a new version
curl -X POST \\
  -H "Content-Type: application/json" \\
  -d '{"content": "Version 1 content", "message": "First version"}' \\
  https://<your-worker-url>/content

# Get current version
curl -X GET \\
  https://<your-worker-url>/content/default

# List all versions
curl -X GET \\
  https://<your-worker-url>/content/default/versions

# Get specific version
curl -X GET \\
  https://<your-worker-url>/content/default/versions/{id}

# Compare versions
curl -X GET \\
  https://<your-worker-url>/content/default/versions/{id1}/diff?compare={id2}

```

## Tag operations:

```
# Create a new tag
curl -X POST \\
  -H "Content-Type: application/json" \\
  -d '{"versionId": 1, "name": "v1.0"}' \\
  https://<your-worker-url>/content/versions/tags

# List all tags
curl -X GET \\
  https://<your-worker-url>/content/versions/tags

# Update tag name
curl -X PUT \\
  -H "Content-Type: application/json" \\
  -d '{"newName": "stable"}' \\
  https://<your-worker-url>/content/versions/tags/v1.0

```

## Publish và unpublish operations:

```
# Publish a version
curl -X POST \\
  -H "Content-Type: application/json" \\
  -d '{"publishedBy": "test-user"}' \\
  https://<your-worker-url>/content/default/versions/{id}/publish

# View publication history
curl -X GET \\
  https://<your-worker-url>/content/default/publish-history
  
# Unpublish a version
# Thay thế {id} bằng id thực tế của bạn.
curl -X POST \
  https://<your-worker-url>/content/default/versions/{id}/unpublish

```

## 6. Additional resources

Nếu bạn muốn implement CI/CD cho nền tảng Worker, bạn có thể xem blog này: [Continuous Deployment for Cloudflare Workers with GitHub Actions](https://blog.cloudflare.com/workers-builds-integrated-ci-cd-built-on-the-workers-platform/)

Bạn có thể tìm thấy:

- Mã nguồn hoàn chỉnh cho dự án này trên [GitHub](https://github.com/shinchan79/content-version-system.git)
- Code Frontend: https://github.com/shinchan79/version-control-frontend-copy
