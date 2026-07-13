# Bài toán 05: Lựa chọn chiến lược phân mảnh cơ sở dữ liệu (Choosing a Database Sharding Strategy)

---

## 1. Đặt ra vấn đề / tình huống (Problem Statement)

4 chiến lược phân mảnh cơ sở dữ liệu (database sharding) khác nhau sẽ dẫn đến 4 kết quả mở rộng quy mô hoàn toàn khác biệt. Chọn sai một lần, bạn có thể làm sập máy chủ cơ sở dữ liệu chính vào ngày Black Friday.

Bảng `orders` chạy trên PostgreSQL của bạn vừa vượt mức **500 triệu dòng**. Các truy vấn quét theo dải (range scans) trước đây chỉ mất 40ms nay đã tăng lên hơn **800ms**. Việc mở rộng máy chủ theo chiều dọc (vertical scaling) đã chạm giới hạn vật lý. Bạn buộc phải thực hiện phân mảnh (shard).

**Đặc thù tải công việc (workload) như sau:**

- **Quy mô:** Bảng `orders` hiện có 500 triệu dòng, tăng trưởng đều đặn 3 triệu dòng/tuần.
- **Tác vụ Đọc (80%):** Truy vấn có dạng `"Lấy lịch sử đơn hàng trong 30 ngày qua của khách hàng X"`.
- **Tác vụ Đọc phân tích (15%):** Các phép JOIN phục vụ phân tích dữ liệu theo khoảng thời gian.
- **Tác vụ Ghi (5%):** Tạo đơn hàng mới (tần suất ổn định ở mức 400 RPS, tăng gấp đôi vào các ngày khuyến mãi).

Bạn chọn chiến lược phân mảnh nào?

### Câu hỏi trắc nghiệm

Lựa chọn chiến lược phân mảnh nào sau đây là tối ưu nhất cho hệ thống trên?

- **A.** **Hash sharding theo order_id** — phân bổ dữ liệu đồng đều dựa trên cơ chế băm ID đơn hàng, không lo điểm nóng (hotspots).
- **B.** **Range sharding theo created_at** — phân mảnh theo khoảng thời gian tạo đơn hàng, giúp các truy vấn chuỗi thời gian (time-series) chạy cục bộ trên một shard.
- **C.** **Directory-based sharding (Phân mảnh dựa trên danh mục/bảng tra cứu)** — sử dụng một bảng ánh xạ tra cứu (lookup table) để định tuyến khách hàng đến shard tương ứng.
- **D.** **Consistent hashing với các nút ảo (virtual nodes)** — phân bổ bằng băm nhất quán để dễ dàng tái cân bằng tải khi thêm máy chủ.

**ĐÁP ÁN ĐÚNG:** **C. Directory-based sharding (Phân mảnh dựa trên danh mục/bảng tra cứu)**

---

## 2. Trạng thái / Cấu hình của hệ thống hiện tại (Current System State / Configuration)

Hệ thống hiện tại đang sử dụng một cơ sở dữ liệu PostgreSQL đơn lẻ (Single Instance) để lưu trữ toàn bộ dữ liệu đơn hàng. Khi kích thước bảng `orders` vượt mốc 500 triệu dòng:

- **Dung lượng Index vượt quá RAM:** Chỉ mục (B-Tree Index) trên cột `customer_id` và `created_at` phình to quá lớn, không thể nằm trọn trong RAM (Buffer Pool). Các truy vấn quét dải (Range Scan) buộc phải truy cập dữ liệu trực tiếp từ ổ đĩa cứng (Random Disk I/O), khiến thời gian phản hồi tăng từ 40ms lên hơn 800ms.
- **Nghẽn cổ chai ghi (Write Bottleneck):** Tải ghi 400 RPS (tăng lên 800+ RPS vào các đợt khuyến mãi) liên tục ghi vào Write-Ahead Log (WAL) và gây tranh chấp khóa (Lock Contention) trên các chỉ mục, làm chậm toàn bộ các tác vụ đọc.

