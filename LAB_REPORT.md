# Lab Report: HITL PR Review Agent

## 1. Mục tiêu

Bài lab xây dựng một agent review pull request có **Human-in-the-Loop (HITL)** bằng LangGraph.  
Agent có thể:

- đọc diff của pull request,
- phân tích và sinh nhận xét review,
- tự chọn luồng xử lý theo mức confidence,
- yêu cầu con người can thiệp khi cần,
- ghi lại toàn bộ quá trình vào audit trail,
- và cung cấp giao diện web bằng Streamlit.

## 2. Kiến trúc hệ thống

Luồng tổng quát của hệ thống:

```text
fetch_pr → analyze → route
                     ├─ auto_approve
                     ├─ human_approval → commit
                     └─ escalate → synthesize → commit
```

Các node chính:

- `fetch_pr`: lấy metadata và diff từ GitHub
- `analyze`: gọi LLM để tạo `PRAnalysis`
- `route`: chọn nhánh xử lý dựa trên confidence
- `human_approval`: dừng graph để reviewer approve / reject / edit
- `escalate`: hỏi reviewer các câu hỏi cụ thể khi confidence thấp
- `synthesize`: phân tích lại review sau khi có câu trả lời từ reviewer
- `commit`: đăng review comment lên pull request

## 3. Luồng xử lý theo confidence

Hệ thống dùng confidence để chia pull request thành ba nhóm:

| Confidence | Nhánh xử lý | Ý nghĩa |
| --- | --- | --- |
| Cao | `auto_approve` | Agent đủ chắc chắn để tự đăng review |
| Trung bình | `human_approval` | Cần reviewer xác nhận trước khi commit |
| Thấp | `escalate` | Agent thiếu ngữ cảnh và cần hỏi thêm con người |

Trong bài lab:

- PR #1 đại diện cho thay đổi mức trung bình và phù hợp với nhánh `human_approval`
- PR #2 chứa nhiều rủi ro hơn và phù hợp với nhánh `escalate`

## 4. Human-in-the-Loop

Điểm cốt lõi của bài lab là cơ chế:

```python
interrupt(...)
Command(resume=...)
```

Cơ chế này cho phép graph:

- tạm dừng tại đúng điểm cần con người,
- lưu trạng thái hiện tại,
- và tiếp tục từ vị trí cũ sau khi reviewer phản hồi.

Nhờ vậy, agent không chỉ chạy tự động một chiều mà có thể phối hợp với con người trong các quyết định quan trọng.

## 5. Audit trail và checkpointing

Hệ thống dùng SQLite cho hai mục đích khác nhau:

| Cơ chế | Mục đích |
| --- | --- |
| LangGraph checkpointer | Lưu trạng thái để resume graph sau khi bị gián đoạn |
| `audit_events` | Lưu log có cấu trúc để kiểm tra, replay và audit |

Audit trail ghi lại các sự kiện như:

- fetch PR,
- analyze,
- route,
- human approval,
- escalation,
- synthesize,
- commit.

Nhờ đó, toàn bộ phiên review có thể được xem lại bằng:

```bash
python -m audit.replay --list
python -m audit.replay --thread <thread_id>
```

## 6. Giao diện Streamlit

Streamlit UI cho phép reviewer:

- nhập URL pull request,
- xem review do agent đề xuất,
- approve / reject / edit khi confidence trung bình,
- trả lời các câu hỏi escalation khi confidence thấp,
- xem kết quả cuối cùng,
- và theo dõi các session gần đây.

## 7. Kết quả kiểm thử

Các phần đã được kiểm tra:

- confidence routing hoạt động đúng,
- HITL có thể pause và resume,
- escalation branch hoạt động,
- audit trail có thể replay lại đầy đủ,
- Streamlit UI hiển thị đúng luồng theo từng nhánh.

Các bài test chính:

```bash
python exercises/exercise_1_confidence.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1
python exercises/exercise_1_confidence.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/2

python exercises/exercise_2_hitl.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1
python exercises/exercise_3_escalation.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/2
python exercises/exercise_4_audit.py --pr https://github.com/VinUni-AI20k/PR-Demo/pull/1

python -m streamlit run app.py
```

## 8. Bài học rút ra

- Không phải pull request nào cũng cần cùng một mức độ can thiệp của con người.
- Confidence có thể được dùng như tín hiệu điều khiển flow, không chỉ là một con số hiển thị.
- HITL tốt không chỉ hỏi “approve hay reject”, mà còn biết hỏi đúng câu hỏi khi thiếu ngữ cảnh.
- Checkpointing và audit trail là hai nhu cầu khác nhau, nên được thiết kế tách biệt.
- Một agent đáng tin cậy cần không chỉ sinh câu trả lời, mà còn phải để lại dấu vết ra quyết định có thể kiểm tra lại.

