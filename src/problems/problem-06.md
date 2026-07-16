# Bài toán 06: Khóa phân tán an toàn cho tác vụ định kỳ (Safe Distributed Locks for Cron Jobs)

---

## 1. Đặt ra vấn đề / tình huống (Problem Statement)

Trình lập lịch công việc (job scheduler) của bạn chạy trên 3 thực thể (instances) đứng sau một bộ cân bằng tải ALB (Application Load Balancer).

Cứ mỗi 5 phút, một cron job kích hoạt cùng một tác vụ `"generate-daily-report"` — và cả 3 instances đều tìm cách giành lấy tác vụ này để xử lý. Bạn chọn giải pháp khóa phân tán kinh điển sử dụng Redis:

`SET job:daily-report locked NX EX 300` &rarr; chỉ một instance giành được khóa thành công và thực thi tác vụ.

Hệ thống hoạt động hoàn hảo trong vòng một tháng, cho đến lúc 2 giờ sáng ngày thứ Bảy:

1. **Instance B** giành được khóa và bắt đầu chạy tác vụ. Sau 60 giây, pod chứa B bị lỗi tràn bộ nhớ (OOM-killed). Khóa vẫn tồn tại trên Redis với thời gian hết hạn (TTL) còn lại là 240 giây (4 phút).
2. Không có thực thể nào xử lý tác vụ báo cáo đó trong 4 phút tiếp theo do khóa vẫn bị chiếm dụng.
3. Khi khóa hết hạn, **Instance A** giành được khóa và bắt đầu chạy. Tuy nhiên, lúc này pod của **Instance B** cũng vừa được tự động khởi động lại và CŨNG đang tìm cách chạy tiếp tác vụ cũ. Kết quả là cả hai instances cùng xử lý một tác vụ trùng lặp.
4. **Nguy hiểm hơn:** Một đợt dừng chạy thu gom rác (slow GC pause) trên Instance A khiến nó bị tạm dừng hoạt động. Thời gian TTL của khóa trên Redis hết hạn. Redis lặng lẽ chuyển giao khóa này cho **Instance C**. Khi Instance A "tỉnh dậy", cả A và C đều tin rằng mình đang sở hữu khóa độc quyền và cùng ghi đè lên một tệp kết quả đầu ra.

Mẫu thiết kế (pattern) phân khóa an toàn ở đây là gì?

### Câu hỏi trắc nghiệm

Lựa chọn giải pháp nào sau đây là tối ưu nhất để bảo vệ hệ thống khỏi các lỗi trên?

- **A.** **Redis SETNX + TTL ngắn + fencing token (mã bảo vệ)** — sử dụng một số tăng dần đơn điệu (monotonically increasing token) mà tài nguyên hạ nguồn (downstream resource) sẽ kiểm tra trên mỗi lượt ghi.
- **B.** **Redlock** — chiếm khóa trên 5 nút Redis độc lập, áp dụng nguyên tắc đa số thắng (majority wins).
- **C.** **Khóa bi quan dựa trên DB (DB-based pessimistic lock)** — sử dụng cú pháp `SELECT ... FOR UPDATE` trên bảng quản lý tác vụ (jobs table), transaction của cơ sở dữ liệu sẽ giữ khóa này.
- **D.** **Kiểm soát đồng thời lạc quan (Optimistic concurrency)** — không dùng khóa. Mỗi instance cố gắng insert kết quả với một unique key duy nhất; DB sẽ tự động từ chối các bản ghi trùng lặp.

**ĐÁP ÁN ĐÚNG:** **A. Redis SETNX + TTL ngắn + fencing token (mã bảo vệ)**

---

## 2. Trạng thái / Cấu hình của hệ thống hiện tại (Current System State / Configuration)

Hệ thống hiện tại đang sử dụng dịch vụ khóa phân tán (Distributed Lock Service) đơn giản thông qua Redis:

- **Cấu hình khóa:** `SET lock:job:daily-report {random_value} NX EX 300`.
- **Luồng hoạt động:** Instance nào ghi khóa thành công sẽ thực thi tác vụ. Khi hoàn thành, instance đó thực thi script Lua để kiểm tra và giải phóng khóa.