---

## 3. Thiết kế tổng quan (High-level Design)

Để giải quyết triệt để giới hạn hiệu năng của PostgreSQL đơn lẻ, chúng ta chuyển dịch sang kiến trúc cơ sở dữ liệu phân mảnh (Shared-Nothing Sharding Architecture). Theo đó, dữ liệu của bảng `orders` sẽ được phân bổ trên nhiều máy chủ cơ sở dữ liệu vật lý độc lập (Shards).

Chiến dịch tối ưu nhất cho bài toán này là **Directory-based Sharding (Phân mảnh dựa trên danh mục)**. Ý tưởng cốt lõi là định tuyến dữ liệu theo **`customer_id`** làm khóa phân mảnh (Shard Key), giúp gom toàn bộ đơn hàng của cùng một khách hàng về cùng một máy chủ đích.

### Sơ đồ 1: Luồng Directory-based Sharding (Định tuyến tối ưu)

```mermaid
graph TD
    subgraph s_dir ["Directory-based Sharding (Định tuyến thông minh)"]
        Client[Client Query: Đơn hàng của Khách hàng X] --> App[App Server]
        App -->|1. Tra cứu nhanh| Lookup[Bảng ánh xạ customer_shard_map trong Redis]
        Lookup -->|2. Trả về: Khách hàng X ở Shard 2| App
        App -->|3. Truy vấn trực tiếp| S2[(Shard 2: Chứa toàn bộ orders của khách hàng X)]
        S1[(Shard 1)]
        S3[(Shard 3)]
        S4[(Shard 4)]
    end
```

### Sơ đồ 2: Luồng Hash Sharding theo order_id (Scatter-Gather Bottleneck)

Nếu chúng ta chọn phương án băm theo `order_id` (Phương án A), dữ liệu đơn hàng của cùng một khách hàng sẽ bị xé nhỏ và phân bổ ngẫu nhiên trên toàn bộ các shard, dẫn đến hiện tượng nghẽn cổ chai do phân tán và thu thập dữ liệu (Scatter-Gather fan-out).

```mermaid
graph TD
    subgraph s_hash ["Hash Sharding (Scatter-Gather fan-out)"]
        Client[Client Query: Đơn hàng của Khách hàng X] --> App[App Server]
        App -->|Truy vấn song song 4 Shards| S1[(Shard 1: Chứa order_1)]
        App -->|Truy vấn song song 4 Shards| S2[(Shard 2: Chứa order_2)]
        App -->|Truy vấn song song 4 Shards| S3[(Shard 3: Chứa order_3)]
        App -->|Truy vấn song song 4 Shards| S4[(Shard 4: Chứa order_4)]
        App -.-> Note["Chú ý: Phải chờ shard chậm nhất phản hồi để gom dữ liệu!"]
    end
```

---

## 4. Thiết kế chi tiết (Detailed Design)

### 4.1. Thiết kế Bảng Tra cứu (Lookup Table) và Cache Fallback

Bảng định tuyến `customer_shard_map` sẽ được lưu trữ trong một database Postgres tập trung (Metadata Database) và được cache lại trên Redis để tối ưu hóa hiệu năng tra cứu:

- **Cấu trúc Redis Cache**: Key dạng String `customer_shard:{customer_id}` lưu trữ giá trị `shard_id`.
- **Cơ chế Fallback khi Cache Miss**:
  1. Khi có request, Server gửi lệnh `GET customer_shard:{customer_id}` tới Redis.
  2. Nếu có (Cache Hit), sử dụng kết nối tới Shard tương ứng.
  3. Nếu không có (Cache Miss), Server truy vấn xuống Metadata Database: `SELECT shard_id FROM customer_shard_map WHERE customer_id = ?`.
  4. Trả về kết quả và ghi ngược (Write-back) vào Redis với thời gian sống (TTL) là 1 giờ (3600 giây).

### 4.2. Chiến lược di chuyển dữ liệu Tenant lớn (Whales Migration) với downtime gần như bằng không

