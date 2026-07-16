# Bài toán 08: Đồng bộ hóa Cache và Cơ sở dữ liệu (Keeping Cache and Database in Sync)

---

## 1. Đặt ra vấn đề / tình huống (Problem Statement)

Bạn đang phát triển hệ thống danh mục sản phẩm thương mại điện tử, sử dụng Redis làm bộ nhớ đệm (cache) đứng trước PostgreSQL làm cơ sở dữ liệu gốc (source of truth). Lượng tải đạt khoảng **40K RPS** vào giờ cao điểm.

Trong môi trường staging, tỷ lệ tìm thấy dữ liệu trên cache (cache hit ratio) đạt mức ấn tượng 94%, với thời gian phản hồi dưới 20ms. Mọi thứ hoạt động trơn tru.

Tuy nhiên, sau khi triển khai lên production khoảng 48 giờ, các ticket hỗ trợ khách hàng bắt đầu đổ về dồn dập:

- Khách hàng nhìn thấy sai giá sản phẩm khi thanh toán.
- Số lượng tồn kho hiển thị cũ.
- Sản phẩm hiển thị trạng thái "còn hàng" nhưng thực tế đã được bán hết và vận chuyển từ 3 tiếng trước.

**Thiết lập hệ thống:**

- Ứng dụng (Node.js) &rarr; Redis (cache) &rarr; PostgreSQL (source of truth).
- **Các luồng ghi:** Admin panel (cập nhật giá sản phẩm), inventory service (giảm tồn kho khi thanh toán), và order service (ghi nhận sự kiện mua hàng).
- **Các luồng đọc:** Trang chi tiết sản phẩm, kết quả tìm kiếm, và luồng thanh toán.

Cache đang bị rò rỉ dữ liệu cũ (stale data) trên production. Nhóm phát triển đang tranh luận gay gắt về chiến lược xóa cache (cache invalidation). Bạn sẽ chọn phương án nào?

### Câu hỏi trắc nghiệm

Lựa chọn chiến lược đồng bộ hóa cache nào sau đây là tối ưu nhất để khắc phục triệt để lỗi dữ liệu cũ trên?

- **A.** **Write-through (Ghi trực tiếp qua cache)** — mỗi tác vụ ghi sẽ cập nhật cả cache và Postgres trong cùng một transaction. Đảm bảo dữ liệu cache không bao giờ cũ.
- **B.** **Write-behind / Write-back (Ghi hoãn lại)** — dữ liệu ghi được lưu vào Redis trước, sau đó một worker không đồng bộ sẽ đẩy dữ liệu xuống Postgres theo từng lô (batches).
- **C.** **Cache-aside (Cập nhật bên lề / Tải chậm)** — ứng dụng ghi trực tiếp vào Postgres, đồng thời xóa key cache tương ứng (invalidate key), để lượt đọc tiếp theo tự động nạp lại dữ liệu mới vào cache.
- **D.** **Read-through (Đọc trực tiếp qua cache)** — Redis đóng vai trò trung gian xử lý, tự động kiểm tra và kéo dữ liệu mới từ Postgres về khi bị miss cache.

**ĐÁP ÁN ĐÚNG:** **C. Cache-aside (Cập nhật bên lề)**

---

## 2. Trạng thái / Cấu hình của hệ thống hiện tại (Current System State / Configuration)

Hệ thống hiện tại triển khai mô hình Cache đơn giản không đồng bộ giữa 3 dịch vụ ghi độc lập (admin panel, inventory service, order service) và luồng đọc dữ liệu. Do các dịch vụ ghi cập nhật dữ liệu trực tiếp vào PostgreSQL nhưng không có cơ chế invalidate (xóa) cache Redis tương ứng một cách nhất quán, dữ liệu trên Redis nhanh chóng bị cũ (stale data).

Khách hàng truy vấn đọc từ trang chi tiết sản phẩm hay trang tìm kiếm sẽ liên tục hit cache Redis chứa thông tin cũ, gây sai lệch giá cả và tồn kho nghiêm trọng.

---

## 3. Thiết kế tổng quan (High-level Design)

Để giải quyết vấn đề, PostgreSQL phải luôn đóng vai trò là nguồn chân lý duy nhất (Source of Truth) của toàn hệ thống. Chúng ta chuyển dịch sang mô hình **Cache-aside (Cập nhật bên lề)**.