```mermaid
sequenceDiagram
    autonumber
    participant AppA as Instance A
    participant Redis as Lock Service (Redis)
    participant AppC as Instance C
    participant Storage as Downstream Storage (DB)

    note over AppA, Redis: Instance A giành được khóa
    AppA->>Redis: Yêu cầu lấy khóa
    Redis-->>AppA: Cấp khóa thành công (TTL 5 phút)
    
    note over AppA: Instance A bị dừng do GC pause kéo dài...
    note over Redis: Khóa hết hạn TTL! Redis xóa khóa cũ.
    
    note over AppC, Redis: Instance C giành được khóa mới
    AppC->>Redis: Yêu cầu lấy khóa
    Redis-->>AppC: Cấp khóa thành công
    
    AppC->>Storage: Ghi dữ liệu báo cáo
    Storage-->>AppC: Ghi thành công

    note over AppA: Instance A tỉnh lại sau GC pause và cố gắng ghi đè dữ liệu cũ
    AppA->>Storage: Ghi dữ liệu báo cáo (Đè lên dữ liệu của Instance C!)
    Storage-->>AppA: Ghi thành công (Gây sai lệch/trùng lặp kết quả!)
```

### Các hạn chế lớn của kiến trúc hiện tại

- **Rủi ro rò rỉ khóa / Tranh chấp đồng thời (Split-brain):** Dịch vụ khóa (Redis/Zookeeper) hoạt động độc lập và hoàn toàn tách biệt với tài nguyên lưu trữ (Database). Dịch vụ khóa không có cách nào biết được tiến trình đang thực thi thực tế đã bị chết hay chỉ bị tạm dừng (GC Pause/Network Partition).
- **Thiếu cơ chế xác thực ở tầng tài nguyên:** Tài nguyên lưu trữ chấp nhận mọi yêu cầu ghi mà không xác thực xem client thực hiện yêu cầu đó có thực sự đang giữ khóa hợp lệ tại thời điểm ghi hay không.

---

## 3. Thiết kế tổng quan (High-level Design)

Để giải quyết vấn đề, chúng ta bắt buộc phải đưa cơ chế **Fencing Token (Mã bảo vệ)** vào hệ thống. Nguyên tắc cốt lõi là: **Dịch vụ khóa không thể là nguồn chân lý duy nhất (Source of Truth). Nguồn chân lý cuối cùng phải nằm ở tài nguyên lưu trữ hạ nguồn (Downstream Storage).**

```mermaid
sequenceDiagram
    autonumber
    participant AppA as Instance A (Token 42)
    participant Redis as Lock Service (Redis)
    participant AppC as Instance C (Token 43)
    participant Storage as Downstream Storage (DB)

    note over AppA, Redis: Instance A giành được khóa
    AppA->>Redis: Yêu cầu lấy khóa
    Redis-->>AppA: Cấp khóa (Token 42)
    
    note over AppA: Instance A bị dừng do GC pause kéo dài...
    note over Redis: Khóa hết hạn TTL! Redis xóa khóa cũ.
    
    note over AppC, Redis: Instance C giành được khóa mới
    AppC->>Redis: Yêu cầu lấy khóa
    Redis-->>AppC: Cấp khóa (Token 43)
    
    AppC->>Storage: Ghi dữ liệu kèm Token 43
    Storage->>Storage: Lưu Token 43 làm mốc cao nhất
    Storage-->>AppC: Ghi thành công

    note over AppA: Instance A tỉnh lại sau GC pause và cố gắng ghi dữ liệu
    AppA->>Storage: Ghi dữ liệu kèm Token 42
    Storage->>Storage: Kiểm tra Token 42 < 43
    Storage-->>AppA: Từ chối yêu cầu ghi! (Fenced)
```

**Luồng hoạt động tổng quan:**

1. Mỗi khi lấy khóa thành công, dịch vụ khóa (Redis) sẽ sinh ra một số tăng dần đơn điệu (fencing token, ví dụ: 42, 43, 44...).
2. Khi client ghi dữ liệu xuống Storage, nó bắt buộc phải gửi kèm theo fencing token này.
3. Storage sẽ kiểm tra: Nếu token gửi lên nhỏ hơn hoặc bằng token lớn nhất mà Storage từng ghi nhận trước đó, Storage sẽ **từ chối** giao dịch ngay lập tức.

---

## 4. Thiết kế chi tiết (Detailed Design)

### 4.1. Sinh Fencing Token trên Redis và Kiểm tra Điều kiện ở Database