Trong Directory-based sharding, do chúng ta có toàn quyền kiểm soát bảng ánh xạ, khi một shard bị quá tải cục bộ vì chứa một khách hàng có lượng đơn hàng khổng lồ (Whale tenant), ta tiến hành di chuyển họ sang Shard trống khác theo quy trình:

1. Thiết lập cấu hình Logical Replication giữa Shard hiện tại (Source) và Shard mới (Target) chỉ dành cho bảng `orders` của `customer_id` cần di chuyển.
2. Đợi dữ liệu được đồng bộ hóa thời gian thực và độ trễ đồng bộ (Replication Lag) tiệm cận 0.
3. Chuyển đổi trạng thái của khách hàng này sang `read-only` tạm thời trên API Gateway để ngăn chặn ghi dữ liệu mới.
4. Đồng bộ nốt các bản ghi delta cuối cùng, cập nhật cột `shard_id` trong Metadata DB và Redis trỏ về Shard mới.
5. Mở khóa cho phép ghi dữ liệu bình thường. Tiến trình di chuyển hoàn tất với thời gian nghẽn ghi chỉ trong vài mili-giây.

### 4.3. Ví dụ mã nguồn định tuyến phân mảnh

#### TypeScript (Express + Redis Lookup Routing)

```typescript
import { Request, Response, NextFunction } from "express";
import Redis from "ioredis";
import { Pool } from "pg";

const redis = new Redis();
const lookupDbPool = new Pool({
  connectionString: "postgres://lookup_user:pass@lookup-db:5432/metadata",
});

// Khởi tạo Connection Pools cho từng Shard
const shardPools: Record<string, Pool> = {
  shard_1: new Pool({
    connectionString: "postgres://user:pass@shard-db-1:5432/orders",
  }),
  shard_2: new Pool({
    connectionString: "postgres://user:pass@shard-db-2:5432/orders",
  }),
  shard_3: new Pool({
    connectionString: "postgres://user:pass@shard-db-3:5432/orders",
  }),
};

async function getShardIdForCustomer(customerId: string): Promise<string> {
  const cacheKey = `customer_shard:${customerId}`;

  // 1. Tra cứu từ Redis Cache
  const cachedShard = await redis.get(cacheKey);
  if (cachedShard) return cachedShard;

  // 2. Cache Miss -> Fallback xuống Metadata Database
  const queryResult = await lookupDbPool.query(
    "SELECT shard_id FROM customer_shard_map WHERE customer_id = $1",
    [customerId],
  );

  if (queryResult.rows.length === 0) {
    throw new Error(
      `Customer ${customerId} không tồn tại trong sơ đồ phân mảnh.`,
    );
  }

  const shardId = queryResult.rows[0].shard_id;

  // 3. Write-back vào Redis Cache với TTL 1 giờ
  await redis.set(cacheKey, shardId, "EX", 3600);

  return shardId;
}

export async function getOrdersForCustomer(req: Request, res: Response) {
  const customerId = req.query.customerId as string;
  if (!customerId) return res.status(400).json({ error: "Missing customerId" });

  try {
    const shardId = await getShardIdForCustomer(customerId);
    const targetPool = shardPools[shardId];

    // Thực thi câu lệnh trực tiếp trên 1 Shard duy nhất
    const result = await targetPool.query(
      "SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 50",
      [customerId],
    );

    return res.status(200).json(result.rows);
  } catch (error: any) {
    return res.status(500).json({ error: error.message });
  }
}
```

#### Java (Spring Boot + AbstractRoutingDataSource)

Trong Java, Spring JDBC hỗ trợ định tuyến Datasource động qua class `AbstractRoutingDataSource`:

