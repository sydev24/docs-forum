# VinFast EV Community Forum — tính năng, luồng và công nghệ

Tài liệu này tóm tắt các tính năng đang có trong prototype. Ứng dụng là một codebase full-stack
Next.js; giao diện và API cùng chạy trên server Node.js.

## Bức tranh tổng quát

```text
Người dùng (VI/EN)
  → Next.js App Router + React UI
  → Route Handlers / API
  → PostgreSQL (Prisma) + Redis
  → AI moderation / Trending (OpenAI, khi được cấu hình)
  → Moderator hoặc Admin xử lý nghiệp vụ
```

| Thành phần      | Công nghệ sử dụng                                           | Vai trò                                                    |
| --------------- | ----------------------------------------------------------- | ---------------------------------------------------------- |
| Web/UI          | Next.js 16 App Router, React 19, TypeScript, Tailwind CSS 4 | Render trang, form, trang theo role                        |
| Đa ngôn ngữ     | `next-intl`                                                 | Giao diện tiếng Việt và tiếng Anh                          |
| API             | Next.js Route Handlers, Zod                                 | Endpoint, validate request và trả DTO an toàn              |
| Cơ sở dữ liệu   | PostgreSQL 16, Prisma 7                                     | User, Topic, Post, Comment, moderation log                 |
| Phiên đăng nhập | bcryptjs, `jose` JWT HS256, cookie `httpOnly`               | Hash mật khẩu, xác thực và phân quyền                      |
| Quota           | Redis 7, ioredis, Lua script                                | Giới hạn Post theo ngày theo thao tác atomic               |
| AI              | OpenAI moderation + structured output + embedding           | Moderation, gợi ý Topic và gom cụm Trending                |
| Quan sát        | Langfuse, OpenTelemetry (tùy chọn)                          | Trace metadata của moderation, không là điều kiện app chạy |
| Kiểm thử        | Vitest, Playwright, Redocly, Prisma                         | Unit, integration, E2E, contract và seed checks            |

## 1. Đăng ký, đăng nhập và phân quyền

**Mục đích:** quản lý phiên làm việc và giới hạn chức năng theo vai trò `Customer`, `Expert`,
`Moderator`, `Admin`.

```text
Đăng ký/đăng nhập
  → validate dữ liệu
  → bcrypt hash hoặc verify password
  → ký JWT thời hạn 24 giờ
  → ghi cookie session httpOnly
  → middleware/route kiểm tra session và role
  → mở đúng giao diện, API và dữ liệu được phép xem
```

- Customer và Expert có thể tạo Post và Comment tại Post công khai; cả hai đều đi qua moderation.
- Moderator xử lý hàng chờ kiểm duyệt; Admin quản trị và xem audit. Route đổi role đã được triển
  khai, nhưng không thuộc baseline acceptance vì User Directory trong đặc tả được mô tả chỉ-đọc;
  cần quyết định scope trước khi công bố như tính năng chính thức.
- Nếu scope đổi role được phê duyệt: Admin không thể tự đổi role và không thể hạ cấp Admin cuối
  cùng. Role mới được đọc lại từ DB ở request kế tiếp, nên có hiệu lực ngay sau khi cập nhật thành
  công.
- API không trả password hash, token hoặc email qua DTO công khai.
- Công nghệ chính: `lib/auth/*`, Prisma `User`, JWT `jose`, bcryptjs, Route Handlers
  `/api/auth/*`.

## 2. Chủ đề và duyệt nội dung công khai

**Mục đích:** tổ chức thảo luận theo Topic và chỉ hiển thị nội dung đã được public.

```text
Admin tạo/sửa/ẩn Topic
  → Prisma lưu Topic active/hidden
  → UI chỉ cho chọn Topic active khi tạo Post
  → Post/Comment public được truy vấn theo Topic
  → DTO lọc trường nhạy cảm trước khi trả UI
```

