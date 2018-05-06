# Điều chỉnh hiệu năng MySQL 5.7 ngay sau khi cài đặt
Blog này cập nhật [Blog của Stephane Combaudon về điều chỉnh hiệu năng MYSQL](https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/), và bao gồm điều 
chỉnh hiệu năng MySQL 5.7 ngay sau khi cài đặt.

Một vài năm trước, Stephane Combaudon đã viết bài viết về [10 cài đặt điều chỉnh hiệu
năng MySQL ngay sai khi cài đặt](https://www.percona.com/blog/2014/01/28/10-mysql-performance-tuning-settings-after-installation/) 
bao gồm các phiên bản cũ hơn (hiện tại) của MySQL: 5.1, 5.5 và 5.6 . Trong bài viết này,
Tôi sẽ tập trung vào những gì để điều chỉnh trong MySQL 5.7 (với việc tập trung vào  InnoDB)

Tin tốt là MySQL 5.7 có giá trị mặc định tốt hơn rất nhiều. Morgan Tocker đã tạo ra
một [trang có danh sách đầy đủ các tính năng trong MySQL 5.7](http://www.thecompletelistoffeatures.com/), 
và nó là một điểm tham chiếu tuyệt vời. Ví dụ, các giá trị sau được cài đặt theo mặc định:
- innodb_file_per_table = ON
- innodb_stats_on_metadata = OFF
- innodb_buffer_pool_instances = 8 (hoặc 1 nếu innodb_buffer_pool_size < 1GB)
- query_cache_type = 0; query_cache_size = 0; (vô hiệu hóa mutex)

Trong MySQL 5.7, chỉ có bốn biến thực sự quan trọng cần được thay đổi. Tuy nhiên, 
Tuy nhiên, có các biến khác của InnoDB và biến toàn cục của MySQL có thể cần được điều chỉnh cho một khối lượng công việc và phần cứng cụ thể.

Để bắt đầu, thêm các cài đặt sau vào my.cnf trong phần [mysqld]. Bạn cần khởi động lại MySQL:

```mysql
[mysqld]
# other variables here
innodb_buffer_pool_size = 1G # (adjust value here, 50%-70% of total RAM)
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 1 # may change to 2 or 0
innodb_flush_method = O_DIRECT
```

| Biến |  Giá trị | 
| ------------- |:-------------:| 
| innodb_buffer_pool_size |  Bắt đầu với 50% 70% tổng RAM. Không nhất thiết phải lớn hơn kích thước cơ sở dữ liệu|  
| innodb_flush_log_at_trx_commit | * 1   (Mặc định) * 0/2 hiệu suất cao hơn,kém tin cậy hơn)|  
| innodb_log_file_size |  128M – 2G (không cần thiết phải lớn hơn vùng đệm) |  
| innodb_flush_method |  O_DIRECT (tránh buffering 2 lần) | 

**Tiếp theo là?**

Đó là điểm khởi đầu tốt cho mọi cài đặt mới. Có một số biến khác có thể tăng hiệu năng MySQL cho một khối lượng công việc.
Thông thường, tôi sẽ thiết lập công cụ theo dõi / vẽ đồ thị MySQL (ví dụ, [nền tảng giám sát và quản lý Percona](http://pmmdemo.percona.com/))
 và sau đó kiểm tra bảng điều khiển MySQL để thực hiện điều chỉnh thêm.
 
 **Những gì chúng ta có thể điều chỉnh thêm dựa trên các đồ thị**
 
 Kích thước vùng đệm của InnoDB. Nhìn vào biểu đồ:
 
 ![img](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.49.22-PM.png)
 
 ![img](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.48.13-PM.png)
 
 Như các bạn nhìn thấy, chúng ta có thể có lợi từ việc tăng kích thước vùng đệm InnoDB một chút lên ~ 10G, vì chúng ta có RAM và số trang miễn phí nhỏ so với tổng số vùng đệm.
 
 *Kích thước tệp nhật ký InnoDB*. Nhìn vào biểu đồ
 
 ![img](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-12.43.52-PM.png)
 
 Như chúng ta nhìn thấy ở đây, InnoDB thường ghi 2,26 GB dữ liệu mỗi giờ, vượt quá tổng kích thước của tệp nhật ký (2G).
 Bây giờ chúng ta có thể tăng biến `innodb_log_file_size` và khởi động lại MySQL. 
 Ngoài ra, hãy sử dụng "trạng thái InnoDB của công cụ hiển thị" để [tính toán kích thước tệp nhật ký InnoDB tốt](https://www.percona.com/blog/2008/11/21/how-to-calculate-a-good-innodb-log-file-size/)
 
 **Các biến khác**
 
 Có một số biến InnoDB khác có thể được điều chỉnh thêm:
 
 `innodb_autoinc_lock_mode`
 
 Thiết lập innodb_autoinc_lock_mode = 2 (chế độ xen kẽ) có thể loại bỏ sự cần thiết của khóa AUTO-INC ở mức bảng (và có thể tăng hiệu suất khi các câu lệnh chèn nhiều hàng được sử dụng để chèn các giá trị vào các bảng với khóa chính auto_increment).
 Điều này đòi hỏi `binlog_format = ROW` hoặc `MIXED` (và ROW là mặc định trong MySQL 5.7).

`innodb_io_capacity` và `innodb_io_capacity_max`

Đây là một điều chỉnh nâng cao hơn, và chỉ có ý nghĩa khi bạn đang thực hiện viết quá nhiều (nó không áp dụng cho  đọc, tức là SELECT).
 Nếu bạn thực sự cần phải điều chỉnh nó, phương pháp tốt nhất là biết có bao nhiêu IOPS hệ thống có thể làm.
 Ví dụ, nếu máy chủ có một ổ SSD, chúng ta có thể thiết lập `innodb_io_capacity_max = 6000` và `innodb_io_capacity = 3000` (tối đa 50%). Đó là một ý tưởng tốt để chạy sysbench hoặc bất kỳ công cụ benchmark khác nào để chuẩn hóa thông lượng đĩa.
 
 Nhưng chúng ta có cần phải lo lắng về cài đặt này không? Nhìn vào biểu đồ "[dirty pages](http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dirty_page)" của vùng đệm:
 
 ![img](https://www.percona.com/blog/wp-content/uploads/2016/10/Screen-Shot-2016-10-03-at-7.19.47-PM.png)
 
 Trong trường hợp này, tổng cộng số dirty pages cao, và có vẻ như InnoDB không thể đuổi kịp nó.
 Nếu chúng tôi có hệ thống phụ đĩa nhanh (nghĩa là SSD), chúng tôi có thể hưởng lợi từ việc tăng `innodb_io_capacity` và `innodb_io_capacity_max`.
 
 **Kết luận hoặc TL; DR phiên bản**
 
 Các mặc định mới của MySQL 5.7 tốt hơn nhiều cho các khối lượng công việc có mục đích chung. Đồng thời, chúng ta vẫn cần cấu hình biến InnoDB để tận dụng số lượng RAM trên hộp. Sau khi cài đặt, hãy làm theo các bước sau
 
 1. Thêm biến InnoDB vào my.cnf (như mô tả ở trên) và khởi động lại MySQL
 2. Cài đặt hệ thống giám sát, (ví dụ: nền tảng giám sát và quản lý Percona)
 3. Nhìn vào biểu đồ và xác định xem liệu MySQL có cần được điều chỉnh thêm hay không
 
  
 