- **Sinh Token tự tăng (Monotonic Generator):** Sử dụng lệnh `INCR` của Redis trên một khóa seq chuyên biệt (ví dụ: `lock:daily-report:token_seq`). Giá trị trả về từ `INCR` đảm bảo tăng dần đơn điệu và duy nhất.
- **Xác thực tại Database (Conditional Write):** Bảng metadata của báo cáo cần lưu trữ cột `last_fencing_token`. Câu lệnh SQL cập nhật báo cáo phải kèm điều kiện kiểm tra token:

  ```sql
  UPDATE report_metadata 
  SET report_data = $1, last_fencing_token = $2, status = 'SUCCESS'
  WHERE job_id = $3 AND $2 > last_fencing_token;
  ```

  Nếu `last_fencing_token` hiện tại trên DB lớn hơn hoặc bằng token của request gửi lên, câu lệnh UPDATE sẽ không tác động đến dòng nào (`rowCount == 0`).
- **Xử lý khi bị Fenced (Fail-Fast):** Nếu câu lệnh UPDATE trả về 0 dòng bị tác động, ứng dụng sẽ quăng ra ngoại lệ (Exception) cảnh báo, thực hiện ghi log chi tiết mức `WARN/ERROR` để SRE theo dõi, và dừng ngay lập tức luồng chạy mà **không tự động retry** để tránh tạo thêm tải ảo cho hệ thống.

### 4.2. Triển khai trong Node.js (TypeScript)

```typescript
import Redis from 'ioredis';
import { Pool } from 'pg';

const redis = new Redis();
const dbPool = new Pool({ connectionString: 'postgres://user:pass@db-server:5432/reports' });

interface LockResult {
  acquired: boolean;
  token?: number;
}

async function acquireLockWithToken(lockName: string, ttlSeconds: number): Promise<LockResult> {
  const lockKey = `lock:${lockName}`;
  const tokenKey = `lock:${lockName}:token_seq`;

  // 1. Sinh số tăng dần đơn điệu bằng lệnh INCR của Redis
  const token = await redis.incr(tokenKey);

  // 2. Thử chiếm khóa bằng cách gán token làm value (chỉ ghi nếu chưa tồn tại)
  const isAcquired = await redis.set(lockKey, token.toString(), 'EX', ttlSeconds, 'NX');

  if (isAcquired === 'OK') {
    return { acquired: true, token };
  }
  
  return { acquired: false };
}

async function saveReportData(jobId: string, data: string, token: number) {
  // 3. Database hạ nguồn thực hiện xác thực token khi ghi nhận kết quả
  const query = `
    UPDATE report_metadata 
    SET report_data = $1, last_fencing_token = $2, status = 'success'
    WHERE job_id = $3 AND $2 > last_fencing_token
  `;
  
  const result = await dbPool.query(query, [data, token, jobId]);

  if (result.rowCount === 0) {
    // 4. Nếu rowCount === 0 nghĩa là token của chúng ta đã lỗi thời (bị fenced)
    console.error(`[WARN] Ghi dữ liệu thất bại! Token ${token} đã bị fenced bởi một token lớn hơn.`);
    throw new Error('Fencing Token Validation Failed. Transaction aborted.');
  }
  
  console.log(`[INFO] Ghi dữ liệu thành công với token ${token}`);
}

export async function runCronJob() {
  const jobId = 'daily-report';
  const lock = await acquireLockWithToken(jobId, 300); // Khóa 5 phút

  if (!lock.acquired || !lock.token) {
    console.log('[INFO] Instance khác đang giữ khóa. Hủy bỏ thực thi.');
    return;
  }

  try {
    // Giả lập xử lý tác vụ dài (ví dụ: crawl data, tính toán báo cáo...)
    const reportData = "Nội dung báo cáo hàng ngày...";
    
    // Ghi dữ liệu và kiểm tra token ở Postgres
    await saveReportData(jobId, reportData, lock.token);
  } catch (error: any) {
    console.error(`[ERROR] Cron job thất bại: ${error.message}`);
    // Fail-fast: Hủy bỏ tác vụ, không tự động retry để tránh lãng phí tài nguyên
  } finally {
    // Chỉ giải phóng khóa nếu token của chúng ta vẫn trùng với token đang lưu trên Redis
    const lockKey = `lock:${jobId}`;
    const currentLockValue = await redis.get(lockKey);
    if (currentLockValue === lock.token.toString()) {
      await redis.del(lockKey);
    }
  }
}
```

