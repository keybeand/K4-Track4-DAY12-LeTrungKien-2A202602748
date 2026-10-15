# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Lê Trung Kiên |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602748 |
| GitHub | keybeand |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split seed 8422, model seed 8968 (seeds: 8968, 8969, 8970) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.258 | 20.1% | BLOCK |
| IDM | 1.537 | 0.1% | (baseline) |
| EgoMLP | 0.758 .. 0.761 | 18.0 .. 23.4% | BLOCK |
| PercMLP | 0.613 .. 0.619 | 0.5 .. 2.4% | BLOCK |
| PercMLP+ (sau khi sửa) | 0.801 .. 0.816 | 0.3 .. 0.3% | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Tập dữ liệu thô có 992679 dòng thuộc 9993 quỹ đạo. Quá trình kiểm toán phát hiện 383 quỹ đạo bị trùng lặp giống hệt nhau ở các ID cách xa nhau và đã bị loại bỏ, còn lại 9610 quỹ đạo sạch với độ dài từ 70 đến 109 bước (7.0 đến 10.9 giây, dt = 0.1s). Vấn đề nghiêm trọng nhất nằm ở cột gia tốc xe dẫn Acc_LV: độ tương quan tuyến tính giữa cột này với đạo hàm vận tốc d(Speed_LV)/dt chỉ đạt 0.0454 (gần như bằng 0), trong khi xe sau Acc_FAV tương quan đạt 0.9291. Điều này cho thấy cột Acc_LV bị nhiễu cảm biến hoặc tính toán sai và không thể sử dụng để phân tích vật lý. Em đã thay thế bằng việc tính hiệu phân trung tâm từ vận tốc xe dẫn sau khi làm mượt trung bình trượt cửa sổ 5 bước (hàm derive_lead). Dữ liệu bắt buộc phải chia theo quỹ đạo thay vì chia theo từng khung hình vì các khung hình kề nhau trong một chuỗi thời gian có tương quan rất cao; chia theo khung hình sẽ làm rò rỉ trạng thái giữa tập huấn luyện và kiểm thử, dẫn đến việc đánh giá mô hình bị sai lệch.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 khác bit ở toàn bộ 600/600 đoạn nhưng có 0 đoạn vượt dung sai (differ_beyond_tol = 0), sai số tuyệt đối lớn nhất ghi nhận là 2.03e-05 m/s² (thấp hơn nhiều so với atol = 0.001). Nguyên nhân do thứ tự cộng số thực dấu phẩy động không có tính kết hợp, ví dụ (0.1 + 0.2) + 0.3 = 0.6000000000000001 khác 0.1 + (0.2 + 0.3) = 0.6; do đó kiểm thử bit là không thực tế đối với tính toán số học. Khi thêm luật phanh khẩn cấp AEB, có đúng 9/600 đoạn bị lệch lệnh (các đoạn 168, 197, 201, 226, 281, 283, 330, 453, 534) với độ lệch lớn nhất 1.98 m/s². Mở đoạn 168 cho thấy AEB chỉ kích hoạt ở các khung hình có TTC < 1.5 giây khi xe sau đang lao nhanh về phía xe dẫn, còn các khung hình lái xe bình thường lệnh điều khiển vẫn giữ nguyên. Cuối cùng, chế độ phát nhanh nhất (fastest) có 0 khung trễ và 0 khung rớt do bỏ qua deadline thời gian thực. Chế độ thời gian thực (realtime) làm lộ ra 584 khung trễ và 579 khung bị rớt trên 59859 khung, khiến 374/600 đoạn thay đổi kết quả.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản có tốc độ cruise 12.0 m/s, gồm 60 mức khoảng cách từ 2.76 m đến 59.31 m và 45 mức vận tốc tiến gần từ 0.0 đến 12.0 m/s, tổng cộng 2700 ô. Tại độ trễ t_react = 0.0s, có 2.22% số ô là vùng va chạm bất khả thi theo vật lý (ngay cả khi phanh tối đa 9.0 m/s²). Khi độ trễ tăng lên 0.6s, vùng bất khả thi tăng lên 7.44%, tức mở rộng thêm 5.22% diện tích lưới (gấp hơn 3.3 lần). Vùng va chạm của IDM tăng chính xác từ 2.22% lên 7.44%, nghĩa là diện tích va chạm do lỗi bộ điều khiển (attributable area) luôn bằng 0.0% ở cả hai độ trễ. Về độ bao phủ, người lái trong log phanh mạnh nhất ở phân vị 99.9% chỉ đạt 3.62 m/s² (trong khi phanh xe chịu được 9.0 m/s²). Có 8.56% số ô trên lưới đòi hỏi lực phanh vượt quá ngưỡng này của con người. Tập log chỉ chứa 614 sự kiện xe dẫn phanh (chiếm 6.4% quỹ đạo), cho thấy dữ liệu thu thập thực tế chủ yếu là các tình huống lái êm ái và hoàn toàn bỏ trống các ca nguy hiểm sát biên vật lý.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh L2 vòng hở và tỷ lệ va chạm vòng kín trên 2883 quỹ đạo kiểm thử:
- PercMLP: L2 = 0.615 m (Hạng 1) | Va chạm = 1.17% (dải 0.52 .. 2.36%, Hạng 2)
- EgoMLP: L2 = 0.759 m (Hạng 2) | Va chạm = 20.33% (dải 18.04 .. 23.41%, Hạng 4)
- const-vel: L2 = 1.258 m (Hạng 3) | Va chạm = 20.12% (Hạng 3)
- IDM: L2 = 1.537 m (Hạng 4) | Va chạm = 0.14% (chỉ đâm 4/2883 ca, Hạng 1)