```java
import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.After;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.jdbc.core.JdbcTemplate;
import java.util.concurrent.TimeUnit;

// 1. ThreadLocal Context để lưu Shard Key trong luồng Request hiện tại
public class ShardContext {
    private static final ThreadLocal<String> CONTEXT = new ThreadLocal<>();

    public static void setShardId(String shardId) { CONTEXT.set(shardId); }
    public static String getShardId() { return CONTEXT.get(); }
    public static void clear() { CONTEXT.remove(); }
}

// 2. Class định tuyến Source động của Spring
public class DynamicRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return ShardContext.getShardId();
    }
}

// 3. Interceptor/Aspect tự động tra cứu và định tuyến
@Aspect
@Component
public class ShardingAspect {

    @Autowired
    private StringRedisTemplate redisTemplate;
    @Autowired
    private JdbcTemplate lookupJdbcTemplate;

    @Before("execution(* com.example.service.OrderService.getOrdersForCustomer(..)) && args(customerId)")
    public void route(String customerId) {
        String cacheKey = "customer_shard:" + customerId;
        String shardId = redisTemplate.opsForValue().get(cacheKey);

        if (shardId == null) {
            // Fallback xuống metadata database
            shardId = lookupJdbcTemplate.queryForObject(
                "SELECT shard_id FROM customer_shard_map WHERE customer_id = ?",
                String.class,
                customerId
            );
            // Ghi nhận cache với TTL 1h
            redisTemplate.opsForValue().set(cacheKey, shardId, 1, TimeUnit.HOURS);
        }

        ShardContext.setShardId(shardId);
    }

    @After("execution(* com.example.service.OrderService.getOrdersForCustomer(..))")
    public void clear() {
        ShardContext.clear(); // Giải phóng thread tránh rò rỉ dữ liệu
    }
}
```

---

## 5. Các giải pháp & Đánh đổi (Solutions & Trade-offs)

Dưới đây là bảng so sánh chi tiết các chiến lược phân mảnh cơ sở dữ liệu:

| Chiến lược phân mảnh                   | Cân bằng tải ghi (Write Load Distribution)                  | Hiệu năng đọc theo nhóm (80% read queries)                                                                             | Khả năng tái cân bằng dữ liệu (Rebalancing)                                                                | Độ phức tạp quản lý & Triển khai                                                   | Rủi ro điểm nóng (Hotspots / Whales)                                                                                 |
| :------------------------------------- | :---------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------- |
| **Hash sharding** _(Phương án A)_      | **Cực tốt**. Dữ liệu được băm đều ngẫu nhiên trên toàn cụm. | **Rất kém**. Bắt buộc phải Scatter-Gather truy vấn tới tất cả các shard để thu thập lịch sử đơn hàng của 1 khách hàng. | Rất kém. Khi thêm shard mới, phần lớn dữ liệu phải được rehash và di chuyển.                               | Thấp. Được hỗ trợ sẵn bởi nhiều thư viện và cơ sở dữ liệu.                         | Thấp.                                                                                                                |
| **Range sharding** _(Phương án B)_     | **Tệ**. 100% dữ liệu mới ghi dồn vào shard hiện tại.        | Khá. Tốt nếu truy vấn theo dải thời gian, nhưng tệ nếu đọc theo customer_id.                                           | Trung bình. Phải chia nhỏ các dải cũ thủ công.                                                             | Thấp. Cấu hình phân chia dải đơn giản.                                             | Cực cao. Shard mới nhất sẽ liên tục bị quá tải (Hot Shard).                                                          |
| **Directory-based** _(Phương án C)_    | Tốt. Định tuyến dựa trên ánh xạ thực tế.                    | **Cực tốt**. Trả về kết quả lịch sử đơn hàng của khách hàng X chỉ trong 1 câu truy vấn trên 1 shard duy nhất.          | **Tuyệt vời**. Có thể di chuyển từng khách hàng cụ thể sang shard khác chỉ bằng cách cập nhật bảng ánh xạ. | **Cao**. Phải duy trì Metadata DB, cơ chế cache đồng bộ, và quản lý routing layer. | Trung bình. Có thể xử lý hotspot thủ công bằng cách di chuyển các Whales.                                            |
| **Consistent Hashing** _(Phương án D)_ | Tốt. Tự động phân bổ đều theo vòng tròn băm.                | Tốt khi đọc theo customer_id (nếu dùng customer_id làm khóa băm).                                                      | Tốt. Khi thêm nút mới chỉ cần di chuyển $1/N$ lượng dữ liệu.                                               | Trung bình. Cấu hình thuật toán phức tạp hơn hash thông thường.                    | Cao. Nếu nhiều khách hàng lớn vô tình rơi vào một nút băm, shard đó sẽ sập mà không thể can thiệp thủ công đơn giản. |