### 4.3. Triển khai trong Java (Spring Boot + JPA)

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.concurrent.TimeUnit;

// 1. Repository với câu lệnh UPDATE kiểm tra token điều kiện
@Repository
interface ReportRepository extends JpaRepository<ReportMetadata, String> {
    @Modifying
    @Transactional
    @Query("UPDATE ReportMetadata r SET r.reportData = ?1, r.lastFencingToken = ?2, r.status = 'SUCCESS' " +
           "WHERE r.jobId = ?3 AND ?2 > r.lastFencingToken")
    int updateReportWithFencing(String data, long token, String jobId);
}

// 2. Service thực thi định kỳ
@Service
public class CronJobService {
    private static final Logger log = LoggerFactory.getLogger(CronJobService.class);

    @Autowired
    private StringRedisTemplate redisTemplate;
    @Autowired
    private ReportRepository reportRepository;

    public void executeDailyReport() {
        String jobId = "daily-report";
        String lockKey = "lock:" + jobId;
        String tokenSeqKey = "lock:" + jobId + ":token_seq";

        // 1. Sinh token tăng dần đơn điệu
        Long token = redisTemplate.opsForValue().increment(tokenSeqKey);
        if (token == null) return;
        
        // 2. Thử chiếm khóa trên Redis với TTL 5 phút
        Boolean acquired = redisTemplate.opsForValue().setIfAbsent(lockKey, String.valueOf(token), 5, TimeUnit.MINUTES);

        if (Boolean.FALSE.equals(acquired)) {
            log.info("Không giành được khóa. Hủy bỏ tác vụ.");
            return;
        }

        try {
            // Giả lập xử lý báo cáo
            String data = "Dữ liệu báo cáo...";

            // 3. Ghi dữ liệu có kiểm tra token
            int updatedRows = reportRepository.updateReportWithFencing(data, token, jobId);

            if (updatedRows == 0) {
                // 4. Fail-Fast khi bị Fenced do token nhỏ hơn
                log.error("[WARN] Giao dịch bị từ chối! Token {} nhỏ hơn hoặc bằng token cao nhất trên DB.", token);
                throw new IllegalStateException("Fencing Token Validation Failed. Lock was lost during execution.");
            }
            log.info("Ghi báo cáo thành công với token {}", token);
        } catch (Exception e) {
            log.error("[ERROR] Lỗi thực thi cron job: {}", e.getMessage());
            // Không thực hiện retry tự động để SRE kiểm tra thủ công nguyên nhân GC/OOM
        } finally {
            // Giải phóng khóa an toàn nếu khóa trên Redis vẫn thuộc sở hữu của token này
            String currentToken = redisTemplate.opsForValue().get(lockKey);
            if (String.valueOf(token).equals(currentToken)) {
                redisTemplate.delete(lockKey);
            }
        }
    }
}
```

---

## 5. Các giải pháp & Đánh đổi (Solutions & Trade-offs)

Dưới đây là bảng so sánh chi tiết các giải pháp quản lý cron job:

| Giải pháp | Độ an toàn trước GC Pause / Network Partition | Độ phức tạp triển khai & Vận hành | Tác động hiệu năng DB chính | Khả năng bảo vệ các tác vụ ngoài (External side effects) | Chi phí tài nguyên tính toán |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **SETNX + Fencing Token** *(Phương án A)* | **Tuyệt đối**. Mọi request ghi muộn do GC pause đều bị Storage từ chối nhờ số tăng đơn điệu. | Trung bình. Cần cấu hình Redis `INCR` và câu lệnh cập nhật DB có điều kiện. | Thấp. Chỉ chạy 1 câu SQL UPDATE nhanh kèm check điều kiện. | Tốt khi kết hợp kiểm tra token ở API ngoài (nếu bên thứ 3 hỗ trợ). | Thấp. Chỉ 1 instance chạy tác vụ thực tế. |
| **Redlock** *(Phương án B)* | **Tệ**. Vẫn hoàn toàn bị đánh bại bởi GC pause nếu không có fencing token đi kèm. | **Cao**. Phải vận hành, kết nối và đồng bộ 5 cụm Redis độc lập. | Thấp. | Tệ. Tương tự như khóa SETNX thông thường. | Thấp. |
| **DB Pessimistic Lock** *(Phương án C)* | Tốt. Khóa giải phóng khi transaction kết thúc hoặc kết nối DB đứt. | Thấp. Dùng cú pháp SQL có sẵn (`FOR UPDATE`). | **Rất kém**. Giữ kết nối DB chính quá lâu (60s) làm nghẽn Connection Pool, ảnh hưởng tới toàn bộ ứng dụng. | Trung bình. Khóa DB không ngăn được việc ghi file trùng lên S3 hay gọi API ngoài. | Thấp. |
| **Optimistic Concurrency** *(Phương án D)* | Tốt. DB tự động từ chối bản ghi trùng dựa trên unique key. | Thấp. | Thấp. | **Tệ**. Cả 3 instance cùng chạy song song dẫn đến gọi API ngoài trùng lặp (spam Slack, gửi mail). | **Rất kém**. Lãng phí gấp 3 lần tài nguyên CPU/RAM để chạy các task trùng lặp vô ích. |

---

## 6. Explanation (Giải thích chi tiết & Lựa chọn tối ưu)

### Tại sao Fencing Token (A) là giải pháp tối ưu nhất?

- **Khắc phục triệt để lỗi GC Pause / Network Partition:** Dịch vụ khóa (Redis/ZooKeeper) hoạt động độc lập và không thể phân biệt giữa việc "tiến trình đã chết" hay "tiến trình chỉ đang bị tạm dừng do GC". Fencing Token giải quyết lỗ hổng này bằng cách chuyển trách nhiệm kiểm tra khóa cuối cùng cho chính tài nguyên lưu trữ (Database). Database so sánh mã token đơn điệu và dễ dàng chặn đứng yêu cầu ghi lỗi thời của Instance A (Token 42) sau khi Instance C (Token 43) đã cập nhật thành công.
- **Tiết kiệm tài nguyên và bảo vệ connection pool:** So với việc chiếm dụng connection DB chính trong 60 giây của DB Pessimistic Lock, giải pháp này chỉ chạy 1 câu truy vấn cập nhật nhanh ở cuối tiến trình, giúp giải phóng kết nối ngay lập tức.

### Phân tích chi tiết các lựa chọn không tối ưu khác

- **Redlock (B) - Sự an toàn ảo tưởng:**
  Nhiều kỹ sư cho rằng chạy Redlock trên 5 nút Redis sẽ an toàn tuyệt đối. Tuy nhiên, Redlock chỉ tăng độ sẵn sàng của chính dịch vụ khóa (tránh lỗi sập cụm Redis). Nó **hoàn toàn vô dụng** trước hiện tượng tiến trình bị tạm dừng (GC Pause). Nếu Instance A bị dừng ngay sau khi lấy được Redlock thành công, khóa của nó vẫn hết hạn trên Redis và chuyển giao cho Instance C. Khi A tỉnh dậy, nó vẫn sẽ ghi đè lên dữ liệu của C.
- **DB Pessimistic Lock (C) - Cơn ác mộng nghẽn connection pool:**
  Sử dụng `SELECT ... FOR UPDATE` buộc bạn phải mở một Transaction và giữ khóa dòng trong suốt thời gian cron job chạy (60 giây để lấy dữ liệu, ghi file lên S3, gọi API Slack). Việc này giữ chặt kết nối DB (Connection Pool). Nếu connection pool của ứng dụng có kích thước 50 và bạn chạy vài cron job song song, toàn bộ hệ thống Web API hướng người dùng sẽ bị đóng băng vì không thể lấy được kết nối DB.
- **Optimistic Concurrency (D) - Lãng phí tài nguyên và spam external side effects:**
  Mặc dù DB sẽ từ chối bản ghi cuối cùng của 2 instance chạy sau nhờ ràng buộc Unique Key, nhưng cả 3 instance của bạn vẫn phải chạy song song tác vụ nặng trong 60 giây. Điều này gây lãng phí gấp 3 lần tài nguyên CPU/RAM hệ thống. Quan trọng hơn, nếu tác vụ đó có gọi API ngoài (ví dụ gửi email báo cáo cho đối tác, spam Slack), bạn sẽ gửi đi 3 email trùng lặp cho khách hàng trước khi DB rollback bản ghi ghi nhận.
