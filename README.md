PHÂN TÍCH TRADE-OFF VÀ ĐẶC TẢ SRS

Tích hợp thanh toán và đối soát COD – RikkeiLogistics

Phiên bản: 1.0
Ngày: 24/09/2026
Phạm vi: Thanh toán cước vận chuyển và đối soát COD qua kênh số
Quy mô: Trên 200.000 giao dịch/ngày

1. Mục tiêu

Hệ thống RikkeiLogistics cần tích hợp Cổng thanh toán/ngân hàng để:

Cho phép khách hàng thanh toán cước vận chuyển bằng kênh số.

Nhận kết quả thanh toán từ ngân hàng/cổng thanh toán.

Đảm bảo trạng thái PAID chỉ được ghi nhận khi chữ ký HMAC-SHA256 hợp lệ.

Xử lý webhook trùng bằng Idempotency Key.

Đối soát COD tự động sau khi shipper nộp tiền và chuyển tiền vào ví người gửi trong vòng 24 giờ.

Đảm bảo giao dịch tài chính có tính ACID và có Audit Trail Log.

Hoạt động ổn định khi mạng nghẽn, timeout hoặc webhook được gửi lại.

2. Bối cảnh và giả định

Khối lượng hơn 200.000 giao dịch/ngày khiến hệ thống cần ưu tiên khả năng chịu tải, khả năng retry và tính nhất quán dữ liệu. Hai phương án được xem xét:

Giải pháp 1 – Sync API: RikkeiLogistics gọi API thanh toán và chờ phản hồi đồng bộ.

Giải pháp 2 – Async Webhook/Queue: Ngân hàng/cổng thanh toán gửi webhook; Message Queue tiếp nhận và Billing Service xử lý bất đồng bộ.

PHẦN 1 – PHÂN TÍCH TRADE-OFF

3. Cơ chế hai giải pháp

3.1. Giải pháp 1 – Sync API

Luồng cơ bản:

RikkeiLogistics → Payment Gateway/Bank → phản hồi trực tiếp → RikkeiLogistics

Ứng dụng gửi yêu cầu thanh toán và giữ request trong thời gian chờ kết quả. Khi nhận phản hồi thành công, hệ thống xác thực dữ liệu, ghi nhận giao dịch và cập nhật trạng thái.

Ưu điểm

Kiến trúc đơn giản, ít thành phần.

Dễ triển khai và dễ debug ở giai đoạn đầu.

Người dùng có thể nhận kết quả gần như ngay lập tức.

Hạn chế

Timeout/nghẽn mạng tạo ra trạng thái không rõ ràng: ngân hàng có thể đã trừ tiền nhưng ứng dụng chưa nhận được phản hồi.

Request đồng bộ giữ kết nối trong thời gian xử lý.

Khi lưu lượng lớn, việc xử lý trực tiếp làm tăng áp lực lên API và khó hấp thụ các đợt tăng tải.

Cần cơ chế reconciliation/retry riêng để xử lý giao dịch “đã thanh toán nhưng chưa cập nhật”.

3.2. Giải pháp 2 – Async Webhook/Queue

Luồng cơ bản:

Ngân hàng → Webhook → Message Queue → Consumer/Billing Service → cập nhật Payment/Order

Webhook được tiếp nhận nhanh và đưa vào Queue. Worker xử lý độc lập, xác thực HMAC-SHA256, kiểm tra idempotency, thực hiện transaction ACID, cập nhật trạng thái đơn hàng và ghi AuditLog.

Ưu điểm

Không phụ thuộc vào việc giữ kết nối HTTP trong suốt quá trình xử lý.

Queue hấp thụ lưu lượng cao và hỗ trợ retry khi Billing Service tạm thời lỗi.

Dễ mở rộng số lượng consumer theo tải.

Phù hợp với mô hình webhook có thể gửi lại.

Dễ thiết kế idempotency để tránh ghi nhận cùng một giao dịch nhiều lần.

Hạn chế

Kiến trúc phức tạp hơn: phải vận hành Queue, Worker, retry/dead-letter và monitoring.

Kết quả cập nhật có thể có độ trễ nhỏ thay vì đồng bộ ngay lập tức.

Cần thiết kế kỹ cơ chế duplicate, ordering và reconciliation.

4. Bảng Trade-off

Tiêu chí

Giải pháp 1 – Sync API

Giải pháp 2 – Async Webhook/Queue

Độ tin cậy khi nghẽn mạng

