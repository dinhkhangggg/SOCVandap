# TÀI LIỆU ÔN TẬP VÀ 80 CÂU HỎI VẤN ĐÁP BẢO VỆ BÁO CÁO CUỐI KỲ SOC
## ĐỀ TÀI: THIẾT KẾ VÀ ĐÁNH GIÁ BỘ TĂNG TỐC NHÂN MA TRẬN THEO KHỐI TƯƠNG THÍCH LUỒNG DỮ LIỆU SỬ DỤNG VITIS HLS TRÊN NỀN TẢNG ZYNQ-7000 ZC702
**Môn học: Thiết kế Hệ thống trên vi mạch (SoC Design - SOCD334264)**  
**Đơn vị: Bộ môn Kỹ thuật Máy tính – Khoa Điện – Điện tử, Trường ĐH Sư phạm Kỹ thuật TP.HCM (HCMUTE)**  
*Nhóm tác giả (Nhóm 04): Chống Định Khang, Phạm Trung Kiên, Nguyễn Hồng Lịch, Hoàng Bảo Phúc, Lê Thanh Tú*

---

# PHẦN 1: TÓM TẮT CỐT LÕI SIÊU NGẮN GỌN (CHEAT SHEET ÔN NHANH 5 PHÚT)

### 1. Bản chất đề tài trong 3 gạch đầu dòng
1. **Vấn đề giải quyết:** Phép nhân ma trận $N \times N$ có độ phức tạp bậc ba $\mathcal{O}(N^3)$. Khi chạy trên CPU tuần tự (ARM Cortex-A9), CPU bị nghẽn cổ chai bộ nhớ (**von Neumann bottleneck**). Nếu làm phần cứng thông thường (Baseline), số DSP tăng vọt theo $N$, ma trận $N \ge 128$ sẽ tràn chip Zynq XC7Z020.
2. **Giải pháp kiến trúc:** Chia ma trận thành các khối con $B \times B$ ($B=4$ hoặc $B=8$), áp dụng phép **Tích ngoài (Outer Product)** tương thích luồng `hls::stream`. Dải ma trận ngang của $A$ được lưu đệm trên Block RAM để **tái sử dụng 4 lần liên tiếp, cắt giảm $75\%$ băng thông đọc DDR RAM**.
3. **Đột phá thực nghiệm:**
   * **Bất biến tài nguyên:** Số DSP và BRAM không đổi khi $N$ tăng ($B=8$ luôn cố định ở **192 DSP** và **8 BRAM** dù $N=8, 16$ hay $32$).
   * **Tối ưu tốc độ:** Đạt $II = 1$, hoàn tất nhân ma trận $32 \times 32$ (khối $8 \times 8$) chỉ trong **1,832 chu kỳ** ($18.32\ \mu\text{s}$ ở $100\text{ MHz}$), tăng tốc **$15.9\text{ lần}$** so với khi chưa tối ưu.
   * **Tích hợp SoC hoàn chỉnh:** Kết nối qua **Full-Loop Dual DMA** vào cổng hiệu năng cao **`S_AXI_HP0`** của Zynq PS. Đạt chuẩn định thời ($WNS = +0.320\text{ ns}$) và công suất an toàn ($1.762\text{ W}$).

---

### 2. Bảng thông số kỹ thuật "nằm lòng" để trả lời ngay khi bị hỏi

| Thông số / Đại lượng | Giá trị thực tế trong đề tài | Ý nghĩa kỹ thuật cốt lõi |
| :--- | :--- | :--- |
| **Chip mục tiêu** | Zynq-7000 XC7Z020-1CLG484C (kit ZC702) | Dòng FPGA SoC lai: ARM Cortex-A9 + Artix-7 PL. |
| **Tần số / Chu kỳ clock** | $100\text{ MHz}$ / $T_{\text{clk}} = 10\text{ ns}$ | Chu kỳ xung nhịp chuẩn của bộ tăng tốc và bus DMA. |
| **Quỹ DSP của chip** | **220 khối DSP48E1** | Mỗi khối là bộ nhân $18 \times 25\text{-bit}$. |
| **Tài nguyên DSP dùng ($B=8$)**| **192 khối DSP48E1** ($87.27\%$) | 64 bộ MAC song song; mỗi phép nhân `int` 32-bit tốn 3 DSP. |
| **Tài nguyên BRAM dùng ($B=8$)**| **8 khối BRAM_18K** ($2.86\%$) | Chứa mảng tĩnh $A[8][32]$ sau khi gom từ 256-bit. |
| **Tổng chu kỳ Cosim ($32 \times 32$)**| **1,832 chu kỳ** ($18.32\ \mu\text{s}$) | Thời gian chạy thực tế trên dạng sóng C/RTL Cosimulation. |
| **Tỷ lệ tiết kiệm đọc RAM** | **$75\%$** | Dải ngang $A$ chỉ nạp 1 lần ở đầu mỗi hàng khối, dùng lại 3 lần. |
| **Worst Negative Slack (WNS)** | **$+0.320\text{ ns}$** ($> 0$, an toàn) | Mạch chạy nhanh hơn yêu cầu clock, $0$ điểm lỗi timing. |
| **Tổng công suất trên chip** | **$1.762\text{ W}$** (PS chiếm $94\%$ động) | PS tốn $1.573\text{ W}$ do ARM xung cao; PL tốn rất ít ở trạng thái tĩnh. |
| **Nhiệt độ tiếp giáp ($T_j$)** | **$45.3^\circ\text{C}$** (Ngưỡng hỏng $85^\circ\text{C}$) | Biên độ an toàn nhiệt lên tới $39.7^\circ\text{C}$. |

---

# PHẦN 2: 80 CÂU HỎI VẤN ĐÁP BẢO VỆ BÁO CÁO CUỐI KỲ (KÈM LỜI GIẢI CHI TIẾT)

---

## CHUYÊN ĐỀ 1: VẤN ĐÁP TRÊN HÌNH 1, HÌNH 2 & KIẾN TRÚC BLOCKMATMUL (CÂU 1 - 10)

#### Câu 1: *Thầy chỉ vào Hình 1 (Sơ đồ khối Blockmatmul) hỏi: "Nhìn vào tấm ảnh này, em biết được gì? Phân tích kiến trúc tổng thể của bộ tăng tốc xem?"*
* **Trả lời:**
  * Thưa thầy, Hình 1 minh họa kiến trúc nhân ma trận theo khối dựa trên **tích ngoài (Outer Product)** từ nghiên cứu của Kastner et al.
  * Ba thành phần cốt lõi trên hình:
    1. **Hai luồng dữ liệu đầu vào:** `Arows` và `Bcols` được truyền dạng Stream, mỗi gói là một vector $B$ phần tử.
    2. **Mảng lưu trữ trên chip:** Mảng tĩnh $A$ kích thước $B \times N$ lưu trong Block RAM và mảng tích lũy $AB$ kích thước $B \times B$ lưu trên thanh ghi.
    3. **Lõi tính toán song song:** Thực hiện tích ngoài giữa 1 vector cột của $A$ và 1 vector hàng của $B$ để cập nhật đồng thời cho toàn bộ khối $AB[B][B]$.

#### Câu 2: *Thầy chỉ vào Hình 1 hỏi: "Tại sao trong hình lại vẽ dải ngang của A kích thước $B \times N$ còn dải dọc của B lại là $N \times B$?"*
* **Trả lời:**
  * Để tạo ra một khối kết quả $AB_{\text{partial}}$ kích thước $B \times B$, theo nguyên lý đại số tuyến tính:
    * Ta cần lấy $B$ hàng của ma trận $A$ (dải ngang $B \times N$) nhân với $B$ cột của ma trận $B$ (dải dọc $N \times B$).
    * Phép nhân giữa dải $B \times N$ và dải $N \times B$ tương đương với tổng của $N$ phép nhân tích ngoài giữa vector cột $B \times 1$ và vector hàng $1 \times B$.

#### Câu 3: *Thầy chỉ vào Hình 2 (Đoạn code C++) hỏi: "Nhìn vào đoạn mã này, biến `blockvec` được định nghĩa thế nào và tại sao lại dùng struct?"*
* **Trả lời:**
  * `blockvec` được định nghĩa là một cấu trúc chứa mảng $B$ phần tử số nguyên 32-bit: `typedef struct { DTYPE a[BLOCK_SIZE]; } blockvec;`.
  * **Mục đích dùng struct:** Để công cụ Vitis HLS đóng gói $B$ phần tử thành một gói dữ liệu duy nhất trên giao tiếp AXI4-Stream (với $B=4$ là từ 128-bit, với $B=8$ là từ 256-bit). Nhờ đó, mỗi chu kỳ xung nhịp có thể truyền trọn vẹn cả vector $B$ phần tử mà không phải truyền từng số lẻ tẻ.

