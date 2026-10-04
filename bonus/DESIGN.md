# Bonus B2 — Thiết kế pipeline cho trợ lý CSKH tiếng Việt

## Bài toán và các ràng buộc thực tế

Tôi chọn thiết kế pipeline dữ liệu cho một trợ lý CSKH tiếng Việt của sàn thương mại điện tử. Trợ lý trả lời các câu hỏi về đơn hàng, đổi trả, thanh toán và lỗi ứng dụng; khi không chắc chắn, nó phải chuyển người dùng sang nhân viên. Người dùng cuối cần câu trả lời trong vài giây, nhưng dữ liệu nguồn lại không đồng nhất: ticket và trạng thái đơn hàng thay đổi liên tục, tài liệu chính sách là PDF/HTML, còn feedback đến từ click, rating và transcript hội thoại. Một câu trả lời sai về hoàn tiền có thể gây tổn thất tiền bạc và mất niềm tin, nên mục tiêu không chỉ là trả lời đúng mà còn là biết từ chối hoặc escalation đúng lúc.

Ràng buộc quan trọng ở Việt Nam là tiếng Việt có dấu, cách viết không chuẩn, xen tiếng Anh và nhiều PII như số điện thoại, địa chỉ, mã đơn hàng. Dữ liệu đơn hàng là dữ liệu vận hành nhạy cảm; không được đưa nguyên văn vào prompt hoặc tập huấn nếu chưa có cơ sở pháp lý và cơ chế xoá. Nhóm CSKH cũng không có khả năng vận hành một nền tảng streaming phức tạp ngay từ ngày đầu, do đó thiết kế cần tăng dần theo quy mô thay vì tối ưu quá sớm.

## Năm quyết định và đánh đổi

### 1. Nguồn dữ liệu và hợp đồng: CDC cho thực thể, Bronze bất biến cho mọi nguồn

Tôi chọn CDC từ PostgreSQL cho ticket, đơn hàng và trạng thái hoàn tiền; các sự kiện click/rating đi qua topic event; tài liệu chính sách được ingest từ kho nội bộ. Mỗi record vào Bronze được lưu bất biến cùng nguồn, batch ingest, offset/LSN và schema version. Silver chuẩn hoá `ticket_id`, `order_id`, thời gian UTC, trạng thái xoá và che PII; dòng sai đi vào quarantine có lý do.

Đánh đổi là chi phí lưu Bronze và phải quản lý nhiều version schema. Tôi chọn X thay vì chỉ lưu bảng Silver cuối cùng (Y), vì CDC redelivery, sửa parser và backfill là tình huống chắc chắn xảy ra. Bronze giúp tái lập một snapshot tại thời điểm cần điều tra. Tuy nhiên, raw PII ở Bronze phải có mã hoá, phân quyền hẹp và TTL riêng; tính bất biến không đồng nghĩa với giữ vô hạn.

### 2. Freshness: streaming hẹp cho trạng thái đơn hàng, microbatch cho tài liệu và phân tích

Tôi chọn luồng gần thời gian thực cho các thay đổi đơn hàng/ticket có SLA dưới năm phút. Consumer ghi Silver keyed bằng `MERGE` và chỉ nhận LSN mới hơn. Gold cho feature routing có microbatch theo event time, lookback được đặt từ P99 lateness đo thực tế. Tài liệu chính sách, embedding, báo cáo chất lượng và dataset huấn luyện chạy batch ban đêm.

X là mô hình lai thay vì streaming toàn bộ (Y). Streaming toàn bộ làm tăng chi phí vận hành, khó replay và tạo áp lực alert 24/7, trong khi tài liệu chính sách không cần xuất hiện sau vài giây. Khi traffic tăng 100 lần, bottleneck đầu tiên dự kiến là small files và embedding API, không phải DuckDB hay SQL. Vì vậy tôi sẽ compact Bronze theo giờ/ngày, dùng queue có backpressure và cache embedding theo hash nội dung + model version.

### 3. Chất lượng và PII: chặn trước Gold, không để model tự quyết định dữ liệu hợp lệ

