---
layout: page
title: Guardian (Cơ bản)
category: Thư viện
order: 1
lang: vi
---

[Guardian](https://github.com/ueberauth/guardian) là một thư viện xác thực danh tính người dùng được sử dụng rộng rãi dựa trên chuẩn [JWT](https://jwt.io/) (Javascript Web Token).

{% include toc.html %}

## JWTs

Một chuẩn JWT cung cấp lượng token phong phú dành cho xác thực danh tính người dùng. Nhiều hệ thống xác thực danh tính một chủ thể tới tài nguyên máy tính, JWTs cho chúng ta các thông tin khác như :

* Ai đã tạo token
* Token đó dùng cho ai
* Hệ thống nào sử dụng token
* Thời điểm issued được tạo
* Thời điểm issue hết hạn

Bổ sung thêm Guardian cung cấp các tính năng tiện lợi :

* Kiểu của token là gì
* Những hành vi nào được làm

Đây là các fields cơ bản trong JWT. Bạn tùy ý thêm bất cứ thông tin nào cần cho ứng dụng của bạn. Nhớ điều là keep it short, là JWT đặt vừa vặn trong HTTP header.

Sự phong phú này cho phép bạn đặt JWTs khắp hệ thống của bạn.

### Sử dụng chúng ở đâu

JWT tokens có thể sử dụng xác thực định danh ở bộ phận bất kỳ của ứng dụng.

* Các ứng dụng Single page
* Các controllers (qua phiên làm việc trình duyệt)
* Các controllers (qua authorization headers - API)
* Các kênh Phoenix
* Các request Service to service
* Inter-process
* Chứng thực bởi bên thứ 3
* Chức năng nhớ tự động
* Các giao diện khác - raw TCP, UDP, CLI, etc

JWT tokens có thể sử dụng ở bất cứ chỗ nào trong ứng dụng cần thực hiện hành vi xác thực định danh.

### Tôi có sử dụng cho một cơ sở dữ liệu không?

Bạn không cần kiểm tra JWT qua một cơ sở dữ liệu. Cách đơn giản bạn dựa trên thời điểm tạo và thời điểm hết hạn để điều khiển truy cập. Thường thi bạn sẽ mở cơ sở dữ liệu để tra cứu người dùng nào đó nhưng JWT tự thân nó không cần điều này.

Ví dụ, nếu bạn lên kế hoạch sử dụng JWT để thực hiện chứng thực qua socket dùng giao thức UDP thay vì bạn sử dụng một cơ sở dữ liệu. Nén tất cả thông tin bạn cần một cách trực tiếp vào token khi bạn khởi tạo nó. Bạn xác minh nó (kiểm tra xem nó hợp lệ).

Bạn có thể tuy nhiên sử dụng 1 cơ sở dữ liệu kiểm soát JWT. Nếu bạn thực hiện, bạn chứng thực token vẫn hợp lệ. Hoặc bạn có thể sử dụng các bản ghi trong DB để giải phóng tất cả token của user. Điều này khá dễ dàng trong Guardian bởi sử dụng [GuardianDb](https://github.com/hassox/guardian_db). GuardianDb sử dụng Guardians 'Hooks' để thực hiện kiểm tra xác thực, lưu và xóa khỏi DB. Chúng ta sẽ đề cập nó sau.

## Thiết lập

Có nhiều lựa chọn cho việc thiết lập Guardian. Chúng ta đề cập tới chúng ở vài điểm nhưng chỉ ở mức rất đơn giản.

### Thiết lập tối giản

Để bắt đầu đơn giản bạn chỉ cần vài thứ.

#### Cấu hình

`mix.exs`

```elixir
def application do
  [
    mod: {MyApp, []},
    applications: [:guardian, ...]
  ]
end

def deps do
  [
    {guardian: "~> x.x"},
    ...
  ]
end
```

`config/config.ex`

```elixir
config :guardian, Guardian,
  issuer: "MyAppId",
  secret_key: Mix.env, # trong mỗi tệp tin cấu hình từng môi trường bạn nên ghi đè nó nếu nó là ngoại vi
  serializer: MyApp.GuardianSerializer
```

Đây chỉ là thiết lập ở mức tối thiểu để bạn sử dụng Guardian. Bạn không nên nén khóa bí mật - secret key một cách thô bạo ở phạm vi rộng. Thay vào đó, mỗi môi trường lựa chọn khóa bí mật của riêng nó. Thường thì ta cất khóa bí mật ở môi trường Mix cho môi trường phát triển và kiểm thử - dev và test. Tuy nhiên riêng ở Staging and production khóa bí mật yêu cầu phải cất giữ tối mật. (ví dụ: được kèm theo `mix phoenix.gen.secret`)

`lib/my_app/guardian_serializer.ex`

```elixir
defmodule MyApp.GuardianSerializer do
  @behaviour Guardian.Serializer

  alias MyApp.Repo
  alias MyApp.User

  def for_token(user = %User{}), do: { :ok, "User:#{user.id}" }
  def for_token(_), do: { :error, "Unknown resource type" }

  def from_token("User:" <> id), do: { :ok, Repo.get(User, id) }
  def from_token(_), do: { :error, "Unknown resource type" }
end
```
Serializer của bạn đảm nhiệm phần tìm kiếm tài nguyên ở trong trường `sub` (subject). Nó có thể tìm trong DB, một API hoặc thậm chí trong nội dung một chuỗi dữ liệu.
Nó tìm kiếm từng tài nguyên đơn lẻ ở trường `sub`.

Đó là những cấu hình đơn giản nhất. Thực tế có thể có nhiều mã phức tạp hơn nhưng vừa đủ cho ta bắt đầu.

#### Sử dụng trong ứng dụng

Lúc này chúng ta đang đăng ký sử dụng Guardian trong file cấu hình, chúng ta cần gọi nó trong ứng dụng. Khi ta đang thiết lập mức đơn giản nhất, chúng ta bắt đầu với các yêu cầu chạy trong giao thức HTTP.

## Các yêu cầu trong giao thức HTTP

Guardian cung cấp một số Plugs để dễ dàng nhúng vào HTTP requests. Bạn có thể học ở đây về Plug [separate lesson](../specifics/plug/). Đặc điểm cần nhớ Guardian làm việc không nhất thiết cần Phoenix, nhưng chúng ta sử dụng Phoenix trong ví dụ dưới đây sẽ dễ dàng mô tả cách hoạt động.

Dễ nhất là sử dụng HTTP qua một thiết bị định hướng gói tin - router. Khi Guardian tích hợp HTTP hoàn toàn dựa trên plugs, bạn có thể sử dụng nó bất kỳ chỗ nào có sử dụng plug.

Pha tiến trình chung của Guardian plug là:

1. Tìm ra một token trong request và xác minh nó : `Verify*` plugs
2. Tìm ra tài nguyên tương ứng với mỗi token: `LoadResource` plug
3. Đảm bảo tính hợp lệ của token đó nếu không từ chối nó. `EnsureAuthenticated` plug

Đáp ứng tất cả các nhu cầu của các nhà phát triển ứng dụng, Guardian hiện thực các pha riêng rẽ. Để tìm token sử dụng `Verify*` plugs.

Để tạo một số pipelines.

```elixir
pipeline :maybe_browser_auth do
  plug Guardian.Plug.VerifySession
  plug Guardian.Plug.VerifyHeader, realm: "Bearer"
  plug Guardian.Plug.LoadResource
end

pipeline :ensure_authed_access do
  plug Guardian.Plug.EnsureAuthenticated, %{"typ" => "access", handler: MyApp.HttpErrorHandler}
end
```

Các pipelines có thể được sử dụng để tạo các yêu cầu xác thực khác nhau. Pipeline thứ nhất cố gắng tìm kiếm ra token đầu tiên trong phiên làm việc và sau đó tới header. Nếu nó tìm thấy token, nó sẽ đọc/ghi các thông tin cho bạn.

Pipeline thứ 2 cần token hợp lệ, xác nhận hợp lệ token hiện tại và đánh dấu nó "access". Để sử dụng nó, ta thêm chúng vào scope.

```elixir
scope "/", MyApp do
  pipe_through [:browser, :maybe_browser_auth]

  get "/login", LoginController, :new
  post "/login", LoginController, :create
  delete "/login", LoginController, :delete
end

scope "/", MyApp do
  pipe_through [:browser, :maybe_browser_auth, :ensure_authed_access]

  resource "/protected/things", ProtectedController
end
```

Các login routes ở trên sẽ chứng thực danh tính của user nếu cùng một đối tượng. Ở trong scope thứ 2 đó là một token hợp lệ tất cả các actions.
Bạn không đưa chúng vào trong một pipelines, bạn có thể đưa chúng vào trong controller của bạn sao cho việc điều chỉnh mã lệnh lắt léo hơn nhưng chúng ta đang nói cách thiết lập tối giản.

Chúng ta chưa nói phần mã sau này dùng. Đó là bắt các lỗi xảy ra khi ấy thêm `EnsureAuthenticated` plug. Đây là một module rất đơn giản trả về tới user

* `unauthenticated/2`
* `unauthorized/2`

Cả hai chức năng nhận một struct Plug.Conn và các parameter đầu vào sẽ có các lỗi tương ứng xảy ra. Bạn thậm chí có thể sử dụng một Phoenix controller!

#### Bên trong controller

Bên trong controller, đó là vài xử lý khi user đã đăng nhập. Bắt đầu với ví dụ đơn giản nhất.

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller
  use Guardian.Phoenix.Controller

  def some_action(conn, params, user, claims) do
    # làm gì đó
  end
end
```

Bằng việc sử dụng `Guardian.Phoenix.Controller` module, các action sẽ nhận 2 tham trị mà bạn tùy ý sử dụng pattern match cho chúng. Nên nhớ, nếu bạn không sử dụng `EnsureAuthenticated` bạn có thể nhận giá trị nil cho user và claims.

Mặt khác - chúng ta có cách viết lắt léo/rườm rà hơn sau - là để sử dụng plug helpers.

```elixir
defmodule MyApp.MyController do
  use MyApp.Web, :controller

  def some_action(conn, params) do
    if Guardian.Plug.authenticated?(conn) do
      user = Guardian.Plug.current_resource(conn)
    else
      # Không phải user
    end
  end
end
```

#### Đăng nhập/Thoát

Đăng nhập và thoát của phiên làm việc trên trình duyệt là rất đơn giản. Trong controller login ta viết như sau:

```elixir
def create(conn, params) do
  case find_the_user_and_verify_them_from_params(params) do
    {:ok, user} ->
      conn
      |> Guardian.Plug.sign_in(user, :access) # Ở đây ta dùng access. Các token khác có thể sử dụng, như :resfresh vân vân
      |> respond_somehow()
    {:error, reason} ->
      # xử lý xảy ra lỗi xác minh user
  end
end

def delete(conn, params) do
  conn
  |> Guardian.Plug.sign_out()
  |> respond_somehow()
end
```

Khi sử dụng API đăng nhập, nó có chút khác biệt bởi vì ở đó không có dựa trên session và bạn cần cung cấp một raw token - token gốc trở về lại cho người dùng. Hàm này tiện lợi khi bạn không có ý định cho việc sử dụng một session.

```elixir
def create(conn, params) do
  case find_the_user_and_verify_them_from_params(params) do
    {:ok, user} ->
      {:ok, jwt, _claims} = Guardian.encode_and_sign(user, :access)
      conn
      |> respond_somehow({token: jwt})
    {:error, reason} ->
      # xử lý xảy ra lỗi xác minh user
  end
end

def delete(conn, params) do
  jwt = Guardian.Plug.current_token(conn)
  Guardian.revoke!(jwt)
  respond_somehow(conn)
end
```

Bản chất việc session đăng nhập trình duyệt gọi `encode_and_sign` vẫn thế vì vậy bạn có thể sử dụng chúng khá tương tự.