#### Câu 4: *Thầy chỉ vào Hình 2 hỏi: "Dòng lệnh `int counter = it % (N / B);` và `if (counter == 0)` có vai trò sống còn gì trong thuật toán?"*
* **Trả lời:**
  * Đây là **bộ điều khiển nạp dữ liệu có điều kiện**:
    * `it` là biến đếm số thứ tự khối kết quả ($0$ đến $(N/B)^2 - 1$).
    * `counter == 0` chỉ đúng ở khối đầu tiên của mỗi hàng khối (ví dụ với $N=32, B=8$ thì $N/B=4$, đúng khi `it = 0, 4, 8, 12`).
    * **Vai trò:** Khi `counter == 0`, mạch mới kích hoạt đọc luồng `Arows` để nạp dải ngang $A$ mới vào BRAM. Ở các khối tiếp theo (`counter != 0`), mạch bỏ qua việc đọc `Arows`, giúp **tiết kiệm $75\%$ băng thông đọc DDR RAM**.

#### Câu 5: *Thầy hỏi: "Nếu em xóa dòng `if (counter == 0)` và cho nạp luồng `Arows` liên tục ở mỗi lần gọi hàm thì hậu quả là gì?"*
* **Trả lời:**
  * Hệ thống vẫn tính ra kết quả đúng nhưng:
    1. Băng thông đọc ma trận A tăng gấp 4 lần (mất đi khả năng tiết kiệm $75\%$).
    2. Độ trễ của mọi khối đều bị kéo dài thành $\text{Latency}_{\max} = 145\text{ chu kỳ}$, làm tổng thời gian nhân ma trận tăng thêm hàng trăm chu kỳ, làm giảm hiệu năng toàn hệ thống.

#### Câu 6: *Thầy chỉ vào Hình 2 hỏi: "Tại sao mảng A lại được khai báo là `static DTYPE A[B][N];`? Từ khóa `static` có ý nghĩa gì đối với phần cứng?"*
* **Trả lời:**
  * Trong C++, `static` giữ giá trị của biến tồn tại xuyên suốt giữa các lần gọi hàm.
  * **Ý nghĩa phần cứng trong HLS:** Khi tổng hợp, Vitis HLS sẽ ánh xạ biến `static` thành **bộ nhớ nội vi Block RAM (BRAM) trên chip FPGA** thay vì dùng thanh ghi tạm thời. Vùng nhớ này lưu giữ dải dữ liệu $A$ để các lần gọi hàm tiếp theo truy xuất lại mà không bị mất dữ liệu.

#### Câu 7: *Thầy chỉ vào Hình 2 hỏi: "Vòng lặp `PARTIAL_SUM_K` chạy bao nhiêu lần và bên trong nó làm nhiệm vụ gì?"*
* **Trả lời:**
  * Vòng lặp chạy đúng **$N$ lần** ($k = 0 \dots N-1$).
  * Mỗi bước lặp $k$: Mạch đọc 1 vector từ luồng `Bcols`, lấy cột thứ $k$ của mảng $A$, sau đó thực hiện $B \times B = 64$ phép nhân và cộng tích lũy đồng thời vào mảng $AB[B][B]$. Đây chính là giai đoạn tính toán tích ngoài song song.

#### Câu 8: *Thầy hỏi: "Tại sao mảng $AB[B][B]$ lại phải được xóa về 0 ở vòng lặp `INIT_AB` trước khi tính toán?"*
* **Trả lời:**
  * Vì $AB$ là mảng tích lũy cộng dồn ($AB += A \times B$).
  * Nếu không reset về 0 ở đầu mỗi khối, giá trị của khối tính toán trước đó sẽ bị cộng dồn tiếp vào khối hiện tại, dẫn đến kết quả ma trận bị sai hoàn toàn.

#### Câu 9: *Thầy hỏi: "Hàm `blockmatmul` xuất kết quả ra luồng `ABpartial` ở giai đoạn nào và xuất bao nhiêu phần tử?"*
* **Trả lời:**
  * Xuất ở giai đoạn 4 (vòng lặp `WRITE_OUT_COL`), ngay sau khi vòng lặp `PARTIAL_SUM_K` tính xong $N$ bước tích ngoài.
  * Mạch xuất đúng **$B$ gói `blockvec`** (tương đương toàn bộ ma trận con $B \times B$) ra luồng ngõ ra `ABpartial`.

#### Câu 10: *Thầy hỏi: "Cấu hình $N=32, B=8$ thì hàm `blockmatmul` được gọi tổng cộng bao nhiêu lần để tính xong toàn bộ ma trận?"*
* **Trả lời:**
  * Số lần gọi hàm bằng đúng số khối con của ma trận:
    $$\text{Số lần} = \left(\frac{N}{B}\right)^2 = \left(\frac{32}{8}\right)^2 = 4^2 = \mathbf{16\text{ lần}}$$
  * Trong đó có 4 lần chạy ở độ trễ cực đại (có nạp $A$) và 12 lần chạy ở độ trễ cực tiểu (tái sử dụng $A$).

---

## CHUYÊN ĐỀ 2: VẤN ĐÁP TRÊN HÌNH 3, BẢNG 2 & CÁC CHỈ THỊ DIRECTIVES (CÂU 11 - 20)

#### Câu 11: *Thầy chỉ vào Hình 3 (Minh họa ARRAY_RESHAPE) hỏi: "Nhìn vào tấm ảnh này, em phân tích cơ chế hoạt động của `ARRAY_RESHAPE` xem? Nó khác gì `ARRAY_PARTITION`?"*
* **Trả lời:**
  * Thưa thầy, Hình 3 chỉ ra cơ chế gộp dữ liệu của `ARRAY_RESHAPE`:
    * Nó **không chia nhỏ mảng thành nhiều khối BRAM rời rạc** như `ARRAY_PARTITION`.
    * Thay vào đó, nó **ghép các phần tử dữ liệu lại với nhau để tạo thành một từ dữ liệu có độ rộng lớn hơn (Wide Word)** lưu trong cùng 1 khối BRAM.
    * Nhờ đó, mạch chỉ cần 1 cổng đọc duy nhất là lấy được toàn bộ $B$ phần tử trong cùng 1 chu kỳ xung nhịp.

#### Câu 12: *Thầy chỉ vào Bảng 2 hỏi: "Tại sao mảng A lại dùng `#pragma HLS ARRAY_RESHAPE variable=A complete dim=1` mà không phải `dim=2`?"*
* **Trả lời:**
  * Mảng $A[B][N]$ có chiều 1 là chiều hàng ($0 \dots B-1$) và chiều 2 là chiều cột ($0 \dots N-1$).
  * Phép tích ngoài đòi hỏi tại mỗi bước $k$ phải đọc **toàn bộ 1 cột $B$ phần tử** của $A$.
  * Chọn `dim=1` nghĩa là gộp tất cả các phần tử theo chiều dọc của cột đó thành 1 từ rộng $B \times 32\text{-bit} = 256\text{-bit}$. Do đó phát 1 địa chỉ cột $k$ là lấy trọn vẹn cả cột $B$ phần tử trong 1 chu kỳ.

#### Câu 13: *Thầy hỏi: "Nếu em đặt nhầm thành `dim=2` cho mảng A thì điều gì sẽ xảy ra?"*
* **Trả lời:**
  * Nếu đặt `dim=2`, HLS sẽ gộp các phần tử trên cùng 1 hàng thành từ rộng $N \times 32\text{-bit}$.
  * Khi mạch cần đọc theo cột, các phần tử nằm rải rác ở các từ nhớ khác nhau. Khối BRAM không cung cấp đủ cổng đọc đồng thời, dẫn đến **vi phạm II (II Violation)**, $II$ sẽ bị tăng lên lớn hơn 1 và độ trễ tăng mạnh.

#### Câu 14: *Thầy chỉ vào Bảng 2 hỏi: "Tại sao mảng AB lại dùng `ARRAY_PARTITION complete dim=0`? Ý nghĩa của `dim=0` là gì?"*
* **Trả lời:**
  * `dim=0` nghĩa là phân tách triệt để **toàn bộ tất cả các chiều** của mảng 2 chiều $AB[B][B]$.
  * **Tác dụng:** Mảng $AB$ bị tách hoàn toàn khỏi BRAM và chuyển thành **$B^2$ thanh ghi Flip-Flop độc lập** ($64\text{ FF}$ với $B=8$). Việc này cho phép 64 bộ cộng có thể đọc, cộng dồn và ghi lại kết quả đồng thời vào 64 thanh ghi trong cùng 1 chu kỳ xung nhịp.