```mermaid
sequenceDiagram
    autonumber
    participant App as Application Server
    participant Redis as Redis Cache
    participant DB as Postgres (Source of Truth)

    note over App, DB: Luồng Ghi Dữ Liệu (Write Path)
    App->>DB: Cập nhật giá sản phẩm (UPDATE product SET price = 499)
    DB-->>App: Thành công
    App->>Redis: Xóa key cache (DEL product:123)
    Redis-->>App: Thành công

    note over App, DB: Luồng Đọc Dữ Liệu (Read Path)
    App->>Redis: Lấy dữ liệu (GET product:123)
    Redis-->>App: Cache Miss (Không tìm thấy)
    App->>DB: Truy vấn dữ liệu mới (SELECT * FROM product WHERE id = 123)
    DB-->>App: Trả về dữ liệu mới nhất (price = 499)
    App->>Redis: Nạp lại dữ liệu vào Cache (SET product:123 499)
    Redis-->>App: Thành công
```

**Nguyên lý hoạt động:**

- **Tác vụ Ghi:** Khi có cập nhật từ admin panel hay inventory service, ứng dụng sẽ thực hiện cập nhật PostgreSQL trước, sau đó phát lệnh xóa key tương ứng trên Redis cache.
- **Tác vụ Đọc:** Luồng đọc luôn tra cứu Redis cache trước. Nếu bị miss cache, nó sẽ tự động tìm xuống Postgres để nạp dữ liệu mới nhất và ghi ngược lại vào Redis.

---

## 4. Thiết kế chi tiết (Detailed Design)

### 4.1. Race Condition kinh điển của Cache-aside và cách xử lý

Mặc dù Cache-aside rất an toàn, nó vẫn tồn tại một kịch bản Race Condition hiếm gặp khi đọc/ghi đồng thời (Martin Kleppmann):

1. **Bước 1:** Key `product:123` không có trong cache (hoặc vừa bị hết hạn).
2. **Bước 2:** Luồng đọc (Reader A) bị miss cache, truy vấn Postgres lấy dữ liệu cũ (ví dụ: `price = 100`).
3. **Bước 3:** Luồng ghi (Writer B) cập nhật Postgres thành giá trị mới `price = 150`, sau đó gửi lệnh xóa key `product:123` trên Redis.
4. **Bước 4:** Luồng đọc (Reader A) bị trễ do GC pause hoặc lag mạng, lúc này mới ghi đè giá trị cũ `price = 100` ngược vào Redis cache.
5. **Kết quả:** Cache bị kẹt dữ liệu cũ `100` vĩnh viễn, trong khi DB đã cập nhật `150`.

#### Giải pháp khắc phục

- **Thiết lập TTL ngắn:** Luôn cấu hình thời gian sống (TTL) ngắn cho các key cache (ví dụ: 5-10 phút). Nếu xảy ra race condition, cache cũng sẽ tự phục hồi sau khi hết TTL.
- **Đồng bộ bằng Change Data Capture (CDC):** Thay vì xóa cache trực tiếp từ ứng dụng, sử dụng công cụ CDC (như Debezium + Kafka) để lắng nghe Transaction Log (WAL) của PostgreSQL. Khi Postgres ghi nhận commit cập nhật giá trị mới, Debezium phát hiện sự kiện và kích hoạt worker xóa cache Redis. Cách này đảm bảo tính nhất quán tuyệt đối và giảm tải cho code ứng dụng.

### 4.2. Chống hiện tượng Cache Stampede (Thundering Herd) dưới tải 40K RPS

Khi một sản phẩm "hot" (ví dụ: iPhone mới mở bán) bị xóa cache (do admin cập nhật), dưới tải 40K RPS, hàng ngàn request đọc đồng thời sẽ bị miss cache cùng lúc và đổ dồn xuống Postgres để nạp lại dữ liệu, khiến DB bị quá tải và sập nguồn (Database Crash).

**Giải pháp Single-flight (Promise Sharing):** Ở tầng ứng dụng (Node.js), khi phát hiện cache miss cho một key cụ thể, hệ thống sẽ chỉ cho phép duy nhất **một** request thực thi truy vấn xuống Postgres để nạp cache. Các request khác đến cùng lúc sẽ xếp hàng chờ và cùng lắng nghe kết quả trả về từ Promise của request đầu tiên.

### 4.3. Ví dụ mã nguồn

#### TypeScript (Express + Redis Cache-aside với Single-flight)

