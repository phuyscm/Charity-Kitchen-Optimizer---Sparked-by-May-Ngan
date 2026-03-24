# Hệ Thống Tối Ưu Hóa Bếp Ăn Từ Thiện

Một ứng dụng Web dựa trên Quy hoạch tuyến tính (LP) ngẫu nhiên, được thiết kế để tối ưu hóa các quyết định mua sắm hàng ngày cho các bếp ăn từ thiện xã hội. 

Lấy cảm hứng từ hoạt động hàng ngày của Quán cơm xã hội Mây Ngàn 2.000 VNĐ, dự án này giải quyết một câu hỏi vận hành cốt lõi: Làm thế nào để duy trì ổn định 360 suất ăn đủ dinh dưỡng với ngân sách eo hẹp khi nguồn thực phẩm quyên góp mỗi ngày là hoàn toàn ngẫu nhiên?

## Giá Trị Kinh Doanh & Tác Động
- Hiệu quả chi phí: Tối thiểu hóa chi phí mua sắm hàng ngày trong khi tuân thủ nghiêm ngặt các tiêu chuẩn dinh dưỡng của Bộ Y tế Việt Nam.
- Xử lý biến động ngẫu nhiên: Tự động điều chỉnh quyết định mua sắm dựa trên lượng hàng hóa quyên góp (in-kind donations) biến động mỗi ngày.
- Độ chính xác: Đã được back-test trên dữ liệu hóa đơn mua hàng thực tế của bếp ăn, đạt biên độ sai số chi phí chỉ 1.5%.
- Giao diện thực tiễn: Tích hợp mô hình toán học vào giao diện Web Flask, cho phép người quản lý bếp (không cần kiến thức kỹ thuật) dễ dàng nhập dữ liệu quyên góp và nhận danh sách mua sắm tức thời.

---

## Công Nghệ Sử Dụng
- Mô hình Toán học: Quy hoạch tuyến tính (Linear Programming), Bài toán pha trộn (Blending Problem), Ràng buộc mềm (Soft Constraints).
- Bộ giải (Solver Engine): Google OR-Tools (GLOP).
- Backend/API: Python, Flask.
- Frontend: HTML, Bootstrap (Jinja2 templates).

---

## Cấu Trúc Mô Hình Toán Học

Lõi tính toán là một bài toán Quy hoạch tuyến tính bao gồm 16 biến chính và 17 ràng buộc cấu trúc. Điểm nhấn kiến trúc là việc sử dụng các ràng buộc mềm để ngăn chặn lỗi vô nghiệm (INFEASIBLE) khi lượng hàng quyên góp làm lệch cân bằng dinh dưỡng vĩ mô.

### 1. Tham Số Đầu Vào & Dữ Liệu Mô Phỏng

Dưới đây là định nghĩa các tham số đầu vào kèm theo một tập dữ liệu giả lập (Mock Data) minh họa cách lượng hàng hóa quyên góp ($D_k$) và giới hạn dinh dưỡng tương tác với nhau.

- $K$: Tập hợp các nguyên liệu cần thiết.
- $J$: Tập hợp các tiêu chuẩn dinh dưỡng mục tiêu.
- $N$: Số lượng suất ăn mục tiêu mỗi ngày (ví dụ: 360).
- $B$: Ngân sách mua sắm tối đa trong ngày.
- $P_k$: Đơn giá trên mỗi kg của nguyên liệu $k$.
- $W_k$: Khối lượng định mức (kg) cần thiết cho 1 suất ăn của nguyên liệu $k$.
- $D_k$: Khối lượng nguyên liệu $k$ được quyên góp (đã giới hạn tối đa bằng mức nhu cầu trong ngày).
- $Nut_{k,j}$: Hàm lượng chất dinh dưỡng $j$ có trong 1 đơn vị nguyên liệu $k$.
- $Min_j, Max_j$: Giới hạn dưới và trên của tiêu chuẩn dinh dưỡng $j$.
- $M_1, M_2$: Trọng số phạt khi vi phạm dinh dưỡng ($M_1 \gg M_2$ nhằm ưu tiên việc đạt ngưỡng tối thiểu hơn là tránh việc dư thừa).

Bảng 1: Thông Số Nguyên Liệu & Quyên Góp (Mô phỏng)

| Nguyên Liệu | Giá/kg (VNĐ) | Định mức/Suất (g) | Quyên góp ($D_k$) | Calo/100g | Protein/100g |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Gạo trắng | 18,000 | 100 | 12.5 kg | 130 | 2.7 |
| Thịt heo | 90,000 | 50 | 0.0 kg | 242 | 14.0 |
| Rau cải | 20,000 | 80 | 30.0 kg | 15 | 1.5 |

Bảng 2: Tiêu Chuẩn Dinh Dưỡng/Suất Ăn (Mô phỏng)

| Dinh Dưỡng ($J$) | Giới hạn Min | Giới hạn Max |
| :--- | :--- | :--- |
| Calo (kcal) | 650 | 900 |
| Protein (g) | 25 | 45 |

### 2. Biến Quyết Định (Decision Variables)
- $x_k \geq 0$: Khối lượng (kg) nguyên liệu $k$ cần mua thêm.
- $s_j \geq 0$: Biến nới lỏng (Slack variable) đại diện cho lượng dinh dưỡng $j$ bị thiếu hụt.
- $u_j \geq 0$: Biến nới lỏng (Surplus variable) đại diện cho lượng dinh dưỡng $j$ bị vượt mức.

### 3. Hàm Mục Tiêu (Objective Function)
Tối thiểu hóa tổng chi phí mua sắm thực tế cộng với điểm phạt khi vi phạm tiêu chuẩn dinh dưỡng.

Dạng Tổng Quát (Sigma Formulation):
$$\min Z = \sum_{k \in K} (P_k \cdot x_k) + \sum_{j \in J} (M_1 \cdot s_j + M_2 \cdot u_j)$$

Dạng Khai Triển (Minh họa ma trận bộ giải):
$$\min Z = 15000 x_{rice} + 80000 x_{pork} + ... + 10^6 s_{calo} + 10^4 u_{calo} + ...$$

### 4. Các Ràng Buộc (Constraints)

[C1] Đảm Bảo Nguồn Cung (Giới hạn dưới của biến)
Tổng lượng nguyên liệu (Mua thêm + Quyên góp) phải đáp ứng đủ định mức công thức cho $N$ suất ăn.
$$x_k \geq (N \cdot W_k) - D_k \quad \forall k \in K$$

[C2] Tiêu Chuẩn Dinh Dưỡng Tối Thiểu (Ràng buộc mềm)
$$\sum_{k \in K} (Nut_{k,j} \cdot x_k) + s_j \geq Min_j - DonatedNut_j \quad \forall j \in J$$

[C3] Tiêu Chuẩn Dinh Dưỡng Tối Đa (Ràng buộc mềm)
$$\sum_{k \in K} (Nut_{k,j} \cdot x_k) - u_j \leq Max_j - DonatedNut_j \quad \forall j \in J$$

[C4] Giới Hạn Ngân Sách (Ràng buộc cứng)
$$\sum_{k \in K} (P_k \cdot x_k) \leq B$$

---

## Cài Đặt & Sử Dụng

### Yêu cầu hệ thống
Đảm bảo máy tính đã cài đặt Python 3.8 trở lên.

### Thiết lập
1. Clone repository về máy:
   ```bash
   git clone [https://github.com/phuyscm/charity-kitchen-optimizer.git](https://github.com/phuyscm/charity-kitchen-optimizer.git)
   cd charity-kitchen-optimizer