#### Câu 15: *Thầy hỏi: "Tại sao không dùng BRAM cho mảng AB mà bắt buộc phải chuyển sang Flip-Flop bằng `ARRAY_PARTITION`?"*
* **Trả lời:**
  * Vì BRAM vật lý chỉ có tối đa 2 cổng truy xuất mỗi chu kỳ.
  * Trong khi phép tích ngoài đòi hỏi cập nhật đồng thời cả 64 phần tử mỗi chu kỳ. BRAM không thể đáp ứng được 64 cổng ghi cùng lúc, bắt buộc phải dùng Flip-Flop phân tán trên mảng logic FPGA.

#### Câu 16: *Thầy chỉ vào Bảng 2 hỏi: "Chỉ thị `#pragma HLS PIPELINE II=1` đặt tại `PARTIAL_SUM_K` làm được điều gì cho phần cứng?"*
* **Trả lời:**
  * Nó thiết lập đường ống phần cứng đạt khoảng khởi tạo lý tưởng **$II = 1$**.
  * Cứ sau **mỗi chu kỳ xung nhịp**, mạch lại tiếp nhận một bước lặp $k$ mới. Nghĩa là mỗi chu kỳ xung nhịp, hệ thống hoàn thành trọn vẹn 1 phép tính tích ngoài kích thước $B \times B$.

#### Câu 17: *Thầy hỏi: "Chỉ thị `#pragma HLS UNROLL` đặt tại `PARTIAL_SUM_I` và `PARTIAL_SUM_J` có nhiệm vụ gì?"*
* **Trả lời:**
  * Nó duỗi phẳng hoàn toàn hai vòng lặp lồng nhau kích thước $B \times B$.
  * Về mặt phần cứng, nó nhân bản mạch tính toán thành một mảng gồm **$B^2 = 64$ bộ nhân và 64 bộ cộng (MAC)** hoạt động song song hoàn toàn trong không gian logic.

#### Câu 18: *Thầy hỏi: "Nếu không dùng `UNROLL` tại `PARTIAL_SUM_I/J`, mạch sẽ hoạt động thế nào?"*
* **Trả lời:**
  * Mạch sẽ chạy tuần tự từng phép nhân một. Để tính xong 1 bước $k$, mạch mất 64 chu kỳ xung nhịp thay vì 1 chu kỳ. Tốc độ tính toán của bộ tăng tốc sẽ bị chậm đi 64 lần.

#### Câu 19: *Thầy hỏi: "Chỉ thị `PIPELINE II=1` đặt tại `LOAD_A_COL` và `WRITE_OUT_COL` có tác dụng gì?"*
* **Trả lời:**
  * Đảm bảo việc nạp luồng dữ liệu vào và xuất luồng ra đạt thông lượng tối đa: **mỗi chu kỳ xung nhịp truyền nhận thành công đúng 1 gói dữ liệu `blockvec`**, không có chu kỳ chờ bọt khí (bubble cycles).

#### Câu 20: *Thầy hỏi: "Tổ hợp 4 chỉ thị trong Bảng 2 phối hợp với nhau như thế nào để hệ thống đạt hiệu năng tối đa?"*
* **Trả lời:**
  * `ARRAY_RESHAPE` cung cấp đủ dữ liệu cột $A$ trong 1 chu kỳ.
  * `ARRAY_PARTITION` cung cấp 64 thanh ghi đích để ghi dữ liệu trong 1 chu kỳ.
  * `UNROLL` tạo ra 64 bộ nhân/cộng phần cứng để tính toán trong 1 chu kỳ.
  * `PIPELINE II=1` kết nối tất cả lại thành dây chuyền liên tục mỗi chu kỳ 1 bước tích ngoài.

---

## CHUYÊN ĐỀ 3: VẤN ĐÁP TRÊN HÌNH 4, HÌNH 5 & KẾT QUẢ MÔ PHỎNG (CÂU 21 - 30)

#### Câu 21: *Thầy chỉ vào Hình 4 (Kết quả C Synthesis) hỏi: "Nhìn vào tấm ảnh này, em đọc được các thông số hiệu năng nào của cấu hình $N=32, B=8$?"*
* **Trả lời:**
  * Thưa thầy, Hình 4 là báo cáo C Synthesis của Vitis HLS:
    * **Latency:** Dao động từ $110\text{ chu kỳ}$ ($\min$) đến $145\text{ chu kỳ}$ ($\max$), trung bình là $128\text{ chu kỳ}$.
    * **Initiation Interval (II):** Từ 111 đến 146 chu kỳ.
    * **Timing:** Chu kỳ ước tính $8.694\text{ ns}$ (đạt yêu cầu so với mục tiêu $10.00\text{ ns}$).
    * **Tài nguyên:** Tiêu tốn 8 BRAM_18K, 192 DSP48E1, 17,652 FF và 17,582 LUT.

#### Câu 22: *Thầy chỉ vào Hình 4 hỏi: "Tại sao trong báo cáo C Synthesis, Latency lại ghi 2 giá trị Min và Max khác nhau chứ không phải 1 số cố định?"*
* **Trả lời:**
  * Vì hàm `blockmatmul` có cấu trúc rẽ nhánh điều kiện `if (counter == 0)`.
  * **Latency Min (110 chu kỳ):** Xảy ra khi `counter != 0`, bỏ qua bước đọc luồng `Arows`.
  * **Latency Max (145 chu kỳ):** Xảy ra khi `counter == 0`, phải thực hiện thêm vòng lặp nạp luồng `Arows` mất thêm 35 chu kỳ.

#### Câu 23: *Thầy chỉ vào Hình 5 (Báo cáo C/RTL Cosimulation) hỏi: "Nhìn vào ảnh này, em biết được kết quả Cosimulation thế nào? Cột Status ghi 'Pass' có ý nghĩa gì?"*
* **Trả lời:**
  * Thưa thầy, Hình 5 xác nhận quá trình đồng mô phỏng C/RTL đã hoàn tất thành công với trạng thái **Pass**.
  * Chữ **Pass** chứng minh rằng: Vector dữ liệu đầu ra sinh ra bởi mạch phần cứng RTL (Verilog) hoàn toàn khớp từng bit với kết quả tính toán của thuật toán C gốc trên Testbench, không có bất kỳ sai số số học nào.

#### Câu 24: *Thầy chỉ vào Hình 5 hỏi: "So sánh độ trễ thực tế của Cosimulation với ước lượng của C Synthesis, em thấy có gì đặc biệt?"*
* **Trả lời:**
  * Độ trễ trung bình thực tế trên Cosimulation đạt **$114.5\text{ chu kỳ}$**, tốt hơn mức ước lượng tĩnh $128\text{ chu kỳ}$ của C Synthesis.
  * Toàn bộ ma trận $32 \times 32$ được tính xong chỉ trong **1,832 chu kỳ**, nhanh hơn con số tính toán lý thuyết cộng dồn ($1,900\text{ chu kỳ}$).

#### Câu 25: *Thầy hỏi: "Tại sao thời gian chạy thực tế trên Cosimulation lại nhanh hơn mức ước lượng của C Synthesis?"*
* **Trả lời:**
  * Vì C Synthesis ước lượng tĩnh từng lần gọi hàm độc lập (cộng dồn rời rạc).
  * Trong khi mô phỏng RTL thực tế có cơ chế gối đầu đường ống (**Pipelined Execution** giữa các lần gọi hàm): giai đoạn xuất kết quả của khối trước diễn ra đồng thời với giai đoạn nạp dữ liệu của khối sau, giúp tiết kiệm chu kỳ chuyển tiếp.

#### Câu 26: *Thầy hỏi: "Thời gian $1,832\text{ chu kỳ}$ ở tần số $100\text{ MHz}$ quy đổi ra thời gian thực tế là bao nhiêu micro-giây?"*
* **Trả lời:**
  * Với tần số $100\text{ MHz}$, chu kỳ xung nhịp là $T = 10\text{ ns}$.
  * Thời gian thực thi là:
    $$t = 1,832 \times 10\text{ ns} = 18,320\text{ ns} = \mathbf{18.32\ \mu\text{s}}$$

#### Câu 27: *Thầy hỏi: "Trong Testbench (testbench.cpp), nhóm đã dùng hàm gì để sinh ma trận và kiểm tra tính đúng đắn?"*
* **Trả lời:**
  * Nhóm sử dụng hàm cố định hạt giống `std::srand(42)` để sinh các ma trận số nguyên ngẫu nhiên trong khoảng $[-10, 9]$ đồng nhất giữa mọi lần chạy.
  * Sau khi bộ tăng tốc chạy xong, Testbench dùng 3 vòng lặp `for` tiêu chuẩn của phần mềm để nhân lại ma trận và so sánh từng phần tử một giữa phần cứng và phần mềm.