```typescript
import { Request, Response } from "express";
import Redis from "ioredis";
import { Pool } from "pg";

const redis = new Redis();
const dbPool = new Pool({
  connectionString: "postgres://user:pass@db-server:5432/store",
});

// Map lưu giữ các Promise đang chạy để phục vụ Single-flight
const activePromises = new Map<string, Promise<any>>();

async function fetchProductFromDb(productId: string): Promise<any> {
  const result = await dbPool.query("SELECT * FROM products WHERE id = $1", [
    productId,
  ]);
  return result.rows.length === 0 ? null : result.rows[0];
}

async function getProductWithSingleFlight(productId: string): Promise<any> {
  const cacheKey = `product:${productId}`;

  // 1. Kiểm tra cache Redis
  const cachedData = await redis.get(cacheKey);
  if (cachedData) return JSON.parse(cachedData);

  // 2. Cache Miss -> Áp dụng Single-flight chống Cache Stampede
  if (activePromises.has(cacheKey)) {
    console.log(
      `[INFO] Single-flight: Chia sẻ Promise đọc DB cho key ${cacheKey}`,
    );
    return activePromises.get(cacheKey);
  }

  // Khởi tạo Promise nạp cache mới
  const dbPromise = (async () => {
    try {
      const product = await fetchProductFromDb(productId);
      if (product) {
        // Nạp lại cache với TTL ngắn là 300 giây (5 phút) để tự phục hồi
        await redis.set(cacheKey, JSON.stringify(product), "EX", 300);
      }
      return product;
    } finally {
      // Dọn dẹp map khi hoàn thành
      activePromises.delete(cacheKey);
    }
  })();

  activePromises.set(cacheKey, dbPromise);
  return dbPromise;
}

export async function getProductController(req: Request, res: Response) {
  const { productId } = req.params;
  try {
    const product = await getProductWithSingleFlight(productId);
    if (!product)
      return res.status(404).json({ error: "Không tìm thấy sản phẩm" });
    return res.status(200).json(product);
  } catch (error: any) {
    return res.status(500).json({ error: error.message });
  }
}
```

#### Java (Spring Boot + Spring Cache)

```java
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.beans.factory.annotation.Autowired;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class ProductService {
    private static final Logger log = LoggerFactory.getLogger(ProductService.class);

    @Autowired
    private ProductRepository productRepository;

    // 1. Luồng đọc dữ liệu (Read Path)
    // sync = true kích hoạt cơ chế khóa nội bộ (synchronization) của Spring,
    // giúp chặn hiện tượng Cache Stampede bằng cách chỉ cho 1 thread truy vấn DB.
    @Cacheable(value = "product", key = "#productId", sync = true)
    public Product getProductById(String productId) {
        log.info("Cache Miss: Đang truy vấn Database PostgreSQL cho sản phẩm {}", productId);
        return productRepository.findById(productId)
                .orElseThrow(() -> new ResourceNotFoundException("Sản phẩm không tồn tại"));
    }

    // 2. Luồng ghi dữ liệu (Write Path)
    // Cập nhật Postgres, sau đó Spring AOP tự động thực hiện xóa key tương ứng trong Redis.
    @CacheEvict(value = "product", key = "#product.id")
    public Product updateProduct(Product product) {
        log.info("Cập nhật thông tin sản phẩm vào DB: {}", product.getId());
        Product updated = productRepository.save(product);
        log.info("Xóa cache Redis cho sản phẩm {}", product.getId());
        return updated;
    }
}
```

---

## 5. Các giải pháp & Đánh đổi (Solutions & Trade-offs)

Dưới đây là bảng so sánh chi tiết các mô hình đồng bộ hóa cache:

