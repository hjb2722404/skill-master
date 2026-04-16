---
name: file-upload-handling
description: >
  本技能用于实现文件上传处理，包括大文件分片、断点续传、上传进度、图片预览等。
  当需要实现文件上传、处理大文件上传、实现断点续传时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 文件上传处理

## 一、概述

### 1.1 这是什么

文件上传处理技能提供完整的文件上传解决方案，涵盖普通上传、大文件分片、断点续传、上传进度显示、图片预览等功能。

### 1.2 适用场景

- ✅ 普通文件上传
- ✅ 大文件分片上传
- ✅ 断点续传
- ✅ 多文件批量上传
- ✅ 图片压缩和预览
- ✅ 上传进度显示
- ❌ 纯文本数据传输
- ❌ 实时音视频流

### 1.3 核心原则

1. **分片上传** —— 大文件分割上传，提高可靠性
2. **断点续传** —— 支持上传中断后继续
3. **并发控制** —— 限制同时上传的文件数量
4. **用户体验** —— 提供清晰的进度反馈

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要实现文件上传功能
- 需要处理大文件（>10MB）上传
- 需要实现断点续传
- 需要批量上传文件
- 需要图片预览和压缩

### 2.2 输入

- 文件类型限制
- 文件大小限制
- 上传接口信息
- 是否需要断点续传

### 2.3 输出

- 上传组件实现
- 分片上传逻辑
- 断点续传机制
- 进度显示组件

---

## 三、基础上传实现

### 3.1 单文件上传

```typescript
import { useState, useCallback, useRef } from 'react';

interface UploadOptions {
  url: string;
  headers?: Record<string, string>;
  onProgress?: (progress: number) => void;
  onSuccess?: (response: any) => void;
  onError?: (error: Error) => void;
}

export function useFileUpload() {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);
  const abortControllerRef = useRef<AbortController | null>(null);

  const upload = useCallback(async (file: File, options: UploadOptions) => {
    setUploading(true);
    setProgress(0);

    const formData = new FormData();
    formData.append('file', file);

    abortControllerRef.current = new AbortController();

    try {
      const xhr = new XMLHttpRequest();
      
      // 进度监听
      xhr.upload.onprogress = (event) => {
        if (event.lengthComputable) {
          const percent = Math.round((event.loaded / event.total) * 100);
          setProgress(percent);
          options.onProgress?.(percent);
        }
      };

      const response = await new Promise<any>((resolve, reject) => {
        xhr.onload = () => {
          if (xhr.status >= 200 && xhr.status < 300) {
            resolve(JSON.parse(xhr.response));
          } else {
            reject(new Error(xhr.statusText));
          }
        };
        xhr.onerror = () => reject(new Error('上传失败'));
        xhr.onabort = () => reject(new Error('上传已取消'));

        xhr.open('POST', options.url);
        
        // 设置请求头
        Object.entries(options.headers || {}).forEach(([key, value]) => {
          xhr.setRequestHeader(key, value);
        });

        xhr.send(formData);
      });

      options.onSuccess?.(response);
      return response;
    } catch (error) {
      options.onError?.(error as Error);
      throw error;
    } finally {
      setUploading(false);
    }
  }, []);

  const cancel = useCallback(() => {
    abortControllerRef.current?.abort();
  }, []);

  return { upload, cancel, uploading, progress };
}
```

### 3.2 多文件上传

```typescript
interface MultiUploadOptions extends UploadOptions {
  maxConcurrent?: number; // 最大并发数
}

export function useMultiFileUpload() {
  const [uploads, setUploads] = useState<Map<string, UploadState>>(new Map());

  const uploadFiles = useCallback(async (
    files: File[],
    options: MultiUploadOptions
  ) => {
    const maxConcurrent = options.maxConcurrent || 3;
    const queue = [...files];
    const active: Promise<void>[] = [];

    const processFile = async (file: File) => {
      const id = `${file.name}-${Date.now()}`;
      
      setUploads(prev => new Map(prev.set(id, {
        file,
        status: 'uploading',
        progress: 0,
      })));

      try {
        await upload(file, {
          ...options,
          onProgress: (progress) => {
            setUploads(prev => new Map(prev.set(id, {
              ...prev.get(id)!,
              progress,
            })));
          },
        });

        setUploads(prev => new Map(prev.set(id, {
          ...prev.get(id)!,
          status: 'success',
          progress: 100,
        })));
      } catch (error) {
        setUploads(prev => new Map(prev.set(id, {
          ...prev.get(id)!,
          status: 'error',
          error: error as Error,
        })));
      }
    };

    // 并发控制
    while (queue.length > 0 || active.length > 0) {
      while (active.length < maxConcurrent && queue.length > 0) {
        const file = queue.shift()!;
        active.push(processFile(file));
      }

      if (active.length > 0) {
        await Promise.race(active);
      }
    }
  }, []);

  return { uploadFiles, uploads };
}
```