#### Câu 28: *Thầy hỏi: "Nếu kết quả Cosimulation bị sai (Fail), nguyên nhân phần cứng thường nằm ở đâu?"*
* **Trả lời:**
  * Thường do:
    1. Lập lịch đường ống (Scheduling) bị vi phạm phụ thuộc dữ liệu.
    2. Tín hiệu bắt tay FIFO (`read/write`) đọc dữ liệu rác khi luồng chưa sẵn sàng (`empty`).
    3. Tràn số (Overflow) nếu kiểu dữ liệu tích lũy không đủ độ rộng bit.

#### Câu 29: *Thầy hỏi: "Tại sao phải chạy cả C Synthesis lẫn Cosimulation mà không dừng lại ở C Synthesis?"*
* **Trả lời:**
  * C Synthesis chỉ là ước lượng tĩnh trên lý thuyết và sinh mã Verilog.
  * Chỉ có Cosimulation mới đưa mã Verilog vào trình mô phỏng dạng sóng thực tế (Vivado Simulator) với đầy đủ trễ cổng logic và giao thức bắt tay để chứng minh mạch chạy đúng trên thực tế.

#### Câu 30: *Thầy hỏi: "Tỷ số tăng tốc $15.9\times$ được tính dựa trên hai con số nào trong báo cáo?"*
* **Trả lời:**
  * Được tính bằng tỷ số giữa độ trễ trung bình khi chưa dùng Directives ($2,209\text{ chu kỳ}$) và sau khi dùng Directives ($128\text{ chu kỳ}$):
    $$\text{Speedup} = \frac{2,209}{128} \approx \mathbf{15.9\text{ lần}}$$

---

## CHUYÊN ĐỀ 4: VẤN ĐÁP TRÊN HÌNH 6 & DẠNG SÓNG MÔ PHỎNG WAVEFORM (CÂU 31 - 40)

#### Câu 31: *Thầy chỉ vào Hình 6 (Dạng sóng mô phỏng RTL) hỏi: "Nhìn vào tấm ảnh dạng sóng này, em phân tích thanh trạng thái màu xanh dương ở trên cùng xem?"*
* **Trả lời:**
  * Thưa thầy, thanh trạng thái màu xanh dương hiển thị giá trị của biến đếm `it[31:0]` tăng dần từ `00000000` đến `0000000f` (từ 0 đến 15).
  * Nó chia toàn bộ quá trình thành **đúng 16 ô hoạt động liên tục**, tương ứng với 16 khối ma trận $8 \times 8$ được tính toán liên tục không ngừng nghỉ.

#### Câu 32: *Thầy chỉ vào Hình 6 hỏi: "Nhìn vào đường tín hiệu `Arows_read`, em thấy điều gì bất thường? Tại sao nó chỉ bật lên mức cao đúng 4 lần?"*
* **Trả lời:**
  * Tín hiệu `Arows_read` là cờ báo lệnh đọc từ luồng `Arows`.
  * Nó chỉ bật lên mức 1 đúng 4 lần tại các giao dịch: **`it = 00`, `it = 04`, `it = 08` và `it = 0c`** (các khối đầu của mỗi hàng khối).
  * Trong 12 giao dịch còn lại, nó giữ phẳng ở mức 0 vì mạch tái sử dụng dải dữ liệu $A$ đang lưu trong BRAM. Đây là minh chứng trực quan khẳng định việc cắt giảm $75\%$ băng thông đọc ma trận A.

#### Câu 33: *Thầy chỉ vào Hình 6 hỏi: "So sánh độ rộng của các ô giao dịch trên dạng sóng, em thấy ô nào dài hơn và ô nào ngắn hơn? Tại sao?"*
* **Trả lời:**
  * **4 ô giao dịch dài (145 chu kỳ / $1.45\ \mu\text{s}$):** Là các ô `it = 00, 04, 08, 0c` vì có xung `Arows_read` đi trước, mạch mất thêm thời gian nạp mảng $A$.
  * **12 ô giao dịch ngắn (110 chu kỳ / $1.10\ \mu\text{s}$):** Là các ô còn lại, chỉ có xung `Bcols_read`, mạch nhảy thẳng vào tính toán nên thời gian ngắn hơn rõ rệt.

#### Câu 34: *Thầy hỏi: "Khoảng cách giữa hai xung `Arows_read` liên tiếp trên dạng sóng đo được là bao nhiêu chu kỳ?"*
* **Trả lời:**
  * Khoảng cách lý thuyết bằng 1 giao dịch dài cộng 3 giao dịch ngắn:
    $$145 + (3 \times 110) = 475\text{ chu kỳ}$$
  * Trên dạng sóng thực tế đo được là **454 chu kỳ** do công cụ tối ưu hóa gối đầu các chu kỳ chuyển tiếp giữa các lần gọi hàm.

#### Câu 35: *Thầy chỉ vào Hình 6 hỏi: "Tín hiệu `ap_start` có trạng thái như thế nào trong suốt quá trình chạy?"*
* **Trả lời:**
  * Tín hiệu `ap_start` giữ liên tục ở **mức logic 1** từ đầu đến cuối sau khi tín hiệu Reset nhả ra.
  * Điều này chứng tỏ hệ thống chủ liên tục cấp lệnh cho bộ tăng tốc chạy ở chế độ toàn tải, không có thời gian nghỉ rảnh rỗi giữa các khối.

#### Câu 36: *Thầy chỉ vào Hình 6 hỏi: "Tín hiệu `ap_done` và `ap_ready` phát ra bao nhiêu xung trên toàn bộ dạng sóng?"*
* **Trả lời:**
  * Cả hai tín hiệu phát ra **đúng 16 xung nhọn** (mỗi xung có độ rộng đúng 1 chu kỳ clock).
  * Mỗi xung đánh dấu thời điểm một khối ma trận con $8 \times 8$ được tính xong hoàn chỉnh và xuất thành công ra luồng `ABpartial`.

#### Câu 37: *Thầy hỏi: "Mốc thời gian kết thúc toàn bộ mô phỏng trên dạng sóng là bao nhiêu?"*
* **Trả lời:**
  * Mốc thời gian kết thúc là **$18.32\ \mu\text{s}$** kể từ khi tín hiệu Reset nhả ra, tương ứng với 1,832 chu kỳ xung nhịp $100\text{ MHz}$.

#### Câu 38: *Thầy hỏi: "Tín hiệu `Bcols_read` hoạt động như thế nào trên dạng sóng so với `Arows_read`?"*
* **Trả lời:**
  * Khác với `Arows_read` chỉ bật 4 lần, tín hiệu `Bcols_read` **bật lên ở cả 16 giao dịch khối**.
  * Vì mỗi khối kết quả đều cần một dải dọc mới của ma trận $B$, nên luồng $B$ bắt buộc phải được đọc liên tục ở mọi khối mà không thể tái sử dụng như $A$.

#### Câu 39: *Thầy hỏi: "Dạng sóng có xuất hiện chu kỳ treo (Stall) hay xung đột bus nào không?"*
* **Trả lời:**
  * Hoàn toàn không. Dạng sóng hiển thị luồng dữ liệu chạy mượt mà, các giao dịch nối đuôi nhau liên tục, chứng minh cơ chế bắt tay FIFO và độ sâu hàng đợi stream được cấu hình hoàn toàn chuẩn xác.

#### Câu 40: *Thầy hỏi: "Nếu tín hiệu `Arows_read` bật lên ở cả 16 lần thì em kết luận gì về thiết kế?"*
* **Trả lời:**
  * Nếu bật cả 16 lần thì chứng tỏ điều kiện `counter == 0` đã bị viết sai hoặc mảng $A$ không được khai báo `static`, dẫn đến phần cứng không lưu đệm được dữ liệu và cơ chế tái sử dụng dải ngang bị thất bại.

---

## CHUYÊN ĐỀ 5: VẤN ĐÁP TRÊN HÌNH 7 & TÍCH HỢP HỆ THỐNG SOC TRÊN VIVADO (CÂU 41 - 50)