| Tiêu chí                                | Cache-aside _(Phương án C)_                                                                     | Write-through _(Phương án A)_                                                         | Write-behind / Write-back _(Phương án B)_                                                                             | Read-through _(Phương án D)_                                  |
| :-------------------------------------- | :---------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------ | :-------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------ |
| **Tính nhất quán dữ liệu**              | **Khá tốt**. Dễ kiểm soát, có thể đạt mức tuyệt đối nếu kết hợp thêm CDC (Change Data Capture). | Tốt. Cả hai nơi được ghi đồng thời (trên lý thuyết).                                  | **Kém**. Chấp nhận rò rỉ và mất mát dữ liệu tạm thời khi Redis gặp sự cố hoặc trễ đồng bộ.                            | Khá tốt. Nhưng bị động vì chỉ kích hoạt ở chiều đọc.          |
| **Độ trễ khi ghi (Write Latency)**      | **Thấp**. Chỉ tốn thêm chi phí cho 1 lệnh xóa key bất đồng bộ siêu nhanh trên Redis.            | **Cao**. Phải đợi ghi thành công trên cả Postgres và Redis cùng lúc.                  | **Cực thấp**. Chỉ ghi trực tiếp lên bộ nhớ RAM của Redis rồi trả kết quả ngay.                                        | Thấp. Tác vụ ghi thực hiện trực tiếp vào DB gốc.              |
| **Độ trễ khi đọc (Read Latency)**       | Thấp khi hit cache. Tốn thêm thời gian nạp khi miss cache lần đầu.                              | Thấp. Dữ liệu luôn có sẵn trên cache.                                                 | Thấp.                                                                                                                 | Thấp khi hit cache.                                           |
| **Khả năng chịu lỗi (Fault tolerance)** | **Cực tốt**. Nếu Redis sập, ứng dụng vẫn đọc/ghi trực tiếp xuống Postgres bình thường.          | Kém. Luồng ghi bị ràng buộc chặt vào Redis. Nếu Redis sập, toàn bộ tác vụ ghi bị lỗi. | **Rất kém**. Nếu máy chủ Redis sập đột ngột (OOM-killed), các giao dịch chưa kịp ghi xuống Postgres sẽ mất vĩnh viễn. | Trung bình.                                                   |
| **Độ phức tạp triển khai**              | Thấp. Code ứng dụng dễ viết và kiểm soát luồng.                                                 | Trung bình. Phải xử lý transaction phân tán phức tạp ở ứng dụng.                      | Cao. Phải viết thêm worker xử lý gom lô và ghi xuống Postgres an toàn.                                                | Cao. Đòi hỏi cấu hình plugin tích hợp giữa Redis và Postgres. |

---

## 6. Explanation (Giải thích chi tiết & Lựa chọn tối ưu)

### Tại sao Cache-aside (C) là lựa chọn tối ưu nhất?

- **Độ sẵn sàng (High Availability) cao:** Trái ngược với Write-through hay Write-behind coi Redis như một phần bắt buộc của luồng nghiệp vụ ghi, Cache-aside chỉ xem Redis là lớp tối ưu hiệu năng đọc. Nếu cụm Redis bị sập đột ngột, ứng dụng vẫn chạy bình thường bằng cách truy cập trực tiếp PostgreSQL (dù tốc độ phản hồi sẽ chậm hơn). Đây là thuộc tính sống còn của các hệ thống thương mại điện tử lớn.
- **Phù hợp với kiến trúc nhiều nguồn ghi (Multi-Writer):** Danh mục sản phẩm được cập nhật từ nhiều microservices độc lập (inventory service, order service, admin panel). Việc cố gắng đồng bộ hóa ghi đồng thời (Write-through) trên tất cả các dịch vụ này tới cả Postgres và Redis là không khả thi vì không có cơ chế transaction phân tán hiệu quả giữa chúng. Cách xóa key đơn giản của Cache-aside giúp đơn giản hóa luồng dữ liệu.

### Phân tích chi tiết các lựa chọn không tối ưu khác

- **Write-through (A) - Lỗ hổng Transaction phân tán:**
  Không có cơ chế transaction phân tán nguyên tử (atomic distributed transaction) giữa Redis và Postgres. Nếu bạn ghi Postgres thành công nhưng ghi Redis thất bại do timeout, dữ liệu cache sẽ bị cũ vĩnh viễn. Hơn nữa, việc ràng buộc chặt chẽ mọi luồng ghi vào sự hoạt động của Redis sẽ làm giảm nghiêm trọng tính chịu lỗi của toàn bộ hệ thống.
- **Write-behind (B) - Rủi ro mất mát tài chính của doanh nghiệp:**
  Write-behind đưa Redis làm nguồn chân lý ghi. Dữ liệu ghi được lưu tạm trên Redis rồi gom lô cập nhật Postgres sau. Nếu node Redis bị sập hoặc mất điện đột ngột trước khi worker kịp ghi xuống Postgres, các giao dịch mua hàng và cập nhật giá sẽ biến mất vĩnh viễn. Đây là thảm họa đối với các hệ thống thương mại điện tử liên quan đến tiền tệ và hàng hóa thực tế.
- **Read-through (D) - Thiếu khả năng cập nhật từ phía ghi:**
  Read-through là mô hình thụ động ở chiều đọc, tự động kéo dữ liệu từ DB khi miss cache. Tuy nhiên, nó không giải quyết được vấn đề khi có cập nhật ghi từ phía admin panel hay inventory service. Bạn vẫn phải tự thiết lập cơ chế xóa cache thủ công bên ngoài, biến nó trở lại thành mô hình Cache-aside nhưng với cấu trúc triển khai phức tạp hơn rất nhiều.
