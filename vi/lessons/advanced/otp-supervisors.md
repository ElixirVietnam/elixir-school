---
layout: page
title: OTP Supervisors
category: advanced
order: 6
lang: vi
---

Supervisor là các process đặc biệt với chỉ một mục đích: quản lý các process khác. Những supervisor cho phép chúng ta tạo ra các ứng dụng chống chịu lỗi (fault-tolerent) bằng cách tự động khởi động lại các process con khi chúng bị hỏng.

{% include toc.html %}

## Cấu hình

Phép thuật của Supervisor nằm trong hàm `Supervisor.start_link/2`. Ngoài việc khởi động supervisor và các process con, nó cho phép chúng ta định nghĩa chiến thuật cho supervisor để quản lý các process con.

Các process con được định nghĩa bằng cách sử dụng một danh sách, và hàm `worker/3` (được import từ module `Supervisor.Spec`). Hàm `worker/3` nhận vào một module, các đối số, và một tập các tuỳ chọn. Ở bên dưới `worker/3` gọi tới hàm `start_link/3` với các đối số trong quá trình khởi tạo.

Chúng ta sẽ bắt đầu với SimpleQueue trong bài [OTP Concurrency](../../advanced/otp-concurrency):

```elixir
import Supervisor.Spec

children = [
  worker(SimpleQueue, [], [name: SimpleQueue])
]

{:ok, pid} = Supervisor.start_link(children, strategy: :one_for_one)
```

Nếu process của chúng ta bị crash, hoặc bị tắt đi, Supervisor sẽ tự động khởi động lại nó như chưa có điều gì xảy ra.

### Các chiến thuật:

Hiện tại, có bốn chiến thuật khác nhau để khởi động lại các process con:

+ `:one_for_one` - Chỉ khởi động lại process bị fail thôi.

+ `:one_for_all` - Khởi động lại tất cả các process con, nếu có một process bị fail

+ `:rest_for_one` - Khởi động lại process bị fail, và tất cả các process khởi động sau nó.

+ `:simple_one_for_one` - Đây là chiến thuật tốt nhất cho các process con được gắn vào Supervisor một cách động. Supervisor yêu cẩu chỉ chứa duy nhất một process con, nhưng process này có thể được spawn nhiều lần. Chiến thuật này được sử dụng khi bạn muốn khởi động và tắt đi process con một cách động.

### Restart values

Có một vài cách tiếp cận để quản lý việc các process con bỉ hỏng:

+ `:permanent` - Process con luôn luôn được khởi động lại.

+ `:temporary` - Process con không bao giờ được khởi động lại.

+ `:transient` - Process con chỉ dược khởi động lại, nếu như nó bị tắt một cách không bình thường.

Giá trị mặc định là `:permanent`.

### Lồng

Ngoài việc sử dụng với các worker process, chúng ta cũng có thể dùng Supervisor để tạo ra một cây các process. Điểm khác biệt duy nhất là gọi `supervisor/3` thay cho `worker/3`:

```elixir
import Supervisor.Spec

children = [
  supervisor(ExampleApp.ConnectionSupervisor, [[name: ExampleApp.ConnectionSupervisor]]),
  worker(SimpleQueue, [[], [name: SimpleQueue]])
]

{:ok, pid} = Supervisor.start_link(children, strategy: :one_for_one)
```

## Task Supervisor

Task là có một kiểu Supervisor đặc biệt là `Task.Supervisor`. Được thiết kế cho các task động, `Task.Supervisor` sử dụng chiến thuật `:simple_one_for_one` ở bên dưới.

### Cấu hình

Việc thêm vào `Task.Supervisor` là không có gì khác so với các loại supervisors khác:

```elixir
import Supervisor.Spec

children = [
  supervisor(Task.Supervisor, [[name: ExampleApp.TaskSupervisor, restart: :transient]]),
]

{:ok, pid} = Supervisor.start_link(children, strategy: :one_for_one)
```

Điểm khác biệt chính giữa `Supervisor` và `Task.Supervisor` đó là, chiến thuật khởi động lại mặc định là `:temporary` (tức là task sẽ không bao giờ được khởi động lại).

### Supervised Tasks

Với supervisor đã được chạy, chúng ta có thể dùng `start_child/2` để tạo ra một supervised task:

```elixir
{:ok, pid} = Task.Supervisor.start_child(ExampleApp.TaskSupervisor, fn -> background_work end)
```

Nếu task của chúng ta bị crash, nó sẽ được khởi động lại. Điều này đặc biệt hữu dụng khi làm việc với các connection từ bên ngoài hoặc các công việc xử lý dưới nền.