#### Câu 41: *Thầy chỉ vào Hình 7 (Sơ đồ Block Design trên Vivado) hỏi: "Nhìn vào sơ đồ hệ thống này, em phân tích kiến trúc SoC tổng thể xem gồm những khối nào?"*
* **Trả lời:**
  * Thưa thầy, Hình 7 là hệ thống **Hybrid SoC** tích hợp trên Vivado gồm 5 khối chính:
    1. **ZYNQ7 Processing System (PS7):** Vi xử lý ARM Cortex-A9 đóng vai trò điều khiển trung tâm.
    2. **Bộ tăng tốc phần cứng (`blockmatmul_0`):** Lõi HLS thực hiện nhân ma trận.
    3. **Hai bộ điều khiển AXI DMA (`axi_dma_0` và `axi_dma_1`):** Truyền dữ liệu tốc độ cao giữa DDR RAM và bộ tăng tốc.
    4. **Bộ chuyển mạch bộ nhớ (`axi_mem_intercon`):** Gom các kênh DMA vào cổng bộ nhớ hiệu năng cao.
    5. **Mạng bus điều khiển (`ps7_0_axi_periph`):** Bus AXI-Lite để CPU cấu hình các khối.

#### Câu 42: *Thầy chỉ vào Hình 7 hỏi: "Tại sao trong sơ đồ lại cần tới 2 khối AXI DMA (`axi_dma_0` và `axi_dma_1`) mà không dùng 1 khối?"*
* **Trả lời:**
  * Vì thuật toán yêu cầu nạp đồng thời 2 luồng dữ liệu độc lập là `Arows` và `Bcols`.
  * Mỗi khối AXI DMA chuẩn chỉ có 1 kênh đọc MM2S.
  * Việc sử dụng 2 khối DMA chạy song song cho phép đọc đồng thời ma trận A và ma trận B từ hai vùng nhớ DDR khác nhau và bơm song song vào bộ tăng tốc, đảm bảo mạch tính toán không bị nghẽn dữ liệu đầu vào.

#### Câu 43: *Thầy hỏi: "Cơ chế 'Full-Loop DMA' trong thiết kế hoạt động như thế nào?"*
* **Trả lời:**
  * Là chu trình dữ liệu khép kín:
    1. Chiều đi (MM2S): Hai DMA đọc ma trận A, B từ DDR RAM chuyển thành luồng AXI-Stream bơm vào IP HLS.
    2. Chiều tính toán: IP HLS xử lý tăng tốc trên FPGA.
    3. Chiều về (S2MM): Kênh S2MM của DMA thu gom ma trận kết quả từ luồng ra và ghi thẳng ngược lại DDR RAM.
    4. CPU chỉ việc đọc kết quả từ DDR RAM sau khi nhận cờ báo hoàn thành từ DMA.

#### Câu 44: *Thầy chỉ vào cổng `S_AXI_HP0` trên Zynq PS hỏi: "Cổng này là cổng gì? Tại sao các khối DMA lại nối vào cổng này mà không nối vào `M_AXI_GP0`?"*
* **Trả lời:**
  * `S_AXI_HP0` là cổng **Slave AXI High-Performance** 64-bit của PS.
  * Cổng này nối trực tiếp vào bộ điều khiển DDR Controller qua các bộ đệm FIFO tốc độ cao, hỗ trợ burst transfer với băng thông cực lớn (> 1 GB/s).
  * Không nối vào cổng GP vì cổng GP là cổng Master đa dụng băng thông thấp dùng cho CPU điều khiển thanh ghi, nếu truyền mảng dữ liệu lớn qua GP sẽ làm nghẽn CPU nghiêm trọng.

#### Câu 45: *Thầy hỏi: "Khối `axi_mem_intercon` đóng vai trò gì trong sơ đồ Hình 7?"*
* **Trả lời:**
  * Nó đóng vai trò là một **bộ dồn kênh và phân xử bus (Crossbar Switch)**:
    * Gom 3 luồng Master AXI (2 kênh đọc MM2S và 1 kênh ghi S2MM từ 2 DMA) để nối vào duy nhất 1 cổng Slave `S_AXI_HP0` của Zynq PS.
    * Giải mã địa chỉ và phân xử ưu tiên để các kênh truyền nhận không bị xung đột.

#### Câu 46: *Thầy hỏi: "Mạng bus `ps7_0_axi_periph` dùng chuẩn giao tiếp gì và làm nhiệm vụ gì?"*
* **Trả lời:**
  * Dùng chuẩn **AXI4-Lite**.
  * Làm nhiệm vụ truyền tín hiệu điều khiển (Control Plane): CPU ARM phát lệnh qua cổng `M_AXI_GP0` để cấu hình địa chỉ thanh ghi, kích thước gói tin cho 2 bộ DMA và gửi lệnh kích hoạt `ap_start` cho IP HLS.

#### Câu 47: *Thầy hỏi: "Các đường ngắt (Interrupts) từ 2 bộ DMA và IP HLS được nối về đâu trên Zynq PS?"*
* **Trả lời:**
  * Được nối về cổng ngắt chia sẻ **`IRQ_F2P`** (Fabric-to-Processor) của Zynq PS thông qua khối ghép ngắt **Concat IP**.
  * Khi DMA truyền xong hoặc IP tính xong, ngắt sẽ báo về nhân ARM để CPU xử lý tức thời mà không cần chạy vòng lặp thăm dò (polling).

#### Câu 48: *Thầy hỏi: "Độ rộng dữ liệu của các đường luồng Stream vào IP `blockmatmul_0` là bao nhiêu bit?"*
* **Trả lời:**
  * Với cấu hình $B=8$ và dữ liệu số nguyên 32-bit:
    $$\text{Độ rộng} = 8 \times 32\text{ bits} = \mathbf{256\text{ bits}}$$
  * Hai bộ DMA cấu hình kênh Stream độ rộng lớn để bơm liên tục 256-bit dữ liệu vào IP mỗi chu kỳ xung nhịp.

#### Câu 49: *Thầy hỏi: "Bộ điều khiển reset `rst_ps7_0_100M` trong sơ đồ làm nhiệm vụ gì?"*
* **Trả lời:**
  * Tiếp nhận tín hiệu reset thô từ PS (`FCLK_RESET0_N`) và xung nhịp $100\text{ MHz}$, sinh ra các tín hiệu reset đồng bộ cho bus AXI (`interconnect_aresetn`) và các khối ngoại vi (`peripheral_aresetn`), triệt tiêu hiện tượng bất định trạng thái (Metastability) khi khởi động.

#### Câu 50: *Thầy hỏi: "Trong hệ thống này, nhân ARM Cortex-A9 đóng vai trò gì chính?"*
* **Trả lời:**
  * ARM đóng vai trò là **nhà quản lý hệ thống (Host Controller)**: cấp phát vùng nhớ DDR, khởi tạo dữ liệu ma trận, cấu hình bộ điều khiển DMA, kích hoạt phần cứng chạy và thu thập kết quả, không trực tiếp tham gia vào các phép tính nhân ma trận nặng nhọc.

---

## CHUYÊN ĐỀ 6: VẤN ĐÁP TRÊN HÌNH 8, HÌNH 9, BÁO CÁO TIMING & POWER (CÂU 51 - 60)

#### Câu 51: *Thầy chỉ vào Hình 8 (Báo cáo Timing sau Implementation) hỏi: "Nhìn vào bảng này, chỉ số WNS đạt bao nhiêu? Con số này nói lên điều gì?"*
* **Trả lời:**
  * Thưa thầy, chỉ số **WNS (Worst Negative Slack) đạt $+0.320\text{ ns}$**.
  * **Ý nghĩa:**
    * WNS mang giá trị dương khẳng định mạch **đáp ứng hoàn toàn chuẩn định thời về thời gian thiết lập (Setup Time)**.
    * Đường truyền dữ liệu trễ nhất trong toàn bộ 31,186 endpoints vẫn đến đích sớm hơn cạnh xung nhịp yêu cầu $0.320\text{ ns}$, triệt tiêu nguy cơ vi phạm định thời ở tần số $100\text{ MHz}$.

#### Câu 52: *Thầy chỉ vào Hình 8 hỏi: "Chỉ số WHS đạt $+0.008\text{ ns}$ và Failing Endpoints = 0 có ý nghĩa gì?"*
* **Trả lời:**
  * **WHS (Worst Hold Slack) = $+0.008\text{ ns} > 0$:** Đảm bảo tín hiệu giữ mức ổn định an toàn sau sườn xung nhịp, triệt tiêu nguy cơ tranh chấp dữ liệu (Race Condition).
  * **Failing Endpoints = 0 trên 31,186 điểm:** Xác nhận $100\%$ các đường dẫn dữ liệu trong toàn bộ vi mạch đều đạt chuẩn định thời, không có bất kỳ điểm lỗi nào.

#### Câu 53: *Thầy hỏi: "Nếu chỉ số WNS bị âm ($WNS < 0$) thì hệ thống gặp sự cố gì trên phần cứng thực tế?"*
* **Trả lời:**
  * Mạch bị lỗi thời gian thiết lập (Setup Time Violation).
  * Tín hiệu không kịp truyền đến Flip-Flop trước khi xung clock ập tới, dẫn đến dữ liệu chốt bị sai lệch hoặc rơi vào trạng thái bất định (Metastability), làm phần cứng tính toán ra kết quả rác hoặc bị treo mạch.

