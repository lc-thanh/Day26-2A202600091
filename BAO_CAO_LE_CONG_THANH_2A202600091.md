# BÁO CÁO THỰC HÀNH LAB
## Xây dựng hệ thống Multi-Agent với A2A Protocol

**Họ và tên:** Lê Công Thành  
**Mã sinh viên:** 2A202600091  

---

## 1. Giới thiệu

Bài lab này hướng dẫn xây dựng một hệ thống tư vấn pháp lý sử dụng nhiều AI agent phối hợp với nhau. Nội dung được chia theo các stage, bắt đầu từ cách gọi LLM đơn giản, sau đó thêm tools, xây dựng ReAct agent, mở rộng thành multi-agent và cuối cùng là hệ thống phân tán dùng A2A Protocol.

Trong quá trình làm, em tập trung hoàn thành các yêu cầu chính của lab. Các phần đã làm gồm: thêm cấu hình cho LLM, bổ sung knowledge base, tạo tool mới, thêm tool tra cứu án lệ, xây dựng privacy agent, chỉnh routing và thử chạy các script chính.

---

## 2. Mục tiêu bài lab

Các mục tiêu chính của bài lab gồm:

- Hiểu cách gọi trực tiếp LLM và truyền message cho mô hình.
- Biết cách dùng tools để LLM tra cứu hoặc xử lý dữ liệu bên ngoài.
- Hiểu ReAct pattern trong việc xây dựng single agent.
- Biết cách dùng LangGraph để thiết kế multi-agent system.
- Hiểu cách các agent phân tán giao tiếp qua A2A Protocol.
- Quan sát được cách Registry hỗ trợ service discovery.
- Thực hành mở rộng hệ thống bằng cách thêm tool và agent mới.

---

## 3. Tổng quan hệ thống

Hệ thống trong lab là một demo tư vấn pháp lý. Ở phiên bản đầy đủ, hệ thống có các thành phần chính:

- **Customer Agent:** nhận câu hỏi từ người dùng.
- **Law Agent:** phân tích pháp lý tổng quát và điều phối các agent chuyên môn.
- **Tax Agent:** xử lý các vấn đề liên quan đến thuế.
- **Compliance Agent:** xử lý các vấn đề về tuân thủ pháp lý.
- **Registry Service:** nơi các agent đăng ký và tìm kiếm nhau khi cần.

Điểm đáng chú ý là ở Stage 5, các agent chạy như những service riêng. Thay vì hardcode địa chỉ của nhau, agent sẽ gọi Registry để tìm service phù hợp. Cách làm này giống với thiết kế hệ thống thực tế hơn, vì có thể mở rộng hoặc thay đổi từng agent độc lập.

---

## 4. Quá trình thực hiện

### 4.1. Stage 1 - Direct LLM

Ở Stage 1, em tìm hiểu cách gọi LLM trực tiếp bằng `ChatOpenAI` thông qua OpenRouter. Phần này giúp em nắm được cấu trúc cơ bản của một lần gọi LLM, gồm system message và human message.

Em cũng đã thêm `temperature=0.3` vào hàm `get_llm()` trong `common/llm.py`. Việc này giúp output ổn định hơn, phù hợp với bài toán pháp lý vì câu trả lời cần nhất quán và ít ngẫu nhiên.

### 4.2. Stage 2 - LLM với Tools và Knowledge Base

Ở Stage 2, LLM không chỉ trả lời dựa trên kiến thức có sẵn mà còn có thể gọi tools. Em đã hoàn thành hai yêu cầu chính trong `exercises/exercise_2_tools.py`.

Thứ nhất, em thêm một entry mới vào `LEGAL_KNOWLEDGE` về luật lao động Việt Nam. Entry này có các keyword như `lao động`, `sa thải`, `hợp đồng lao động`, `labor`, `termination`, giúp hệ thống có thể tra cứu khi câu hỏi liên quan đến chấm dứt hợp đồng lao động.

Thứ hai, em tạo tool `check_statute_of_limitations`. Tool này nhận loại vụ án như `contract`, `tort`, `property` và trả về thời hiệu khởi kiện tương ứng. Sau đó thêm tool vào danh sách tools và bổ sung phần xử lý tool call trong vòng lặp thủ công.

Qua phần này, em hiểu rõ hơn cách LLM chọn tool, cách chương trình thực thi tool, rồi đưa kết quả tool quay lại cho LLM để tổng hợp câu trả lời.

### 4.3. Stage 3 - Single Agent với ReAct

Ở Stage 3, hệ thống chuyển từ manual tool loop sang ReAct agent. Agent có thể tự quyết định nên gọi tool nào, gọi với tham số gì và khi nào thì đủ thông tin để trả lời.

Em đã thêm tool `search_case_law` vào `stages/stage_3_single_agent/main.py`. Tool này dùng để tra cứu một số án lệ cơ bản theo keyword, ví dụ:

- `breach`: Hadley v. Baxendale.
- `negligence`: Donoghue v. Stevenson.
- `contract`: Carlill v. Carbolic Smoke Ball Co.