---

## 6. Explanation (Giải thích chi tiết & Lựa chọn tối ưu)

### Tại sao Directory-based sharding (C) là lựa chọn tối ưu nhất?

- **Tối ưu hóa 80% tác vụ đọc:** 80% tác vụ của hệ thống là đọc lịch sử đơn hàng của một khách hàng cụ thể. Bằng cách định tuyến phân mảnh theo `customer_id` thông qua bảng tra cứu, hệ thống cam kết toàn bộ đơn hàng của khách hàng X luôn nằm tập trung trên cùng một phân mảnh vật lý. Khi truy vấn, Server chỉ cần gọi trực tiếp tới shard đó, loại bỏ hoàn toàn hiện tượng Scatter-Gather giúp độ trễ giảm từ 800ms xuống chỉ còn khoảng ~20ms (tương đương tốc độ trên bảng nhỏ).

- **Khả năng quản lý Whales Tenant linh hoạt:** Ở quy mô lớn, các khách hàng lớn (ví dụ: các doanh nghiệp đặt hàng số lượng lớn) sẽ có dung lượng dữ liệu và lượng RPS gấp hàng ngàn lần người dùng thường. Directory-based sharding cho phép di chuyển các "Whales" này sang các shard riêng biệt, chuyên dụng một cách thủ công và cập nhật lại bảng tra cứu mà không cần phải thực hiện việc băm lại (rehash) hay tái cấu trúc toàn bộ dữ liệu của hệ thống.

### Phân tích chi tiết các lựa chọn không tối ưu khác

- **Hash sharding theo order_id (A) - Kẻ hủy diệt hiệu năng p99:**
  Băm theo ID đơn hàng sẽ phân tán đơn hàng của cùng một khách hàng ra toàn bộ các shard trong hệ thống. Để lấy lịch sử đơn hàng của một người, server của bạn phải thực hiện gửi truy vấn đồng thời tới cả $N$ shards (Scatter-Gather fan-out), rồi đợi cho đến khi shard chậm nhất phản hồi để tổng hợp dữ liệu. p99 latency của hệ thống lúc này sẽ bị kéo dài theo phân vị chậm nhất của mạng và đĩa cứng trên tất cả các shard.
- **Range sharding theo created_at (B) - Thiết kế tự gây sập hệ thống:**
  Phân mảnh theo thời gian sẽ tạo ra điểm nóng ghi cực lớn. Do toàn bộ giao dịch mới (tần suất 400+ RPS) có thời gian tạo ở hiện tại, chúng sẽ cùng ghi vào một shard duy nhất đại diện cho dải thời gian hiện tại. Shard này sẽ nhanh chóng bị sập do quá tải đĩa và kết nối vào các ngày khuyến mãi lớn (Black Friday), trong khi các shard lưu lịch sử trước đó hoàn toàn nhàn rỗi.
- **Consistent Hashing (D) - Thiếu khả năng can thiệp thủ công:**
  Mặc dù băm nhất quán giải quyết rất tốt bài toán tái phân bổ dữ liệu khi mở rộng/thu hẹp cụm server, nhưng nó hoạt động hoàn toàn tự động dựa trên hàm băm toán học. Bạn không thể chủ động chỉ định di chuyển một tenant cực lớn ra khỏi một shard đang bị quá tải cục bộ. Nếu thuật toán băm vô tình xếp 3 khách hàng lớn vào chung một shard vật lý, shard đó sẽ gặp sự cố quá tải mà bạn không thể cứu vãn một cách linh hoạt.