#### Câu 54: *Thầy chỉ vào Hình 8 hỏi: "Chỉ số WPWS đạt $+3.750\text{ ns}$ đo lường cái gì?"*
* **Trả lời:**
  * **WPWS (Worst Pulse Width Slack):** Đo lường độ dôi độ rộng xung nhịp.
  * Giá trị dương $+3.750\text{ ns}$ đảm bảo các xung clock phân phối tới các Flip-Flop và khối BRAM giữ được độ rộng chuẩn tắc, không bị méo dạng hay suy hao trên cây xung nhịp (Clock Tree).

#### Câu 55: *Thầy chỉ vào Hình 9 (Báo cáo Power sau Implementation) hỏi: "Nhìn vào ảnh này, tổng công suất tiêu thụ trên chip là bao nhiêu? Phân bổ giữa công suất động và tĩnh ra sao?"*
* **Trả lời:**
  * Thưa thầy, tổng công suất tiêu thụ trên chip là **$1.762\text{ W}$**:
    * **Công suất động (Dynamic Power):** Chiếm áp đảo **$92\%$** ($1.619\text{ W}$).
    * **Công suất tĩnh (Static/Quiescent Power):** Chiếm **$8\%$** ($0.143\text{ W}$, năng lượng rò rỉ cơ bản của đế bán dẫn).

#### Câu 56: *Thầy chỉ vào Hình 9 hỏi: "Tại sao khối xử lý trung tâm PS7 lại chiếm tới 94% công suất động (1.573 W)?"*
* **Trả lời:**
  * Vì khối PS7 chứa nhân kép ARM Cortex-A9 chạy ở tần số rất cao ($667\text{ MHz}$), mạch PLL, bộ nhớ đệm Cache L1/L2 và bộ điều khiển DDR RAM liên tục duy trì hoạt động và điều phối hệ thống. Hoạt động chuyển mạch tần số cao này tiêu tốn phần lớn năng lượng của chip.

#### Câu 57: *Thầy chỉ vào Hình 9 hỏi: "Tại sao phần cứng PL có tới 192 khối DSP và 8 BRAM mà công suất tiêu thụ lại ghi nhận cực thấp (< 1%, DSP < 0.001 W)?"*
* **Trả lời:**
  * Thưa thầy, vì công cụ Vivado thực hiện phân tích công suất sau Implementation ở chế độ **ước lượng không có vector hoạt động (Vectorless / Static Estimation)**.
  * Ở trạng thái này, công cụ giả định luồng dữ liệu DMA chưa thực sự được kích hoạt bơm liên tục ở mức toàn tải nên hoạt độ chuyển mạch (switching activity) được tính ở mức tối thiểu. Khi nạp mạch thực tế chạy toàn tải, công suất của DSP và BRAM sẽ tăng lên nhưng vẫn nằm an toàn trong giới hạn tản nhiệt của chip.

#### Câu 58: *Thầy chỉ vào Hình 9 hỏi: "Nhiệt độ tiếp giáp $T_j = 45.3^\circ\text{C}$ và Thermal Margin = $39.7^\circ\text{C}$ có ý nghĩa an toàn thế nào?"*
* **Trả lời:**
  * Nhiệt độ hoạt động tối đa cho phép của chip thương mại Zynq-7000 là **$85^\circ\text{C}$**.
  * Nhiệt độ thực tế đạt $45.3^\circ\text{C}$ (ở nhiệt độ phòng $25^\circ\text{C}$).
  * Biên độ an toàn nhiệt đạt tới gần $40^\circ\text{C}$ (tương đương chip có thể chịu đựng thêm $3.4\text{ W}$ công suất nhiệt gia tăng mà không cần quạt tản nhiệt cưỡng bức), khẳng định thiết kế cực kỳ an toàn về mặt nhiệt học.

#### Câu 59: *Thầy hỏi: "Nếu muốn đo đạc công suất chính xác hơn nữa trên phần cứng thực tế thì nhóm sẽ dùng phương pháp gì?"*
* **Trả lời:**
  * Sử dụng các chip cảm biến đo dòng/áp tích hợp sẵn trên bo mạch ZC702 (qua giao tiếp bus PMBus / I2C) để đọc công suất tiêu thụ thời gian thực trên các đường nguồn $V_{\text{CCINT}}$ và $V_{\text{CCAUX}}$ khi DMA đang truyền dữ liệu ma trận toàn tải.

#### Câu 60: *Thầy hỏi: "Mức tiêu thụ công suất tổng thể 1.762 W có phù hợp cho các ứng dụng nhúng trên thiết bị di động / drone không?"*
* **Trả lời:**
  * Hoàn toàn phù hợp. Mức công suất dưới $2\text{ W}$ cho một hệ thống SoC vừa có vi xử lý ARM vừa có bộ tăng tốc phần cứng đạt thông lượng $0.64\text{ TMACs/s}$ là mức tiêu thụ năng lượng tối ưu, hoàn toàn khả thi để cấp nguồn bằng pin trên các hệ thống nhúng di động.

---

## CHUYÊN ĐỀ 7: PHÂN TÍCH ĐỊNH LƯỢNG SỐ LIỆU BẢNG 1, 3, 4 & TOÁN HỌC (CÂU 61 - 70)

#### Câu 61: *Thầy hỏi: "Tại sao phép nhân 2 ma trận $N \times N$ lại cần đúng $N^3$ phép nhân và $N^2(N-1)$ phép cộng?"*
* **Trả lời:**
  * Ma trận kết quả có $N \times N = N^2$ phần tử.
  * Mỗi phần tử là tích vô hướng của 1 hàng $N$ số với 1 cột $N$ số:
    * Cần $N$ phép nhân giữa các cặp số.
    * Gom $N$ tích đó lại cần $N - 1$ dấu cộng (phép cộng).
  * Nhân lên cho toàn bộ $N^2$ phần tử:
    * Tổng phép nhân: $N^2 \times N = \mathbf{N^3}$.
    * Tổng phép cộng: $N^2 \times (N - 1) = \mathbf{N^2(N - 1)}$.

#### Câu 62: *Thầy chỉ vào Bảng 1 hỏi: "Tại sao một phép nhân số nguyên 32-bit `int` lại tiêu tốn tới 3 khối DSP48E1 trên chip ZC702?"*
* **Trả lời:**
  * Bộ nhân phần cứng của khối DSP48E1 trên Zynq-7000 chỉ hỗ trợ tối đa kích thước **$18 \times 25\text{-bit}$**.
  * Khi nhân 2 số nguyên 32-bit ($A_{32} \times B_{32}$), ta phải phân rã mỗi số thành 2 nửa 16-bit. Công cụ Vivado HLS tối ưu hóa bằng cách **ghép tầng (cascade) đúng 3 khối DSP48E1** để thực hiện trọn vẹn phép nhân 32-bit.
  * Vì vậy, với khối $B=8$ có $64\text{ bộ nhân}$, số DSP tiêu tốn là $64 \times 3 = \mathbf{192\text{ khối DSP48E1}}$.

#### Câu 63: *Thầy chỉ vào Bảng 4 hỏi: "'Tính bất biến tài nguyên' được thể hiện như thế nào khi $N$ tăng từ 8 lên 32?"*
* **Trả lời:**
  * Thưa thầy, nhìn vào cột DSP48E1 và BRAM_18K của Bảng 4:
    * Với khối $B=4$: Khi $N$ tăng từ $8 \rightarrow 16 \rightarrow 32$, số DSP giữ **cố định tuyệt đối ở 48 khối**, BRAM giữ **cố định ở 4 khối**.
    * Với khối $B=8$: Số DSP giữ **cố định ở 192 khối**, BRAM giữ **cố định ở 8 khối**.
  * Mức tiêu thụ tài nguyên phần cứng không hề tăng theo $N$, chứng minh kiến trúc có thể mở rộng tính toán cho các ma trận cực lớn mà không lo tràn chip.

#### Câu 64: *Thầy chỉ vào Bảng 4 hỏi: "Tại sao khi $N$ nhân đôi ($8 \rightarrow 16 \rightarrow 32$), số lượng Flip-Flop chỉ tăng đúng 3 thanh ghi (4582 $\rightarrow$ 4585 $\rightarrow$ 4588)?"*
* **Trả lời:**
  * Khi $N$ nhân đôi, biến đếm vòng lặp $k$ cần mở rộng thêm đúng **1 bit nhị phân** ($\log_2(8)=3$, $\log_2(16)=4$, $\log_2(32)=5$).
  * Biến đếm $k$ này xuất hiện ở đúng **3 vị trí phần cứng**:
    1. Bộ đếm vòng lặp `LOAD_A_COL`.
    2. Bộ đếm vòng lặp `PARTIAL_SUM_K`.
    3. Thanh ghi trạng thái của máy FSM.
  * Mỗi vị trí tăng 1 bit thanh ghi, tổng cộng tăng đúng $3 \times 1 = \mathbf{3\text{ Flip-Flops}}$!

