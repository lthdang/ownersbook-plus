Chạy các test case trong [đường dẫn file TC] trên môi trường local ([URL], branch [branch]).

Với mỗi test case:
- Thực hiện đúng từng bước; dữ liệu cần chuẩn bị tạm (DB, header) thì ghi lại và revert sau khi chạy.
- Điền Actual Result vào bảng danh sách (không điền ở bảng chi tiết), mô tả điều quan sát được và đường dẫn screenshot: `screenshot/TC_xxx.png` (nhiều ảnh thì phân biệt bằng hậu tố _step3...).
- Chụp màn hình lưu vào thư mục screenshot/ cạnh file TC.
- Nếu kết quả khác Expected: không sửa code, không sửa Expected. Ghi vào Notes
  "FAILED locally ([ngày])" kèm nguyên nhân quan sát được (status code, message, stack trace nếu có).
- Nếu bước không thực hiện được đúng như mô tả (ví dụ radio không hiển thị với account này), ghi chênh lệch trong Actual Result trong ngoặc ở đầu.
- Để trống Status và Bug ID.
- Cuối cùng liệt kê dữ liệu tạm đã đổi và xác nhận đã khôi phục.

--- convert to English ---
Run the test cases in `/home/haidang/Works/ownersbook+/Works/Tasks/HASHST_DEV-4562/test-case/TC_HASHST_DEV-4562_sold-out-label.md` on the local environment.
For each test case:
- Follow each step correctly; if temporary data needs to be prepared (DB, header), record it and revert after running.
- Fill in the Actual Result in the list table (do not fill in the detail table), describe the observed behavior and screenshot path: `screenshot/TC_xxx.png` (if multiple images, distinguish with suffix _step3...).
- Take screenshots and save them in the screenshot/ directory next to the TC file.
- If the result differs from Expected: do not modify the code or Expected. Record in Notes "FAILED locally ([date])" along with the observed reason (status code, message, stack trace if any).
- If a step cannot be performed as described (e.g., a radio button does not display for this account), note the discrepancy in Actual Result in parentheses at the beginning.
- Leave Status and Bug ID blank.
- Finally, list the temporary data that was changed and confirm it has been restored.