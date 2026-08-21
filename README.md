# VF Forum — báo cáo HTML tĩnh

Đây là bộ trang HTML đã chứa sẵn toàn bộ nội dung báo cáo. Không cần backend, JavaScript, API,
JSON runtime hoặc HTTP server.

## Cách xem

Mở trực tiếp file `docs/artifact/index.html` bằng trình duyệt. Từ trang chủ có thể đi đến các trang báo
cáo hiện hành bằng menu.

```text
docs/artifact/
├── index.html
├── product.html
├── architecture.html
├── future-features.html
├── reports/
│   ├── moderation.html
│   ├── moderation-cases.html
│   ├── langfuse-live-115.html
│   ├── automated-tests.html
│   └── e2e-quality.html
└── assets/
    ├── styles.css
    └── demo-images/          # 4 ảnh fixture tổng hợp cho Demo cases
```

## Trang lưu trữ không liên kết từ menu

- `progress.html`
- `reports/topic-suggestion.html`
- `slides.html`
- `future-features.html` — roadmap ngoài MVP, đang tạm ẩn khỏi điều hướng

## Nội dung dữ liệu

- `demo.html` là catalog gồm 10 case trình bày: Topic suggestion, Post văn bản, 2 Comment và
  4 Post có ảnh. Ảnh trong `assets/demo-images/` là fixture tổng hợp từ test suite, không phải
  dữ liệu người dùng thật.

- [FEATURES.md](./FEATURES.md) tóm tắt tính năng, luồng hoạt động và công nghệ của ứng dụng.
- `future-features.html` là roadmap đề xuất ngoài MVP; không mô tả tính năng đã hoàn tất và đang
  tạm ẩn khỏi điều hướng.
- Trang `moderation-cases.html` hiển thị sẵn đủ 115 cases trong chính file HTML; chỉ mở một case
  khi cần xem thêm nội dung và rationale.
- `reports/langfuse-live-115.html` tóm tắt lần eval 115 cases: độ phù hợp, độ trễ, chi phí ghi
  nhận và phạm vi diễn giải.
- `moderation.html` trình bày **cổng chính thức lịch sử 12/08: 113/115 (98,3%)**. Một case chỉ
  đúng khi khớp quyết định, mã quy tắc và nguồn nội dung. Con số **114/115 (99,13%)** chỉ là mức
  khớp quyết định, dùng làm chú thích và không thay thế cổng chính thức.
- `langfuse-live-115.html` là snapshot diagnostic/local riêng ngày 14/08 (concurrency 8). Nó không
  thay thế evidence official ngày 12/08 (concurrency 2), cũng không được diễn giải như production SLA.
- Raw actual result theo từng case không được đưa vào artifact; trang cases chỉ ghi canonical input
  và expected label.
- Kết quả archive 107 cases không được dùng như báo cáo hiện hành.
- Dataset là self-reviewed và chưa được tuyên bố mentor-approved.

## Trạng thái đồng bộ

- Nội dung sản phẩm và số liệu fixture đã được đối chiếu với source ngày 21/08/2026: 7 Topic,
  36 tài khoản, 40 Post, 100 Comment, 181 moderation log và 17 ảnh fixture trên 13 Post.
- Unit + integration, coverage, typecheck, lint, format và contract lint đã được chạy lại ngày
  **21/08/2026**: 84/84 files, 944/944 tests; Statements 95,40%, Branches 92,06%, Functions 96,39%,
  Lines 96,19%. Chromium E2E, Prisma validation, production build và live AI vẫn là checkpoint lịch
  sử, chưa được chạy lại sau Comment refinement.
- T169 vẫn mở để xác minh lại Comment; vì vậy SC-002 và SC-010 đang cần evidence mới. T124 Langfuse
  live smoke cũng vẫn mở vì cần credentials thật.
- Các trang archive vẫn giữ số liệu lịch sử và phải ghi rõ mốc checkpoint; không diễn giải chúng là
  kết quả mới nhất.

Khi số liệu nguồn thay đổi, cần tạo lại hoặc sửa nội dung HTML trước khi chia sẻ snapshot mới.