Tôi chọn data contract cho các trường tối thiểu (`event_id`, thời gian, tenant, loại sự kiện), enum trạng thái, kiểm tra quan hệ ticket–order và giới hạn freshness. PII scanner chạy ở đường Bronze-to-Silver: regex xử lý email/điện thoại, NER và rule theo ngữ cảnh xử lý tên/địa chỉ; kết quả có thể mask, tokenize hoặc quarantine. Dashboard theo dõi tỷ lệ quarantine, schema drift, P95/P99 lateness, PII leak rate trên mẫu audit và thời gian xử lý cảnh báo.

Tôi loại bỏ phương án để LLM tự lọc PII hoặc sửa record xấu. Mô hình khó giải thích, không bảo đảm deterministic và có thể chính là nơi dữ liệu nhạy cảm bị lộ. Chi phí của false positive là mất một ít dữ liệu phân tích; chi phí false negative là rò rỉ PII, nên threshold ban đầu nghiêng về an toàn và được hiệu chỉnh bằng review của đội privacy.

### 4. Phục vụ tri thức: RAG trước, knowledge graph cho quan hệ khó

Tôi chọn vector RAG cho FAQ, chính sách đổi trả và hướng dẫn thao tác vì phần lớn truy vấn là tìm đoạn văn phù hợp. Chunks mang `policy_version`, hiệu lực từ/đến, locale và ACL; retrieval lọc metadata trước rồi mới rerank. Câu trả lời luôn trả citation, và nếu confidence thấp hoặc policy hết hiệu lực thì chuyển agent.

Knowledge graph là phương án bị loại bỏ ở giai đoạn đầu. Nó mạnh cho câu hỏi multi-hop như “đơn bị tách, voucher bị hoàn thế nào sau khi đổi một phần”, nhưng chi phí trích xuất entity/relation, kiểm tra sai quan hệ và vận hành graph cao. Tôi chỉ thêm graph khi log cho thấy tỷ lệ câu hỏi multi-hop/escalation đủ lớn. Khi đó graph dùng cho quan hệ order–shipment–refund, còn RAG vẫn phục vụ văn bản chính sách; không cố biến mọi tài liệu thành graph.

### 5. Flywheel và semantics khi lỗi: feedback có version, side effect tách khỏi replay

Mỗi câu trả lời ghi trace gồm retrieval ids, policy version, prompt/model version, confidence, quyết định escalation và feedback người dùng/nhân viên. Một job batch tạo eval set từ các case có rating thấp, escalation hoặc agent sửa câu trả lời; các cặp preference chỉ dùng dữ liệu đã de-identify và có thời gian cắt rõ ràng để tránh leakage. Dataset train dùng point-in-time join: feature và policy chỉ được phép lấy những gì đã tồn tại tại thời điểm hội thoại.

Mỗi stage có run id và key idempotent. Re-run chỉ ghi đè partition Gold hoặc MERGE theo khoá/LSN; gửi email, hoàn tiền hay gọi CRM là side effect ngoài pipeline và phải đi qua outbox/idempotency key riêng. Khi lỗi model provider, pipeline giữ trace và đưa job vào retry queue với exponential backoff; không tự gửi câu trả lời lần hai. Chi phí được kiểm soát bằng cache prompt/response, batch embedding, giới hạn token và budget theo tenant. Tôi ưu tiên đo “cost per resolved ticket” thay vì chỉ cost per token, vì một câu trả lời rẻ nhưng gây escalation hoặc hoàn tiền sai không thực sự rẻ.

## Sơ đồ kiến trúc

```text
Postgres tickets/orders --CDC--> Bronze immutable ----> Silver keyed + PII gate
Click/rating events -----------> Bronze immutable ----> Silver events
Policy PDF/HTML ---------------> Bronze documents ---> parsed/chunked docs
                                                    |          |
                                                    v          v
                                          Gold feature/eval   Vector RAG index
                                                    |          |
Customer chat --> API --> retrieve policy + live order state -> LLM -> answer/escalate
                                   |                              |
                                   +--------- trace + feedback ---+
                                                      |
                                                      v
                                           versioned eval/train datasets
```

Thiết kế này ưu tiên khả năng giải thích, replay và quyền riêng tư trước khi tối ưu latency cực thấp. Các quyết định streaming, graph và model lớn hơn chỉ được mở rộng khi số đo vận hành chứng minh chúng cần thiết.