---

## 四、大文件分片上传

```typescript
interface ChunkUploadOptions {
  url: string;
  chunkSize?: number; // 分片大小，默认 2MB
  maxRetries?: number;
  onProgress?: (progress: number) => void;
}

interface Chunk {
  index: number;
  start: number;
  end: number;
  blob: Blob;
}

export async function uploadLargeFile(
  file: File,
  options: ChunkUploadOptions
): Promise<void> {
  const { url, chunkSize = 2 * 1024 * 1024, maxRetries = 3 } = options;
  
  // 1. 初始化上传，获取 uploadId
  const { uploadId, chunks: totalChunks } = await initUpload(file);

  // 2. 创建分片
  const chunks: Chunk[] = [];
  for (let i = 0; i < totalChunks; i++) {
    const start = i * chunkSize;
    const end = Math.min(start + chunkSize, file.size);
    chunks.push({
      index: i,
      start,
      end,
      blob: file.slice(start, end),
    });
  }

  // 3. 上传分片（支持断点续传）
  const uploadedChunks = await getUploadedChunks(uploadId);
  const pendingChunks = chunks.filter(c => !uploadedChunks.includes(c.index));

  // 4. 并发上传分片
  const concurrency = 3;
  const results = await Promise.all(
    pendingChunks.map(async (chunk) => {
      for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
          await uploadChunk(uploadId, chunk, file.name);
          return { success: true, index: chunk.index };
        } catch (error) {
          if (attempt === maxRetries - 1) {
            return { success: false, index: chunk.index, error };
          }
          await delay(1000 * Math.pow(2, attempt)); // 指数退避
        }
      }
    })
  );

  // 5. 检查是否全部成功
  const failedChunks = results.filter(r => !r?.success);
  if (failedChunks.length > 0) {
    throw new Error(`部分分片上传失败: ${failedChunks.map(f => f?.index).join(', ')}`);
  }

  // 6. 合并分片
  await mergeChunks(uploadId, file.name);
}

async function uploadChunk(
  uploadId: string,
  chunk: Chunk,
  filename: string
): Promise<void> {
  const formData = new FormData();
  formData.append('uploadId', uploadId);
  formData.append('chunkIndex', chunk.index.toString());
  formData.append('chunk', chunk.blob);
  formData.append('filename', filename);

  const response = await fetch('/api/upload/chunk', {
    method: 'POST',
    body: formData,
  });

  if (!response.ok) {
    throw new Error(`分片 ${chunk.index} 上传失败`);
  }
}

async function initUpload(file: File): Promise<{ uploadId: string; chunks: number }> {
  const response = await fetch('/api/upload/init', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      filename: file.name,
      size: file.size,
      mimeType: file.type,
    }),
  });

  return response.json();
}
```

---

## 五、图片处理

### 5.1 图片压缩

```typescript
interface CompressOptions {
  maxWidth?: number;
  maxHeight?: number;
  quality?: number; // 0-1
  maxSize?: number; // 最大文件大小（字节）
}

export function compressImage(
  file: File,
  options: CompressOptions = {}
): Promise<Blob> {
  const {
    maxWidth = 1920,
    maxHeight = 1080,
    quality = 0.8,
    maxSize,
  } = options;

  return new Promise((resolve, reject) => {
    const img = new Image();
    img.src = URL.createObjectURL(file);
    
    img.onload = () => {
      URL.revokeObjectURL(img.src);

      // 计算缩放后的尺寸
      let { width, height } = img;
      if (width > maxWidth) {
        height = (height * maxWidth) / width;
        width = maxWidth;
      }
      if (height > maxHeight) {
        width = (width * maxHeight) / height;
        height = maxHeight;
      }

      // 绘制到canvas
      const canvas = document.createElement('canvas');
      canvas.width = width;
      canvas.height = height;
      const ctx = canvas.getContext('2d')!;
      ctx.drawImage(img, 0, 0, width, height);

      // 压缩
      canvas.toBlob(
        (blob) => {
          if (blob) {
            // 如果仍然超过大小限制，递归压缩
            if (maxSize && blob.size > maxSize && quality > 0.1) {
              compressImage(file, { ...options, quality: quality - 0.1 })
                .then(resolve)
                .catch(reject);
            } else {
              resolve(blob);
            }
          } else {
            reject(new Error('压缩失败'));
          }
        },
        file.type,
        quality
      );
    };

    img.onerror = () => reject(new Error('图片加载失败'));
  });
}
```