- Có trang danh sách Topic, Post theo Topic và Post detail kèm Comment.
- Topic không bị xóa vĩnh viễn trong luồng quản trị; Admin dùng trạng thái ẩn/hiện.
- Công nghệ chính: Prisma `Topic`, `lib/topics/*`, `lib/content/dto.ts`, API `/api/topics/*`.

## 3. Tạo Post, ảnh đính kèm và quota 5 Post/ngày

**Mục đích:** cho phép Customer/Expert đăng bài an toàn, có giới hạn chống spam.

```text
Form Post + tối đa 3 ảnh
  → Zod/kiểm tra kích thước, định dạng ảnh
  → Redis Lua reserve quota theo user + ngày Asia/Ho_Chi_Minh
  → lưu Post trạng thái processing và stage ảnh
  → gọi moderation ngoài DB transaction
  → commit trạng thái cuối + audit log bằng transaction/CAS
  → đánh dấu quota đã commit hoặc xử lý lỗi theo nghiệp vụ
```

- Mỗi ảnh tối đa 5 MB; tối đa 3 ảnh và tổng 15 MB. Ảnh được kiểm tra chữ ký file, không chỉ dựa
  vào extension.
- Quota là tối đa 5 Post đã tiếp nhận mỗi ngày; Redis dùng lệnh Lua atomic để tránh hai request
  đồng thời vượt quota.
- Tác giả có trang theo dõi trạng thái Post và có thể retry khi moderation thất bại.
- Ảnh bị reject được gỡ reference trong transaction, sau đó event listener dọn file vật lý.
- Công nghệ chính: `app/api/posts/route.ts`, `lib/posts/daily-limit.ts`,
  `lib/images/storage.ts`, Redis/ioredis, Prisma transaction và event bus nội bộ.

## 4. Bình luận và trạng thái kiểm duyệt

**Mục đích:** cho phép thảo luận trên Post public, vẫn đi qua chính sách nội dung.

```text
Customer hoặc Expert gửi Comment vào Post public
  → bắt buộc Idempotency-Key, validate request và kiểm tra hạn chế bị từ chối trong ngày
  → lưu/replay Comment processing một cách idempotent
  → Layer 1 safety và Layer 2 community policy
  ├─ approve → published
  ├─ reject → rejected
  ├─ boundary case → pending_moderator_review
  └─ lỗi/timeout → moderation_failed, tác giả có thể retry
```

- Chỉ Comment `published` xuất hiện trong luồng public.
- Cùng một `Idempotency-Key` chỉ tạo một Comment; retry sau khi mất phản hồi trả lại kết quả trước đó.
  Dùng lại key với Post hoặc nội dung khác bị từ chối để tránh ghi nhầm.
- Tác giả xem trạng thái qua endpoint không cache (`private, no-store`) và retry Comment lỗi qua
  endpoint riêng.
- Công nghệ chính: API `/api/posts/[postId]/comments`, `/api/comments/*`, Prisma `Comment`,
  shared moderation pipeline.

## 5. AI moderation hai lớp và audit log

**Mục đích:** chặn nội dung không an toàn trước, sau đó áp quy tắc cộng đồng và giữ dấu vết quyết
định.

```text
Post/Comment processing
  → Layer 1: OpenAI moderation cho text và từng ảnh
  ├─ bị flag → rejected
  └─ pass
       → Layer 2: structured output áp CR-001…CR-006
          ├─ Post approve → pending_moderator_review
          ├─ Comment approve → published
          ├─ reject → rejected
          └─ pending → pending_moderator_review
  → atomic commit content + moderation log
```

- OpenAI luôn chạy ngoài database transaction để không giữ lock trong lúc chờ mạng.
- State machine chỉ cho phép chuyển trạng thái hợp lệ; conditional update (CAS) chặn kết quả AI đến
  muộn ghi đè quyết định mới.