Có 4 cặp bị đảo ngược thứ hạng: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP) và (IDM, PercMLP). IDM có L2 cao nhất vì nó điều khiển theo khoảng cách an toàn chứ không cố mô phỏng từng dao động nhỏ của người lái trong log; nhưng khi vào vòng kín, IDM liên tục nhận phản hồi vị trí để giảm tốc hợp lý nên tránh được gần như toàn bộ va chạm. Ngược lại, EgoMLP có L2 tốt hơn const-vel nhờ học được việc duy trì tốc độ mượt mà từ lịch sử bản thân, nhưng nó hoàn toàn không có cảm biến đo khoảng cách gap và vận tốc xe trước. Khi xe dẫn phanh gấp trong 179 ca phanh, EgoMLP không hề biết để xử lý nên đâm tới 47.5% số ca đó, dẫn đến tỷ lệ va chạm tổng thể 20.33%, tệ hơn cả xe chạy đều.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Tỷ lệ va chạm của EgoMLP qua 3 seed huấn luyện biến động từ 18.04% đến 23.41% (seed 8968: 19.53%, seed 8969: 23.41%, seed 8970: 18.04%), độ lệch biên lên tới 5.37%. Khoảng tin cậy Wilson 95% của seed tốt nhất (8970) là [16.68, 19.48]% và của seed xấu nhất (8969) là [21.90, 24.99]%. Hai khoảng tin cậy này hoàn toàn tách rời nhau và không có điểm giao cắt. Nếu chỉ chạy ngẫu nhiên một seed, người thử nghiệm có thể rơi vào seed 8970 và kết luận mô hình chỉ va chạm 18.0%, hoặc rơi vào seed 8969 và báo cáo tỷ lệ va chạm 23.4%. Sự chênh lệch hơn 5% trên cùng một tập dữ liệu và kiến trúc mô hình là rủi ro lớn trong kiểm thử an toàn; nó che giấu tính bất ổn định của trọng số mạng nơ-ron và có thể khiến một hệ thống chưa đạt chuẩn bị kết luận nhầm là an toàn.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh tỷ lệ va chạm tổng thể trên lưới cắt vào (ở t_react = 0.6s):
- const-vel: Xe sau phát lại 77.6% (va chạm sau 0.0%) | Xe sau phản ứng 77.6% (va chạm sau 0.0%)
- IDM: Xe sau phát lại 75.7% (va chạm sau 68.3%) | Xe sau phản ứng 7.4% (va chạm sau 0.0%)
- EgoMLP: Xe sau phát lại 77.5% (va chạm sau 0.0%) | Xe sau phản ứng 77.5% (va chạm sau 0.0%)
- PercMLP: Xe sau phát lại 75.8% (va chạm sau 47.4%) | Xe sau phản ứng 29.9% (va chạm sau 0.0%)