Thấp hơn: timeout có thể xảy ra sau khi ngân hàng đã xử lý; cần reconciliation để xác định kết quả thật

Cao hơn: webhook được đưa vào Queue, có thể retry khi consumer hoặc mạng gặp lỗi

Độ phức tạp triển khai

Thấp: ít thành phần, luồng dễ hiểu

Cao: Queue, Worker, retry, DLQ, monitoring, idempotency

Rủi ro sai lệch dữ liệu tài chính

Có rủi ro nếu xử lý timeout không có idempotency/reconciliation tốt

Kiểm soát tốt hơn nếu kết hợp HMAC + idempotency + ACID + AuditLog

Khả năng hấp thụ tải

Phụ thuộc trực tiếp vào API và thời gian phản hồi

Tốt hơn nhờ Queue làm lớp đệm và scale consumer

Khả năng retry

Phải tự xây dựng ở tầng API/client

Tự nhiên hơn với Queue/Worker và retry policy

Khả năng mở rộng

Khó hơn khi lưu lượng tăng mạnh

Dễ scale ngang Consumer/Worker

5. Lựa chọn kiến trúc

Đề xuất: Giải pháp 2 – Async Webhook/Queue.

Lý do kỹ thuật:

Khối lượng trên 200.000 giao dịch/ngày cần một lớp đệm để hấp thụ tải.

Webhook + Queue giảm sự phụ thuộc vào một kết nối HTTP kéo dài.

Queue hỗ trợ retry đối với lỗi tạm thời và giúp xử lý webhook gửi lại.

HMAC-SHA256 đảm bảo chỉ kết quả thanh toán có chữ ký hợp lệ mới được phép đi tiếp.

Idempotency Key giúp một giao dịch được xử lý đúng một lần về mặt nghiệp vụ.

Transaction ACID bảo đảm cập nhật tài chính nhất quán.

AuditLog cung cấp dấu vết để truy vết và đối soát khi có tranh chấp.

Kiến trúc có thể mở rộng số lượng Consumer khi tải tăng.

Lưu ý: Async không tự động loại bỏ mọi rủi ro sai lệch tài chính. Cần kết hợp HMAC, idempotency, transaction ACID, retry có kiểm soát và reconciliation.

PHẦN 2 – SEQUENCE DIAGRAM

6. Luồng xử lý hoàn chỉnh

6.1. Luồng chính

Khách hàng bấm Thanh toán trên RikkeiLogistics App.

App tạo/hiển thị mã VietQR cho giao dịch.

Khách hàng quét mã và chuyển khoản.

Ngân hàng gửi Async Webhook chứa kết quả thanh toán và chữ ký HMAC-SHA256.

Message Queue tiếp nhận webhook.

Consumer Worker lấy message và chuyển cho Billing Service.

Billing Service:

kiểm tra Idempotency Key;

xác thực HMAC-SHA256;

kiểm tra số tiền, mã giao dịch và trạng thái;

nếu hợp lệ thì thực hiện transaction ACID.

Billing Service gạch nợ/cập nhật PaymentTransaction.

Sau khi transaction thành công, Billing Service cập nhật trạng thái đơn hàng thành PAID.

Ghi AuditLog và ACK message.

6.2. Nhánh lỗi

HMAC không hợp lệ

Không chuyển trạng thái PAID.

Không gạch nợ.

Ghi AuditLog với kết quả xác thực thất bại.

Trả/reject message theo chính sách bảo mật.

Webhook trùng

So sánh Idempotency Key.

Nếu giao dịch đã xử lý thành công, không thực hiện nghiệp vụ tài chính lần hai.

Có thể ACK message trùng để tránh retry vô hạn.

Timeout/lỗi tạm thời

Không đánh dấu giao dịch thành công nếu transaction chưa commit.

Queue thực hiện retry theo backoff.

Sau số lần retry tối đa, đưa vào DLQ để xử lý thủ công/reconciliation.

Tài khoản nhận COD bị khóa/sai

Không chuyển tiền vào ví.

Ghi nhận trạng thái NEEDS_REVIEW.

Tạo AuditLog và đưa vào quy trình xử lý ngoại lệ.

7. Hai bước bổ sung bắt buộc

Bước 1 – Xác thực HMAC

Đặt trước thao tác gạch nợ:

Billing Service → Verify HMAC-SHA256 → Hợp lệ?

Chỉ khi hợp lệ mới cho phép xử lý tài chính.

Bước 2 – Cập nhật trạng thái đơn hàng