- Timeout/schema error đưa nội dung về `moderation_failed`, không ghi audit log giả.
- Audit log phân biệt quyết định AI và người duyệt; giao diện Admin thấy attribution Moderator mà
  không cần lộ email/hash.
- Công nghệ chính: `lib/moderation/*`, OpenAI SDK, Zod structured output, Prisma, state machine,
  `ModerationLog`.

## 6. Moderator queue và human review

**Mục đích:** đưa các Post cần con người duyệt trước khi public và xử lý các Comment boundary.

```text
Nội dung pending_moderator_review
  → Moderator mở queue theo quyền
  → xem ngữ cảnh, verdict/rule và ảnh (nếu có)
  → approve hoặc reject
  → CAS kiểm tra trạng thái hiện tại
  → cập nhật content + human moderation log
  → nếu approve Post: phát event published và hiển thị public
```

- Post AI approve vẫn cần Moderator approve mới public.
- Quyết định human được ghi với danh tính Moderator, bảo vệ khỏi double-submit/kết quả muộn.
- Công nghệ chính: `/api/moderation/queue/*`, `app/[locale]/moderator/*`, Prisma transaction,
  RBAC và event bus.

## 7. Gợi ý/chỉnh Topic sau moderation

**Mục đích:** hỗ trợ tác giả sửa Topic khi Post thuộc trường hợp community rule CR-006, nhưng hệ
thống không tự đổi Topic.

```text
Post CR-006 đủ điều kiện
  → gọi optional topic suggestion tối đa 8 giây
  → trả Topic active hợp lệ hoặc không có gợi ý
  → tác giả giữ Topic / chọn Topic được gợi ý / yêu cầu review
  → Post trở về Moderator queue
  → chỉ Moderator approve mới public
```

- Có tối đa 2 lượt correction; timeout/lỗi/gợi ý không hợp lệ đều fail-open về Moderator queue.
- Giai đoạn suggestion không làm thay đổi verdict moderation chính.
- Công nghệ chính: OpenAI structured output, `lib/topics/suggestion.ts`,
  `post-moderation-correction.ts`, endpoint `/api/posts/[postId]/topic-correction`.

## 8. Dashboard Trending 24 giờ và 7 ngày

**Mục đích:** cho Admin xem các nhóm thảo luận nổi bật, không phải bảng đếm đơn thuần.

```text
Admin chọn cửa sổ 24h hoặc 7d
  → lấy Post/Comment published trong cửa sổ
  → chuẩn hóa text và tạo embedding theo batch
  → ml-kmeans gom cụm
  → xếp hạng cụm theo tín hiệu hoạt động
  → OpenAI đặt tên cụm hoặc keyword fallback
  → cache in-memory 5 phút
  → trả top cụm cho Dashboard
```

- Chỉ nội dung public được dùng làm nguồn Trending.
- Cache bounded giúp giảm gọi embedding/model khi Admin đổi trang hoặc refresh liên tục.
- Khi provider không sẵn sàng, tên cụm dùng keyword fallback thay vì bịa kết quả AI.
- Snapshot top cụm theo ngày/tuần/tháng trong PostgreSQL chưa có ở bản hiện tại; đây là hạng mục
  roadmap, không phải cơ chế cache đang chạy.
- Công nghệ chính: `lib/trending/*`, OpenAI `text-embedding-3-small`, `ml-kmeans`, cache in-memory,
  API `/api/trending`.

## 9. Admin console

**Mục đích:** vận hành dữ liệu và kiểm tra quyết định moderation trong một khu vực RBAC riêng.

```text
Admin đăng nhập
  → Admin layout kiểm tra role
  → Dashboard: số liệu và Trending
  → Users: tìm kiếm/xem hồ sơ an toàn (đổi role đang chờ quyết định scope)
  → Content: lọc Post/Comment theo trạng thái
  → Moderation logs: xem lịch sử AI/human
  → Topics: tạo, sửa, ẩn/hiện
```

