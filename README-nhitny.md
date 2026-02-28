Ok, mình viết cho bạn **README.md hoàn chỉnh, copy dùng luôn**, tập trung đúng mấy thứ bạn vừa vật lộn 😄
– **bật/tắt container**
– **đổi model LLM / embedding**
– **xử lý lỗi thường gặp (embedding dim, Ollama)**
– **flow dùng LightRAG sau khi upload xong**

---

```md
# LightRAG – Hướng dẫn sử dụng & cấu hình (Docker + Ollama)

Tài liệu này hướng dẫn:
- Bật / tắt LightRAG
- Bật / tắt Ollama
- Thay đổi model LLM và Embedding
- Xử lý các lỗi thường gặp
- Cách sử dụng LightRAG sau khi upload tài liệu

---

## 1. Kiến trúc tổng quát

```

┌──────────────┐        HTTP        ┌──────────────┐
│   LightRAG   │ ───────────────▶  │    Ollama    │
│ (Docker)     │                   │ (Docker)     │
└──────────────┘                   └──────────────┘
│
│ volume
▼
data/rag_storage (vector DB + graph + cache)

````

- **LightRAG**: Server RAG + WebUI
- **Ollama**: Chạy LLM + Embedding
- **rag_storage**: Lưu vector, graph → KHÔNG được đổi embedding dim khi đã có dữ liệu

---

## 2. Bật / tắt hệ thống

### 2.1 Bật toàn bộ (LightRAG + Ollama)

```bash
docker compose up -d
````

Kiểm tra:

```bash
docker ps
```

---

### 2.2 Tắt toàn bộ

```bash
docker compose down
```

---

### 2.3 Tắt và xoá toàn bộ dữ liệu (reset sạch)

⚠️ **Sẽ mất toàn bộ dữ liệu đã index**

```bash
docker compose down -v
```

Nếu bị lỗi permission với `rag_storage`:

```bash
docker run --rm \
  -v $(pwd)/data/rag_storage:/data \
  busybox sh -c "rm -rf /data/*"
```

---

## 3. Truy cập WebUI

Sau khi chạy xong:

* WebUI: [http://localhost:9621](http://localhost:9621)
* API Docs: [http://localhost:9621/docs](http://localhost:9621/docs)

---

## 4. Quy trình sử dụng LightRAG (cơ bản)

### Bước 1. Upload tài liệu

* Vào tab **Documents**
* Upload PDF / TXT / DOCX
* Chờ trạng thái **Completed**

### Bước 2. Kiểm tra Knowledge Graph

* Vào tab **Knowledge Graph**
* Xem entity / relation đã được tạo

### Bước 3. Hỏi đáp (Retrieval)

* Vào tab **Retrieval**
* Nhập câu hỏi → LightRAG sẽ:

  * tìm entity
  * tìm chunk liên quan
  * gọi LLM trả lời

---

## 5. Cấu hình model (QUAN TRỌNG)

### 5.1 LLM (mô hình trả lời)

Trong `.env`:

```env
LLM_BINDING=ollama
LLM_MODEL=llama3.1:8b
LLM_BINDING_HOST=http://ollama:11434
```

Một số model phổ biến:

* `llama3.1:8b`
* `qwen2.5:7b`
* `mistral:7b`

---

### 5.2 Embedding model (CỰC KỲ QUAN TRỌNG)

Ví dụ **nomic-embed-text**:

```env
EMBEDDING_BINDING=ollama
EMBEDDING_MODEL=nomic-embed-text:latest
EMBEDDING_DIM=768
EMBEDDING_BINDING_HOST=http://ollama:11434
```

Hoặc **bge-m3**:

```env
EMBEDDING_MODEL=bge-m3:latest
EMBEDDING_DIM=1024
```

⚠️ **LUẬT BẤT DI BẤT DỊCH**:

> ❌ Không được đổi `EMBEDDING_DIM` khi đã có dữ liệu trong `rag_storage`
> ✅ Muốn đổi embedding → **xoá sạch rag_storage trước**

---

## 6. Đổi model đúng cách (KHÔNG BỊ LỖI)

### 6.1 Đổi LLM (AN TOÀN)

✅ Có thể đổi **bất cứ lúc nào**:

```env
LLM_MODEL=llama3.1:8b → qwen2.5:7b
```

Không cần xoá dữ liệu.

---

### 6.2 Đổi Embedding (NGUY HIỂM)

❌ Sai:

```env
EMBEDDING_DIM=1024 → 768 (khi đã index)
```

Sẽ gây lỗi:

```
Embedding dim mismatch
```

✅ Đúng:

```bash
docker compose down
rm -rf data/rag_storage/*
docker compose up -d
```

---

## 7. Kiểm tra Ollama có hoạt động không

### 7.1 Kiểm tra từ host

```bash
curl http://localhost:11434/api/tags
```

---

### 7.2 Kiểm tra model trong Ollama container

```bash
docker exec -it ollama ollama list
```

---

## 8. Các lỗi thường gặp & cách xử lý

### ❌ Lỗi: Failed to connect to Ollama

Nguyên nhân:

* LightRAG không gọi được Ollama

Cách sửa:

```env
LLM_BINDING_HOST=http://ollama:11434
EMBEDDING_BINDING_HOST=http://ollama:11434
```

⚠️ **KHÔNG dùng `localhost` hoặc `172.17.0.1` trong docker-compose**

---

### ❌ Lỗi: Embedding dim mismatch

```
expected: 768, but loaded: 1024
```

👉 Đã đổi embedding model khi còn dữ liệu cũ

Cách sửa:

```bash
rm -rf data/rag_storage/*
```

---

### ❌ Lỗi: json: unsupported value: NaN

Nguyên nhân:

* Một số embedding model không ổn với text rất ngắn / ký tự đặc biệt

Cách xử lý:

* Dùng:

  * `nomic-embed-text`
  * hoặc `bge-m3`
* Tránh embedding model experimental

---

## 9. Khi nào cần reset toàn bộ?

Reset khi:

* Đổi embedding model
* Đổi embedding dim
* Data bị lỗi / index hỏng

Không cần reset khi:

* Đổi LLM
* Đổi prompt
* Đổi query

---

## 10. Gợi ý cấu hình ổn định (khuyến nghị)

```env
LLM_MODEL=llama3.1:8b
EMBEDDING_MODEL=nomic-embed-text:latest
EMBEDDING_DIM=768
MAX_ASYNC=4
MAX_PARALLEL_INSERT=2
```

Ổn định, ít lỗi, chạy tốt CPU/GPU.

---

## 11. Kết luận nhanh

* ✅ Upload xong → hỏi ngay ở **Retrieval**
* ✅ Đổi LLM thoải mái
* ❌ Đổi embedding phải xoá dữ liệu
* ❌ Không dùng `localhost` giữa container