#### Câu 65: *Thầy chỉ vào Bảng 3 hỏi: "Tại sao hiệu số $\Delta = \text{Latency}_{\max} - \text{Latency}_{\min}$ luôn bằng $N + 3$ chu kỳ?"*
* **Trả lời:**
  * Hiệu số $\Delta$ chính là thời gian thực thi của vòng lặp nạp dữ liệu `LOAD_A_COL`:
    * Đọc $N$ gói dữ liệu từ luồng AXI-Stream ở chế độ pipeline $II=1$ mất đúng **$N$ chu kỳ**.
    * Trễ làm đầy đường ống (Pipeline overhead: đọc FIFO, giải mã struct, ghi BRAM) mất thêm **3 chu kỳ**.
    * Do đó: $\Delta = N + 3$ (với $N=8 \rightarrow 11$; $N=16 \rightarrow 19$; $N=32 \rightarrow 35\text{ chu kỳ}$).

#### Câu 66: *Thầy chỉ vào Bảng 4 hỏi: "Giải thích nghịch lý tại sao ở cấu hình $N=32, B=8$, khi chưa tối ưu tốn 10 BRAM mà khi bật Directives lại giảm còn 8 BRAM?"*
* **Trả lời:**
  * Khi chưa tối ưu: Mảng $AB[8][8]$ bị HLS ánh xạ vào 2 khối BRAM, cộng với 8 BRAM của mảng $A$, tổng là **10 BRAM**.
  * Khi bật Directives: Chỉ thị `ARRAY_PARTITION` chuyển toàn bộ mảng $AB[8][8]$ sang **thanh ghi Flip-Flop**, giải phóng hoàn toàn 2 khối BRAM này. Mảng $A$ sau khi Reshape nằm vừa khít trong **8 BRAM**. Vì vậy BRAM giảm từ 10 xuống còn 8!

#### Câu 67: *Thầy chỉ vào Bảng 3 hỏi: "Tại sao ở cấu hình Baseline, khi chưa dùng Directives, độ trễ lại tuân theo công thức $\text{Latency} \approx \frac{N^3}{2}$?"*
* **Trả lời:**
  * Vì khi tắt Directives, Vitis HLS tự động kích hoạt tính năng auto-pipelining và cấp phát mặc định **2 bộ nhân 32-bit dùng chung** (tương ứng 6 DSP).
  * Do có 2 bộ nhân hoạt động đồng thời, mỗi chu kỳ xử lý được 2 phép nhân. Tổng số phép nhân là $N^3$ nên số chu kỳ tính toán giảm một nửa: $N^3 / 2$.

#### Câu 68: *Thầy chỉ vào Bảng 3 hỏi: "Tại sao Baseline bật Directives có Latency xấp xỉ bằng $N^2$ mà nhóm không chọn Baseline để phát triển tiếp?"*
* **Trả lời:**
  * Dù Baseline có Latency $\approx N^2$, nhưng **số DSP của Baseline tăng tuyến tính theo $N$**:
    * $N=8$ tốn 24 DSP; $N=16$ tốn 48 DSP; $N=32$ tốn 96 DSP.
    * Khi $N=64$ sẽ tốn 192 DSP, và khi $N=128$ sẽ tốn 384 DSP $\rightarrow$ **vượt quá quỹ 220 DSP của chip XC7Z020**.
  * Baseline không có tính mở rộng phần cứng, trong khi Block Matrix giữ cố định DSP nên vượt trội hơn hoàn toàn.

#### Câu 69: *Thầy hỏi: "Tại sao cấu hình $B=8$ lại đạt hiệu năng cao hơn cấu hình $B=4$?"*
* **Trả lời:**
  * Vì cấu hình $B=8$ có $8 \times 8 = 64$ bộ MAC song song, xử lý được 64 phép tính mỗi chu kỳ.
  * Cấu hình $B=4$ chỉ có $4 \times 4 = 16$ bộ MAC song song, xử lý được 16 phép tính mỗi chu kỳ.
  * Do đó, cấu hình $B=8$ tính toán nhanh gấp 4 lần cấu hình $B=4$, đổi lại tiêu tốn 192 DSP so với 48 DSP.

#### Câu 70: *Thầy hỏi: "Thông lượng tính toán cực đại (Peak Compute Throughput) của lõi $B=8$ ở $100\text{ MHz}$ là bao nhiêu?"*
* **Trả lời:**
  * Mỗi chu kỳ xung nhịp thực hiện 64 phép nhân và 64 phép cộng (tương đương 128 thao tác số học - 128 OPs).
  * Ở tần số $100\text{ MHz}$ ($10^8\text{ chu kỳ/giây}$):
    $$\text{Thông lượng} = 128 \times 10^8 = \mathbf{12.8\text{ GFLOPs (hoặc GOPs)}}$$
    (Tương đương $6.4\text{ GMACs/s}$).

---

## CHUYÊN ĐỀ 8: PHẢN BIỆN, SO SÁNH, KHẮC PHỤC SỰ CỐ & HƯỚNG TƯƠNG LAI (CÂU 71 - 80)

#### Câu 71: *Thầy hỏi: "Nếu em muốn tăng kích thước khối lên $B = 16$ trên chip ZC702 hiện tại thì có làm được không? Tại sao?"*
* **Trả lời:**
  * **Nếu giữ nguyên kiểu dữ liệu 32-bit `int` thì KHÔNG làm được**: Vì $B=16$ cần $16^2 = 256\text{ bộ nhân}$, tiêu tốn $256 \times 3 = \mathbf{768\text{ khối DSP}}$, vượt xa con số 220 DSP của chip.
  * **Giải pháp để làm được:** Chuyển đổi kiểu dữ liệu sang **số dấu phẩy tĩnh độ rộng thấp (`ap_int<8>` hoặc `ap_int<16>`)**. Mỗi phép nhân chỉ tốn 1 DSP hoặc ghép 2 phép nhân vào 1 DSP, khi đó $B=16$ chỉ tốn từ 128 đến 256 DSP, hoàn toàn triển khai vừa vặn trên chip ZC702.

#### Câu 72: *Thầy hỏi: "So sánh bộ tăng tốc FPGA này với việc chạy ma trận trên GPU (Graphics Processing Unit), giải pháp nào tốt hơn?"*
* **Trả lời:**
  * **GPU:** Vượt trội về thông lượng tính toán thô khi ma trận có kích thước khổng lồ ($N \ge 2048$), nhưng tiêu tốn công suất rất lớn ($150\text{W} - 350\text{W}$) và độ trễ khởi tạo lệnh cao.
  * **FPGA SoC (Đề tài):** Tiêu thụ công suất cực thấp ($1.76\text{W}$), độ trễ xác định mức nano-giây, giao tiếp trực tiếp phần cứng theo thời gian thực (Deterministic Latency), lý tưởng cho các hệ thống nhúng biên (Edge AI, xe tự hành, drone).

#### Câu 73: *Thầy hỏi: "Nếu nạp bitstream lên bo mạch ZC702 thực tế mà DMA không truyền được dữ liệu, em sẽ debug theo các bước nào?"*
* **Trả lời:**
  1. Kiểm tra đèn LED **DONE** trên mạch xem Bitstream đã nạp thành công vào PL chưa.
  2. Kiểm tra thanh ghi trạng thái của DMA (`DMA_SR`): xem cờ `Halted`, `Idle` hay `Error` (lỗi giải mã địa chỉ hoặc lỗi alignment bộ nhớ).
  3. Kiểm tra địa chỉ bộ nhớ cấp phát cho ma trận: phải đảm bảo đã được Flush/Invalidate bộ nhớ đệm Cache (`Xil_DCacheFlushRange`) để dữ liệu từ CPU thực sự được ghi xuống DDR RAM trước khi DMA đọc.
  4. Bật công cụ phần cứng **ILA (Integrated Logic Analyzer)** của Vivado để bắt dạng sóng tín hiệu `TVALID/TREADY` trên bus AXI-Stream thực tế.

