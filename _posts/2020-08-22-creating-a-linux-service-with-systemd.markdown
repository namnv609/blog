---
layout: post
title: Creating a Linux service with systemd
date: 2020-08-22 20:27:15 +0700
category: Linux
is_editor_choice: true
---

Trước đây mình đã từng viết một bài chia sẻ về việc viết một service bằng init ([**tại đây**](https://viblo.asia/p/write-linux-init-script-bJzKmk9Bl9N)). Nay mình sẽ giới thiệu thêm một cách viết init service nữa là viết cho thằng Systemd nhé. Bài viết này mình đi dịch lại của một tác giả chia sẻ trên Medium. Chúng ta cùng vào làm luôn nhé.

> Mình sẽ bỏ qua phần đề bài của tác giả. Chúng ta sẽ đi luôn vào phần chính cho nhanh

# Creating a Linux service with systemd

Bây giờ, chúng ta sẽ viết một service nho nhỏ sử dụng PHP. Chương trình này sẽ lắng nghe UDP ở cổng 10000 và trả lại tin nhắn đã được chuyển sang dạng [ROT13](https://en.wikipedia.org/wiki/ROT13) (một dạng đảo vị trí các ký tự theo một quy tắc nhất định).

Đầu tiên, chúng ta tạo một file PHP với tên `server.php` có nội dung như sau:

```php
<?php
  $sock = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
  socket_bind($sock, '0.0.0.0', 10000);

  for (;;) {
    socket_recvfrom($sock, $message, 1024, 0, $ip, $port);
    $reply = str_rot13($message);
    socket_sendto($sock, $reply, strlen($reply), 0, $ip, $port);
  }
```

Sau đó, chúng ta khởi động nó lên bằng lệnh:

```
$ php server.php
```

Và bây giờ, chúng ta cùng thử nó trên terminal nhé:

```
$ nc -u 127.0.0.1 10000
Hello, world!
Uryyb, jbeyq!
```

Vâng, nó đã làm việc. Giờ điều chúng ta cần là cho script này (lệnh `php server.php`) chạy mọi lúc, được khởi động lại trong trường hợp xảy ra lỗi và thậm chí máy tính (server) được khởi động lại. Đây là lý do mà chúng ta cần đến Systemd.

## Biến nó thành một service

Đầu tiên, chúng ta cần tạo một file với tên bất kỳ vào thư mục sau (bài viết này sử dụng tên service là `rot13.service`): `/etc/systemd/system/rot13.service` với nội dung sau:

```
[Unit]
Description=ROT13 demo service
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=centos
ExecStart=/usr/bin/env php /path/to/server.php

[Install]
WantedBy=multi-user.target
```

Với script trên thì bạn cần sửa một số chỗ sau:

* Thay đổi tên server sẽ chạy script này ở phần `User=`
* Đổi lại đường dẫn đến file PHP ở `ExecStart=`

Sau khi xong, bây giờ chúng ta có thể bắt đầu service bằng lệnh:

```
systemctl start rot13
```

Và cho service này tự động khởi động cùng hệ thống bằng lệnh:

```
systemctl enable rot13
```

## Chi tiết

Bây giờ service của bạn đã chạy (hy vọng là thế - mình thử cũng ngon rồi nhé :smiley:). Và chúng ta sẽ đi sâu hơn một chút vào các tùy chọn để cấu hình cho một service, đảm bảo rằng nó sẽ luôn hoạt động đúng như chúng ta mong đợi

## Khởi động theo đúng thứ tự

Bạn có thể đã tự hỏi rằng directive `After=` đã làm việc như thế nào không? Vâng, nó chỉ đơn giản rằng service của bạn sẽ được chạy sau khi service liên quan đến network đã sẵn sàng. Ví dụ, nếu chương trình của chúng ta có sử dụng đến MySQL service, thì chúng ta cần phải đảm bảo rằng service của MySQL đã chạy rồi mới chạy service của mình.

```
After=mysqld.service
```

## Khởi động lại khi thoát

Mặc định, systemd sẽ không tự khởi động lại service của chúng ta nếu chương trình không may bị thoát vì một lý do nào đó. Đây là điều chúng ta không muốn cho một service phải luôn được chạy, vì thế chúng ta cần phải báo cho systemd biết rằng service này luôn được khởi động lại khi thoát:

```
Restart=always
```

Bạn cũng có thể sử dụng `on-failure` để systemd chỉ khởi động lại service nếu chương trình của chúng ta bị thoát với status code khác `0`.

Mặc định, systemd sẽ thử khởi động lại service sau 100ms. Bạn có thể xác định chính xác số giây bạn muốn (thay vì 100ms):

```
RestartSec=1
```

### Tránh bẫy: Giới hạn việc khởi động chương trình

Mặc định, khi bạn cấu hình `Restart=always` thì systemd sẽ khởi cố gắng khởi động lại service của chúng ta (không may bị lỗi) 5 lần trong vòng 10s. Sau đó nó sẽ dừng lại việc cố gắng khởi động service của chúng ta mãi mãi. Việc này do hai tùy chọn sau chịu trách nhiệm:

```
StartLimitBurst=5
StartLimitIntervalSec=10
```

Directive `RestartSec` cũng có tác động tới kết quả trên. Nếu bạn đặt là khởi động lại sau 3s thì việc thử 5 lần trong vòng 10s là không thể. Nhưng có cách fix vấn đề này đơn giản hơn là đặt `StartLimitIntervalSec=0`. Điều này có nghĩa là systemd sẽ cố gắng khởi động lại service của chúng ta mãi mãi.

Mặc dù vậy, một ý tưởng hay là đặt `RestartSec` thành ít nhất là 1s để tránh quá tải cho server khi service của chúng ta khởi động có vấn đề.

Thay vào đó, bạn cũng có thể để các cài đặt mặc định và yêu cầu systemd khởi động lại server nếu như nó đạt đến giới hạn (5 lần trong 10s) bằng directive sau:

```
StartLimitAction=reboot
```

# Lời kết

Đến đây là kết thúc bài dịch của mình. Qua bài viết này, bạn có thể thấy việc viết một init service cho Linux bằng Systemd nó đơn giản và dễ sử dụng hơn bài viết trước của mình nhỉ? Kể cả việc cho nó tự khởi động cùng hệ điều hành (startup service) cũng được thực hiện đơn giản hơn. Trong bài viết sau, có thể mình sẽ giới thiệu đến mọi người những directives của thằng Systemd này nhé (nhân tiện mình cũng học luôn :smiley:).

Hẹn gặp lại mọi người trong bài viết sau :wave:


Nguồn bài viết: https://medium.com/@benmorel/creating-a-linux-service-with-systemd-611b5c8b91d6
