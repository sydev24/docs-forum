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

## Nội dung dữ liệu

- `demo.html` là catalog gồm 10 case trình bày: Topic suggestion, Post văn bản, 2 Comment và
  4 Post có ảnh. Ảnh trong `assets/demo-images/` là fixture tổng hợp từ test suite, không phải
  dữ liệu người dùng thật.

- [FEATURES.md](./FEATURES.md) tóm tắt tính năng, luồng hoạt động và công nghệ của ứng dụng.
- `future-features.html` là roadmap đề xuất ngoài MVP; không mô tả tính năng đã hoàn tất.
- Trang `moderation-cases.html` hiển thị sẵn đủ 115 cases trong chính file HTML; chỉ mở một case
  khi cần xem thêm nội dung và rationale.
- `reports/langfuse-live-115.html` tóm tắt lần eval 115 cases: độ phù hợp, độ trễ, chi phí ghi
  nhận và phạm vi diễn giải.
- `moderation.html` giữ summary official trước đó là 113/115. `langfuse-live-115.html` là snapshot
  development/local riêng, tóm tắt 114/115 và không được diễn giải như production SLA.
- Raw actual result theo từng case không được đưa vào artifact; trang cases chỉ ghi canonical input
  và expected label.
- Kết quả archive 107 cases không được dùng như báo cáo hiện hành.
- Dataset là self-reviewed và chưa được tuyên bố mentor-approved.

Khi số liệu nguồn thay đổi, cần tạo lại hoặc sửa nội dung HTML trước khi chia sẻ snapshot mới.