#### Câu 74: *Thầy hỏi: "Tại sao phải gọi hàm `Xil_DCacheFlushRange()` trước khi kích hoạt DMA truyền dữ liệu?"*
* **Trả lời:**
  * Vì CPU ARM có bộ nhớ đệm Cache L1/L2. Khi CPU khởi tạo ma trận, dữ liệu mới có thể vẫn đang nằm trong Cache mà chưa kịp ghi xuống DDR RAM vật lý.
  * Trong khi bộ điều khiển DMA lại đọc dữ liệu trực tiếp từ DDR RAM.
  * Nếu không gọi hàm Flush Cache, DMA sẽ đọc phải dữ liệu rác cũ trong RAM, dẫn đến kết quả nhân ma trận bị sai lệch hoàn toàn.

#### Câu 75: *Thầy hỏi: "Trong bài báo cáo, nhóm đã kiểm chứng đến bước nào và bước nào dự kiến làm tiếp theo?"*
* **Trả lời:**
  * **Đã hoàn thành xuất sắc:** Thiết kế HLS, tối ưu hóa Directives, kiểm chứng C Simulation, C/RTL Cosimulation, phân tích dạng sóng, đóng gói IP Core, tích hợp hệ thống SoC với Full-Loop DMA trên Vivado, đạt chuẩn định thời Implementation và phân tích công suất.
  * **Bước tiếp theo:** Nạp file Bitstream lên kit vật lý ZC702, viết ứng dụng nhúng trên nền tảng Linux/Bare-metal để đo đạc thời gian thực tế bằng bộ đếm phần cứng và thử nghiệm với kiểu dữ liệu 8-bit/16-bit.

#### Câu 76: *Thầy hỏi: "Kiến trúc nhân ma trận theo khối này có ứng dụng trực tiếp được vào mạng nơ-ron học sâu (Deep Learning) không? Ở tầng nào?"*
* **Trả lời:**
  * Có thể ứng dụng trực tiếp và là thành phần cốt lõi:
    1. **Tầng Fully-Connected (Dense Layer):** Bản chất là phép nhân ma trận trọng số với vector ngõ vào ($Y = W \cdot X$).
    2. **Tầng Convolutional (Conv2D):** Sau khi trải ma trận bằng thuật toán `im2col`, phép tích chập ảnh được chuyển đổi hoàn toàn thành phép nhân hai ma trận lớn (GEMM). Bộ tăng tốc `blockmatmul` có thể đảm nhận vai trò lõi tính toán tăng tốc cho cả hai tầng này.

#### Câu 77: *Thầy hỏi: "Điểm yếu lớn nhất của đề tài này là gì và nhóm đề xuất khắc phục thế nào?"*
* **Trả lời:**
  * Điểm hạn chế hiện tại là hệ thống đang sử dụng kiểu dữ liệu số nguyên 32-bit (`int`), dẫn đến việc tiêu tốn 3 khối DSP trên 1 phép nhân và chiếm tới $87\%$ quỹ DSP của chip ở khối $B=8$.
  * Nhóm đề xuất khắc phục bằng cách áp dụng kỹ thuật **lượng tử hóa (Quantization)** sang kiểu số dấu phẩy tĩnh 8-bit (`ap_int<8>`) chuẩn công nghiệp (INT8), giúp giảm 3 lần số DSP và cho phép mở rộng kích thước khối lên $B=16$.

#### Câu 78: *Thầy hỏi: "Em hãy nêu sự khác biệt giữa AXI4-Lite và AXI4-Stream trong hệ thống của nhóm?"*
* **Trả lời:**
  * **AXI4-Lite:** Bus có địa chỉ, truyền từng từ 32-bit đơn lẻ, dùng cho đường dẫn điều khiển (CPU ghi thanh ghi cấu hình DMA và IP HLS).
  * **AXI4-Stream:** Bus không dùng địa chỉ, truyền dữ liệu luồng tốc độ cao không giới hạn kích thước gói qua cơ chế bắt tay `TVALID/TREADY`, độ rộng 256-bit, dùng cho đường dẫn dữ liệu ma trận (truyền các vector `blockvec`).

#### Câu 79: *Thầy hỏi: "Nếu ma trận có kích thước không chia hết cho kích thước khối (ví dụ $N=35, B=8$) thì hệ thống xử lý thế nào?"*
* **Trả lời:**
  * Ta áp dụng kỹ thuật **Zero-Padding (Đệm số 0)**:
    * Mở rộng ma trận lên kích thước là bội số gần nhất của $B$ (từ $35 \times 35$ đệm thêm các số 0 thành $40 \times 40$).
    * Đưa qua bộ tăng tốc $B=8$ để tính toán bình thường.
    * Sau khi tính xong, phần mềm chỉ việc trích xuất đúng ma trận kích thước $35 \times 35$ ở góc trên bên trái và bỏ qua phần đệm số 0.

#### Câu 80: *Thầy hỏi: "Nếu chỉ được dùng 1 câu ngắn gọn nhất để thuyết phục hội đồng về giá trị đề tài của nhóm, em sẽ nói gì?"*
* **Trả lời:**
  * *"Thưa thầy và hội đồng, đề tài của nhóm đã giải quyết triệt để nút thắt nghẽn cổ chai von Neumann và sự bùng nổ tài nguyên của phép nhân ma trận bằng kiến trúc phân khối tích ngoài tương thích luồng dữ liệu, giúp **tiết kiệm 75% băng thông đọc RAM, cố định tài nguyên phần cứng bất kể ma trận mở rộng**, và tăng tốc độ xử lý lên tới **15.9 lần** trên một hệ thống SoC hoàn chỉnh sẵn sàng cho các ứng dụng Edge AI thời gian thực."*

---

# PHẦN 3: BẢNG TRA CỨU ĐỐI CHIẾU HÌNH ẢNH & BẢNG BIỂU TRONG BÁO CÁO

Để tiện lật tài liệu và trả lời ngay khi giảng viên chỉ vào báo cáo:

| Ký hiệu | Tên gọi trong Báo cáo | Vị trí trang / ngữ cảnh | Điểm nhấn cần nhớ |
| :--- | :--- | :--- | :--- |
| **Bảng 1** | Tài nguyên phần cứng SoC ZC702 | Mục 2.1 | 220 DSP, 280 BRAM_18K, 106.4K FF, 53.2K LUT. 1 int32 tốn 3 DSP. |
| **Hình 1** | Sơ đồ kiến trúc phân khối `blockmatmul` | Mục 2.3 | Tích ngoài (Outer product), dải ngang $A$, dải dọc $B$, mảng tích lũy $AB$. |
| **Hình 2** | Đoạn code C++ của `blockmatmul` | Mục 2.3 | `counter == 0`, `hls::stream<blockvec>`, `static A[B][N]`. |
| **Bảng 2** | Bảng các chỉ thị tối ưu hóa (Pragmas)| Mục 2.4 | Reshape dim=1, Partition dim=0, Pipeline II=1, Unroll. |
| **Hình 3** | Minh họa cơ chế gộp `array_reshape` | Mục 2.4 | Gộp cột thành từ rộng 256-bit, đọc 1 chu kỳ. |
| **Hình 4** | Kết quả mô phỏng C Synthesis | Mục 3.2 | Latency 110-145 (avg 128), 192 DSP, 8 BRAM, clock $8.69\text{ ns}$. |
| **Hình 5** | Kết quả mô phỏng C/RTL Cosimulation | Mục 3.2 | Status Pass, tổng 1,832 chu kỳ ($18.32\ \mu\text{s}$), avg $114.5\text{ chu kỳ}$. |
| **Hình 6** | Dạng sóng mô phỏng RTL Waveform | Mục 3.2 | `it` chạy 0..F, `Arows_read` bật 4 lần, `ap_done` phát 16 xung nhọn. |
| **Bảng 3** | Bảng tổng hợp kết quả độ trễ | Mục 3.3 | So sánh Baseline vs Block, tốc độ tăng 15.9x ở $N=32, B=8$. |
| **Bảng 4** | Bảng tổng hợp tối ưu phần cứng | Mục 3.3 | Bất biến DSP=192, BRAM=8; FF chỉ tăng 3 thanh ghi khi $N$ nhân đôi. |
| **Hình 7** | Thiết kế hệ thống SoC trên Vivado | Mục 3.5 | Zynq PS + IP HLS + 2 DMA + AXI Interconnect + cổng `S_AXI_HP0`. |
| **Hình 8** | Báo cáo Timing Vivado sau Impl | Mục 3.5 | $WNS = +0.320\text{ ns}$, $WHS = +0.008\text{ ns}$, 0 điểm vi phạm. |
| **Hình 9** | Báo cáo Power Vivado sau Impl | Mục 3.5 | Tổng $1.762\text{ W}$, PS chiếm $94\%$ động ($1.573\text{ W}$), $T_j = 45.3^\circ\text{C}$. |