Thời khoảng cách hiệu chỉnh tốt nhất của tác nhân xe sau là T = 1.4s với khoảng cách Wasserstein W1 = 0.36 so với phân bố thời khoảng cách trung vị của con người (1.85s), chân thực hơn nhiều so với xe không phản ứng (W1 = 0.87). IDM bị gán 68.3% va chạm phía sau trong thế giới phát lại vì xe sau chỉ nhại lại log tốc độ cũ mà không quan sát phía trước; khi IDM chủ động phanh gấp để tránh xe cắt vào, xe sau đâm thẳng vào đuôi IDM. Đây là va chạm giả do môi trường mô phỏng thụ động tạo ra. Nếu chỉ dùng xe sau phát lại, IDM trông tệ ngang const-vel (75.7% so với 77.6%). Khi chuyển sang xe sau biết phản ứng, va chạm đâm đuôi của IDM triệt tiêu về 0.0% và tổng va chạm rơi về 7.4% (đúng bằng giới hạn vật lý), làm đảo ngược hoàn toàn nhận định về độ an toàn của thuật toán.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi 5 quy tắc: R2 (va chạm 2.4%, cận trên 3.0% vượt ngưỡng baseline 0.1% + 1.0% = 1.1%), R3 (va chạm ca phanh 7.8%, cận trên 12.7% vượt ngưỡng 1.1% + 5.0% = 6.1%), R4 (va chạm lưới 22.5% vượt ngưỡng 0.0% + 2.0% = 2.0%), R5 (dao động seed 0.5 .. 2.4% > 1.0%) và R6 (va chạm với xe sau 29.9% vượt ngưỡng 7.4%). Hai nguyên nhân gốc là dữ liệu log không có các tình huống xe cắt vào nguy hiểm (gây lỗi R4, R6) và sai số tích lũy dạng covariate shift khi mô hình tự lái vòng kín (gây lỗi R2, R3). Phương pháp sửa gồm bổ sung 197090 cửa sổ từ 20000 kịch bản cắt vào mô phỏng và chạy 2 vòng DAgger (với ~894k cửa sổ) để lấy nhãn IDM trên chính các trạng thái mô hình tự đi vào. Mô hình PercMLP+ sau khi sửa đã qua cổng (R2 cận trên 0.6%, R4 còn 0.1%, R6 còn 7.5%). Tuy nhiên, PercMLP+ không tốt hơn IDM mà chỉ đạt mức không kém hơn (non-inferior). Do PercMLP+ được huấn luyện từ nhãn của IDM, năng lực của nó bị chặn trên bởi chính IDM. Chưa nên phát hành PercMLP+ ngay để thay IDM, mà cần thử nghiệm thêm với dữ liệu có độ trễ truyền động thực tế và kiểm thử trên xe thật ở bãi thử đóng.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn lớn nhất của bài lab đối với quyết định phát hành gồm:
Thứ nhất, bài toán giới hạn trong động học một chiều dọc (1D longitudinal). Mô hình chỉ xét phanh tối đa 9.0 m/s² trên đường thẳng ở tốc độ cruise 12.0 m/s, bỏ qua hoàn toàn chuyển động đánh lái ngang, góc quay vô-lăng và độ trượt lốp; trong thực tế khi xe trước phanh gấp, việc phanh kết hợp chuyển làn né tránh có thể gây mất ổn định thân xe hoặc va chạm với làn bên cạnh.
Thứ hai, hệ thống bỏ qua toàn bộ chuỗi nhận thức cảm biến (perception pipeline). Mô hình nhận trực tiếp giá trị khoảng cách gap và vận tốc chính xác tuyệt đối từ bộ mô phỏng mà không tính độ trễ xử lý 0.6s hay nhiễu cảm biến thực (trong khi thử nghiệm cho thấy nhiễu khoảng cách ±1.0 m và tốc độ ±0.5 m/s có thể làm tăng rủi ro va chạm), trong khi tín hiệu Camera/Radar/LiDAR luôn có độ trễ phát hiện, nhiễu đốm và suy giảm mạnh khi thời tiết xấu.
Thứ ba, các tác nhân giao thông xung quanh thiếu tính tương tác xã hội hai chiều. Xe dẫn trong toàn bộ 2883 quỹ đạo kiểm thử chỉ chạy theo chuỗi log cố định bất kể xe thử nghiệm bám sát ra sao. Trên thực tế, người lái phía trước có thể nhả phanh, tăng tốc hoặc chuyển làn khi nhận thấy xe sau tiến lại quá gần.
Để thu hẹp khoảng cách này, cần tích hợp mô hình vào phần mềm mô phỏng 3D đa tác nhân (như CARLA), đưa luồng nhận thức cảm biến có nhiễu vào vòng lặp kiểm thử (software-in-the-loop), và thực hiện thử nghiệm trên bãi thử khép kín với các mục tiêu giả trước khi đưa lên xe thương mại.
