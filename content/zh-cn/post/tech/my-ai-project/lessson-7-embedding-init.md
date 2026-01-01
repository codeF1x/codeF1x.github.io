# 第七课_Next.js 本地 Embedding 初始化

# Next.js 本地 Embedding 初始化开发文档

本手册记录了在 Next.js 环境下，利用 `transformers.js` 实现纯浏览器端文本向量化（Embedding）的完整实战过程。

## 1. 核心架构设计

为了保证 UI 的流畅性并优化国内下载体验，我们采用了以下架构：

- **计算层 (Web Worker)**：将 CPU 密集型的模型推理任务移至后台线程。
- **逻辑层 (Custom Hook)**：封装 Worker 通信逻辑，提供响应式的进度状态和 Promise 化的调用接口。
- **存储层 (Browser Cache)**：利用浏览器 Cache API 实现模型的持久化存储。
- **加速层 (Mirror Source)**：配置国内镜像源，解决模型下载缓慢问题。

---

## 2. 核心代码实现

### 2.1 后台计算：`embedding-worker.ts`

负责模型的加载、缓存配置及向量计算。

```tsx
import { pipeline, env } from "@huggingface/transformers";

// 核心配置：禁用本地模型检查，启用浏览器 cache，设置国内镜像
env.allowLocalModels = false;
env.useBrowserCache = true;
env.remoteHost = "https://hf-mirror.com";

class EmbeddingPipeline {
  static task = "feature-extraction";
  static model = "Xenova/all-MiniLM-L6-v2"; // 轻量级模型，适合浏览器端
  static instance: any = null;

  static async getInstance(progress_callback?: (progress: any) => void) {
    if (this.instance === null) {
      this.instance = pipeline(this.task as any, this.model, {
        progress_callback,
      });
    }
    return this.instance;
  }
}

// 监听主线程指令
self.addEventListener("message", async (event) => {
  const { text, task } = event.data;

  if (task === "init") {
    await EmbeddingPipeline.getInstance((x) =>
      self.postMessage({ status: "progress", ...x })
    );
    self.postMessage({ status: "ready" });
  } else if (task === "embed") {
    const extractor = await EmbeddingPipeline.getInstance();
    const output = await extractor(text, { pooling: "mean", normalize: true });
    self.postMessage({
      status: "complete",
      embedding: Array.from(output.data),
    });
  }
});
```

### 2.2 逻辑封装：`use-embedding.ts`

通过 React Hook 管理 Worker 生命周期，并将事件驱动转换为 Promise 驱动。

```tsx
export function useEmbedding() {
  const [ready, setReady] = useState(false);
  const [progress, setProgress] = useState<EmbeddingProgress[]>([]);
  const worker = useRef<Worker | null>(null);

  useEffect(() => {
    // 初始化 Worker
    worker.current = new Worker(
      new URL("../lib/embedding-worker.ts", import.meta.url),
      { type: "module" }
    );

    // 处理进度和就绪状态
    worker.current.onmessage = (e) => {
      const { status, ...rest } = e.data;
      if (status === "progress") {
        setProgress((prev) => {
          const existing = prev.findIndex((p) => p.file === rest.file);
          return existing !== -1
            ? ((prev[existing] = rest), [...prev])
            : [...prev, rest];
        });
      } else if (status === "ready") setReady(true);
    };

    worker.current.postMessage({ task: "init" });
    return () => worker.current?.terminate();
  }, []);

  // 封装向量计算为 Promise
  const embed = useCallback(
    async (text: string) => {
      return new Promise((resolve) => {
        const listener = (e: MessageEvent) => {
          if (e.data.status === "complete") {
            worker.current?.removeEventListener("message", listener);
            resolve(e.data.embedding);
          }
        };
        worker.current?.addEventListener("message", listener);
        worker.current?.postMessage({ task: "embed", text });
      });
    },
    [ready]
  );

  return { ready, progress, embed };
}
```

---

## 3. 关键优化点 (Expert Tips)

| 优化项 | 实现方式 | 收益 |
| --- | --- | --- |
| **UI 不卡顿** | Web Worker 独立线程 | 即使在进行复杂的矩阵运算时，主线程依然能保持 60fps。 |
| **秒级加载** | `env.useBrowserCache` | 首次下载后，模型存储在 IndexedDB/Cache 中，后续访问无需网络。 |
| **国内加速** | `hf-mirror.com` | 下载速度从几十 KB/s 提升至数 MB/s。 |
| **类型安全** | TypeScript Interface | 明确 `EmbeddingProgress` 结构，减少开发阶段的低级错误。 |

---

## 4. 验证清单

- [x]  **首次加载**：控制台 Network 面板显示从镜像站下载 `.onnx` 文件。
- [x]  **进度反馈**：UI 实时显示每个分块文件的下载百分比。
- [x]  **二次加载**：刷新页面后，Worker 瞬间返回 `ready` 状态。
- [x]  **计算准确**：输入文本后，返回长度为 384 的浮点数数组（MiniLM-L6 维度）。

---

**文档版本**：v1.0

**核心库**：`@huggingface/transformers`