- Có các trang Users, Content Explorer, Moderation Log, Topic Manager và Trending dashboard.
- API Admin áp phân quyền server-side, không chỉ ẩn nút trên UI.
- Công nghệ chính: `app/[locale]/admin/*`, `/api/admin/*`, `/api/moderation/logs`, Prisma, RBAC.

## 10. Vận hành, health check, retention và observability

**Mục đích:** giúp demo/deployment nhận biết phụ thuộc còn hoạt động và dọn dữ liệu tạm an toàn.

```text
GET /api/health
  → Prisma SELECT 1 song song Redis PING
  → 200 { status: ok } hoặc 503 { status: unavailable }

Scheduler
  → quét content processing quá timeout
  → chuyển moderation_failed bằng CAS
  → dọn orphan image theo retention policy
```

- Health endpoint không cache và yêu cầu đồng thời PostgreSQL + Redis sẵn sàng.
- Langfuse là tùy chọn: chỉ trace metadata đã được sanitize; không phải phụ thuộc bắt buộc để app
  chạy hoặc test deterministic.
- Deployment demo dùng Docker Compose, PostgreSQL, Redis, volume ảnh và reverse proxy HTTPS.
- Công nghệ chính: `app/api/health/route.ts`, `lib/retention/*`, `lib/observability/langfuse/*`,
  Docker Compose, Caddy trong môi trường demo.

## 11. Trang Quy tắc cộng đồng

**Mục đích:** công khai các chuẩn hành vi ngắn gọn để thành viên hiểu trước khi đăng hoặc bình luận.

```text
Người dùng mở /[locale]/community-rules
  → đọc 6 quy tắc tóm tắt và phạm vi áp dụng
  → hiểu nội dung vi phạm có thể bị từ chối hoặc chuyển Moderator xem xét
```

- Trang hỗ trợ VI/EN, có lối tắt từ trang chủ và footer.
- Nội dung là bản tóm tắt dễ đọc của quy tắc CR-001…CR-006; không thay thế policy/moderation
  server-side.
- Công nghệ chính: `app/[locale]/(public)/community-rules/page.tsx`, `messages/*.json`,
  `specs/001-vinfast-community-forum/community-rules.md`.

## 12. Seed demo và quality gates

**Mục đích:** tạo dữ liệu demo có thể lặp lại và chứng minh chức năng không phụ thuộc hoàn toàn vào
provider AI live.

```text
SEED_DEMO_PASSWORD có mặt
  → Prisma migrations
  → kiểm inventory/checksum ảnh
  → seed idempotent có guard chống ghi đè dữ liệu phát sinh
  → 7 Topic, 36 account, 40 Post, 100 Comment, 181 audit log, 17 ảnh trên 13 Post
  → test unit/integration/E2E/contract/fixture checks
```

- Mỗi nội dung seed xuất hiện một lần; Post gồm 33 VI/4 EN/3 mixed và phủ các trạng thái moderation.
- Seed dừng khi phát hiện fixture đã có tương tác thật hoặc namespace bị chiếm, thay vì ép ghi đè.
- Có E2E provider-free để chạy không cần OpenAI/Langfuse; live eval là luồng kiểm tra tách biệt.
- Công nghệ chính: Prisma seed, Vitest, Playwright, Redocly, fixture ảnh `sharp`.

## Liên kết tham khảo trong repository

- [Tổng quan báo cáo HTML](./README.md)
- [Kiến trúc moderation](../../lib/moderation/README.md)
- [Hướng dẫn kiểm thử](../../tests/README.md)
- [Kế hoạch kỹ thuật](../../specs/001-vinfast-community-forum/plan.md)

> Phạm vi: đây là prototype/demo. Lưu trữ ảnh trong deployment hiện dùng volume local; cần object
> storage và durable queue nếu mở rộng thành production workload.