Sau đó đưa tool này vào danh sách `TOOLS` để ReAct agent có thể sử dụng. Khi chạy Stage 3, chương trình có thể hiển thị các bước agent gọi tool và quan sát kết quả, giúp dễ hình dung hơn về vòng lặp Think - Act - Observe.

### 4.4. Stage 4 - Multi-Agent In-Process

Ở Stage 4, thêm một agent mới là **Privacy Agent** trong `exercises/exercise_4_multiagent.py`. Agent này tập trung vào GDPR, bảo vệ dữ liệu cá nhân, privacy rights và data breach.

Các phần đã thực hiện gồm:

- Viết hàm `privacy_agent` theo pattern giống `tax_agent` và `compliance_agent`.
- Thêm điều kiện routing khi câu hỏi có các từ khóa `data`, `privacy`, `gdpr`, `dữ liệu`.
- Thêm `privacy_analysis` vào phần tổng hợp kết quả.
- Đăng ký node `privacy_agent` trong graph.
- Thêm edge từ `privacy_agent` đến `aggregate_results`.

Khi chạy thử, em gặp lỗi do hàm routing trả về danh sách `Send` nhưng lại được dùng như một node thông thường. Sau khi xem lại cách LangGraph hoạt động, em sửa lại bằng cách dùng `add_conditional_edges` trực tiếp từ `law_agent`. Nhờ vậy graph chạy đúng và kết quả cuối có phần phân tích privacy/GDPR.

### 4.5. Stage 5 - Distributed A2A System

Ở Stage 5, em tìm hiểu hệ thống khi các agent được tách thành các service riêng. Các service gồm Registry, Customer Agent, Law Agent, Tax Agent và Compliance Agent.

Điểm quan trọng ở phần này là cơ chế self-register và dynamic discovery. Khi khởi động, các agent đăng ký năng lực của mình với Registry. Khi cần gọi agent khác, hệ thống sẽ discover qua Registry thay vì gọi URL cố định.

Em cũng chỉnh prompt của `Tax Agent` trong `tax_agent/graph.py` để agent trả lời ngắn gọn hơn, tập trung vào rủi ro thuế chính, mức phạt, cơ quan liên quan và bước xử lý tiếp theo. Đây là yêu cầu của bài tập 5.3.

---

## 5. Những gì học được

Qua bài lab, em học được khá nhiều kiến thức thực tế về xây dựng ứng dụng AI agent.

Trước hết, em hiểu rằng gọi LLM trực tiếp chỉ phù hợp với bài toán đơn giản. Khi cần thông tin cụ thể hoặc xử lý theo logic riêng, cần kết hợp LLM với tools và knowledge base.

Hiểu rõ hơn sự khác nhau giữa manual tool loop và ReAct agent. Ở manual loop, lập trình viên phải tự xử lý từng tool call. Còn với ReAct agent, agent có thể tự chọn công cụ và lặp lại quá trình suy luận cho đến khi có câu trả lời.

Với LangGraph, em học được cách mô hình hóa luồng xử lý dưới dạng graph. Các khái niệm như state, node, edge, conditional edge và `Send` rất hữu ích khi cần chia bài toán cho nhiều agent cùng xử lý.

Phần multi-agent giúp em thấy rõ lợi ích của việc chuyên môn hóa. Một agent tổng quát có thể trả lời được nhiều thứ, nhưng khi tách thành Law Agent, Tax Agent, Compliance Agent hoặc Privacy Agent, câu trả lời có thể rõ hơn và dễ mở rộng hơn.

Ở Stage 5, em hiểu thêm về cách triển khai agent theo hướng distributed system. Registry Service và A2A Protocol giúp các agent giao tiếp có tổ chức hơn, thay vì phụ thuộc vào địa chỉ hardcode.

---

## 6. Khó khăn gặp phải

Khó khăn đầu tiên là phân biệt vai trò của từng stage. Ban đầu Stage 2 và Stage 3 khá giống nhau vì đều có tools, nhưng sau khi làm em hiểu rằng Stage 2 là gọi tool thủ công, còn Stage 3 để agent tự điều phối.

Khó khăn thứ hai là phần routing trong LangGraph. Lỗi ở Exercise 4 giúp em hiểu rõ hơn rằng một node cần trả về dict cập nhật state, còn routing function trả về `Send` thì phải được dùng trong conditional edge.

Ngoài ra, Stage 5 có nhiều service chạy cùng lúc nên việc debug cần theo dõi log cẩn thận. `trace_id` là một phần quan trọng để theo dõi request đi qua các agent.

---

## 7. Kết luận

Sau bài lab, em đã nắm được quy trình xây dựng một hệ thống AI agent từ đơn giản đến phức tạp. Đã thực hành gọi LLM, thêm tools, xây dựng ReAct agent, thêm agent mới vào LangGraph và tìm hiểu cách các agent phân tán giao tiếp qua A2A Protocol.

Bài lab giúp em nhận ra rằng để xây dựng ứng dụng AI thực tế, chỉ dùng LLM là chưa đủ. Cần có tools, routing, state management, agent specialization và cơ chế giao tiếp giữa các thành phần. Những kiến thức này là nền tảng tốt để tiếp tục tìm hiểu các phần nâng cao như memory, authentication, retry logic và monitoring trong hệ thống multi-agent.