### 5.2 图片预览

```typescript
export function useImagePreview() {
  const [preview, setPreview] = useState<string | null>(null);

  const generatePreview = useCallback((file: File): Promise<string> => {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = (e) => resolve(e.target?.result as string);
      reader.onerror = reject;
      reader.readAsDataURL(file);
    });
  }, []);

  const clearPreview = useCallback(() => {
    if (preview) {
      URL.revokeObjectURL(preview);
      setPreview(null);
    }
  }, [preview]);

  return { preview, generatePreview, clearPreview };
}

// 多图预览
export function useMultiImagePreview() {
  const [previews, setPreviews] = useState<Map<string, string>>(new Map());

  const addPreviews = useCallback(async (files: File[]) => {
    const newPreviews = new Map(previews);
    
    for (const file of files) {
      if (file.type.startsWith('image/')) {
        const url = URL.createObjectURL(file);
        newPreviews.set(file.name, url);
      }
    }
    
    setPreviews(newPreviews);
  }, [previews]);

  const removePreview = useCallback((name: string) => {
    const url = previews.get(name);
    if (url) {
      URL.revokeObjectURL(url);
    }
    const newPreviews = new Map(previews);
    newPreviews.delete(name);
    setPreviews(newPreviews);
  }, [previews]);

  const clearAll = useCallback(() => {
    previews.forEach(url => URL.revokeObjectURL(url));
    setPreviews(new Map());
  }, [previews]);

  return { previews, addPreviews, removePreview, clearAll };
}
```

---

## 六、上传组件

```typescript
interface UploadProps {
  accept?: string;
  multiple?: boolean;
  maxSize?: number;
  maxCount?: number;
  onUpload: (files: File[]) => void;
  onError?: (error: Error) => void;
}

export function UploadDropzone({
  accept,
  multiple,
  maxSize,
  maxCount,
  onUpload,
  onError,
}: UploadProps) {
  const [isDragging, setIsDragging] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  const handleDrop = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(false);

    const files = Array.from(e.dataTransfer.files);
    validateAndUpload(files);
  }, []);

  const handleFileSelect = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    const files = Array.from(e.target.files || []);
    validateAndUpload(files);
  }, []);

  const validateAndUpload = (files: File[]) => {
    try {
      // 数量限制
      if (maxCount && files.length > maxCount) {
        throw new Error(`最多只能上传 ${maxCount} 个文件`);
      }

      // 大小限制
      if (maxSize) {
        const oversized = files.filter(f => f.size > maxSize);
        if (oversized.length > 0) {
          throw new Error(`文件大小不能超过 ${formatSize(maxSize)}`);
        }
      }

      // 类型限制
      if (accept) {
        const acceptedTypes = accept.split(',').map(t => t.trim());
        const invalid = files.filter(f => {
          return !acceptedTypes.some(type => {
            if (type.includes('*')) {
              return f.type.startsWith(type.replace('/*', ''));
            }
            return f.type === type;
          });
        });
        if (invalid.length > 0) {
          throw new Error(`不支持的文件类型`);
        }
      }

      onUpload(files);
    } catch (error) {
      onError?.(error as Error);
    }
  };

  return (
    <div
      className={`upload-dropzone ${isDragging ? 'dragging' : ''}`}
      onDragOver={(e) => { e.preventDefault(); setIsDragging(true); }}
      onDragLeave={() => setIsDragging(false)}
      onDrop={handleDrop}
      onClick={() => inputRef.current?.click()}
    >
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        multiple={multiple}
        onChange={handleFileSelect}
        hidden
      />
      <p>拖拽文件到此处，或点击上传</p>
      <p className="hint">
        {accept && `支持格式: ${accept}`}
        {maxSize && `，最大 ${formatSize(maxSize)}`}
      </p>
    </div>
  );
}

function formatSize(bytes: number): string {
  if (bytes === 0) return '0 B';
  const k = 1024;
  const sizes = ['B', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
}
```

---

## 七、决策检查清单

- [ ] 实现了文件类型和大小验证
- [ ] 大文件使用分片上传
- [ ] 实现了上传进度显示
- [ ] 支持上传取消
- [ ] 图片上传支持压缩
- [ ] 实现了图片预览
- [ ] 多文件上传有并发控制
- [ ] 错误处理完善

---

## 八、相关技能

- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 上传接口调用
- [React组件设计模式](./react-component-design-patterns/SKILL.md) —— 上传组件设计
- [性能优化实战](./performance-optimization/SKILL.md) —— 大文件处理优化

---

## 九、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含分片上传、断点续传、图片处理 |