Đặt sau khi transaction gạch nợ commit thành công:

Billing Service → Order Service → cập nhật Order/Shipment = PAID

Không cập nhật PAID trước khi giao dịch tài chính được commit thành công.

8. File sơ đồ draw.io

Sơ đồ đã được thiết kế theo UML Sequence Diagram, gồm:

Khách hàng

RikkeiLogistics App

Ngân hàng

Message Queue

Billing Service

Order Service

HMAC-SHA256

Idempotency

Transaction ACID

Cập nhật PAID

Retry

File editable: RikkeiLogistics_Sequence_Thanh_toan_COD.drawio

PHẦN 3 – ĐẶC TẢ SRS

9. Tổng quan

9.1. Mục đích

Tài liệu SRS mô tả yêu cầu cho module thanh toán cước và đối soát COD của RikkeiLogistics, tập trung vào tính đúng đắn của giao dịch tài chính, bảo mật, khả năng chịu tải và khả năng phục hồi khi có lỗi mạng.

9.2. Phạm vi

Trong phạm vi

Tạo yêu cầu thanh toán.

Sinh/hiển thị VietQR.

Nhận webhook.

Xác thực HMAC-SHA256.

Idempotency.

Ghi nhận PaymentTransaction.

Gạch nợ.

Cập nhật trạng thái đơn hàng.

Đối soát COD.

Chuyển tiền vào ví người gửi.

Audit Trail.

Retry và xử lý ngoại lệ.

Ngoài phạm vi

Core Banking của ngân hàng.

Hệ thống nội bộ của nhà cung cấp cổng thanh toán.

Quy trình giao hàng vật lý của shipper, ngoại trừ dữ liệu COD cần cho đối soát.

10. SRS – Mục 3.1: Yêu cầu chức năng

ID

Yêu cầu

Mô tả

FR-01

Tạo thanh toán

Hệ thống phải tạo PaymentTransaction cho vận đơn đã tính cước.

FR-02

VietQR

Hệ thống phải cung cấp thông tin cần thiết để khách hàng thực hiện chuyển khoản qua VietQR.

FR-03

Nhận webhook

Hệ thống phải tiếp nhận kết quả thanh toán bất đồng bộ từ ngân hàng/cổng thanh toán.

FR-04

HMAC

Hệ thống phải xác thực HMAC-SHA256 trước mọi thao tác làm thay đổi trạng thái tài chính.

FR-05

Idempotency

Hệ thống phải phát hiện webhook trùng dựa trên Idempotency Key/Transaction ID.

FR-06

Gạch nợ

Chỉ giao dịch hợp lệ mới được gạch nợ trong transaction ACID.

FR-07

PAID

Chỉ chuyển trạng thái đơn hàng sang PAID sau khi transaction tài chính commit thành công.

FR-08

Audit

Hệ thống phải ghi AuditLog cho các sự kiện tài chính và sự kiện bảo mật quan trọng.

FR-09

Retry

Hệ thống phải retry lỗi tạm thời theo chính sách backoff và giới hạn số lần retry.

FR-10

COD Settlement

Hệ thống phải đối soát COD tự động và hoàn tất chuyển tiền vào ví người gửi trong tối đa 24 giờ khi dữ liệu hợp lệ.

FR-11

Ngoại lệ COD

Nếu ví nhận bị khóa hoặc sai thông tin, hệ thống phải giữ tiền ở trạng thái chờ xử lý và tạo log/audit.

FR-12

Reconciliation

Hệ thống phải hỗ trợ đối soát các giao dịch có trạng thái không xác định hoặc chênh lệch.

11. SRS – Mục 3.4: Yêu cầu phi chức năng và bảo mật

11.1. Hiệu năng

Hệ thống phải hỗ trợ quy mô trên 200.000 giao dịch/ngày.

Webhook phải được tiếp nhận nhanh và đưa vào Queue thay vì phụ thuộc vào thời gian xử lý của Billing Service.

Consumer có thể scale ngang.

Các thao tác DB quan trọng phải có transaction rõ ràng.

11.2. Độ tin cậy

Message phải không bị mất khi Consumer tạm thời unavailable.

Retry phải có giới hạn và backoff.

Message lỗi nhiều lần phải có cơ chế DLQ hoặc trạng thái chờ xử lý.

Giao dịch đã commit không được thực hiện lại nghiệp vụ tài chính do webhook trùng.

11.3. Tính nhất quán tài chính

Một transaction tài chính phải bảo đảm:

Validate → Idempotency Check → Debit/Record → Commit → Update Order

Nếu transaction rollback, hệ thống không được coi giao dịch là PAID.

11.4. Bảo mật

API Key

Dùng API Key để nhận diện/ủy quyền các request server-to-server theo cơ chế của đối tác.

Không hard-code API Key trong source code.

Lưu secret trong Secret Manager hoặc biến môi trường.

Không ghi API Key đầy đủ vào log.

HMAC-SHA256

Giả sử:

signature = HMAC-SHA256(secret, canonical_payload)

Hệ thống nhận payload và chữ ký từ đối tác, sau đó tính lại chữ ký bằng secret đã lưu.

Nếu chữ ký khớp → tiếp tục xử lý.

Nếu không khớp → reject, không gạch nợ, không chuyển PAID, ghi AuditLog.

Secret HMAC phải được bảo vệ và có quy trình rotation.

11.5. Audit Trail

AuditLog nên lưu tối thiểu:

event_id

transaction_id

actor/source

event_type

old_status

new_status

timestamp

result

request_id/correlation_id

reason/error_code

Không ghi dữ liệu nhạy cảm không cần thiết như secret/API Key.

12. Thiết kế 4 bảng dữ liệu

12.1. Shipment

Lưu thông tin vận đơn.

Khóa chính: shipment_id

Thuộc tính gợi ý:

shipment_id

sender_id

receiver_id

shipping_fee

cod_amount

status

created_at

updated_at

12.2. PaymentTransaction

Lưu giao dịch thanh toán cước.

Khóa chính: payment_transaction_id

Thuộc tính gợi ý:

payment_transaction_id

shipment_id

gateway_transaction_id

idempotency_key

amount

status

paid_at

created_at

updated_at

Ràng buộc quan trọng: idempotency_key phải có unique constraint theo phạm vi nghiệp vụ phù hợp.

12.3. CODSettlement

Lưu quá trình đối soát và chuyển COD.

Khóa chính: settlement_id

Thuộc tính gợi ý:

settlement_id

shipment_id

cod_amount

recipient_wallet_id

status

settlement_at

failure_reason

created_at

updated_at

Các trạng thái có thể gồm:

PENDING → RECONCILED → TRANSFERRED

hoặc

PENDING → NEEDS_REVIEW

12.4. AuditLog

Lưu dấu vết kiểm toán.

Khóa chính: audit_id

Thuộc tính gợi ý:

audit_id

transaction_id

event_type

actor/source

old_status

new_status

result

request_id

created_at

13. Quy tắc trạng thái

PaymentTransaction

PENDING → VERIFIED → PAID

Nhánh lỗi:

PENDING → FAILED

hoặc:

PENDING → NEEDS_REVIEW

Không cho phép:

UNVERIFIED → PAID

CODSettlement

PENDING → RECONCILED → TRANSFERRED

Nếu tài khoản nhận bị khóa/sai:

PENDING → NEEDS_REVIEW

14. Tiêu chí nghiệm thu

ID

Tiêu chí

AC-01

HMAC sai không thể chuyển giao dịch sang PAID.

AC-02

Webhook trùng không tạo thêm một lần gạch nợ.

AC-03

Timeout phía RikkeiLogistics không làm mất khả năng nhận và xử lý lại webhook.

AC-04

Transaction rollback thì trạng thái tài chính không được ghi nhận là thành công.

AC-05

Sau khi debit commit thành công, Order/Shipment được cập nhật PAID.

AC-06

Tài khoản COD bị khóa phải chuyển sang NEEDS_REVIEW, không chuyển tiền tự động.

AC-07

Mọi sự kiện tài chính quan trọng có AuditLog.

AC-08

Có thể truy vết một giao dịch bằng transaction_id/request_id.

15. Kết luận

Giải pháp Async Webhook/Queue phù hợp với bài toán có hơn 200.000 giao dịch/ngày vì cho phép tách tiếp nhận sự kiện khỏi xử lý nghiệp vụ, hỗ trợ retry và scale consumer.

Tuy nhiên, độ tin cậy tài chính không đến từ Queue đơn lẻ. Kiến trúc phải kết hợp đồng thời:

HMAC-SHA256 + Idempotency + ACID Transaction + Audit Trail + Retry/DLQ + Reconciliation

Đặc biệt, trạng thái PAID chỉ được ghi nhận sau khi chữ ký HMAC hợp lệ và transaction tài chính đã commit thành công.
