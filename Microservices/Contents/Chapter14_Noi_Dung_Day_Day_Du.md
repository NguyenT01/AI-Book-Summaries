# Bài học: Từ "chạy được trên máy tôi" đến "chạy y hệt ở mọi nơi" — Container hóa hệ thống microservices

> Chương 14 — Container Hosting and Development Patterns
> Ví dụ xuyên suốt: hệ thống quản lý phòng khám (HealthCare) với các service Patients API, Appointments API, Auth, API Gateway — giờ đây được đóng gói thành container và điều phối bằng Docker Compose lẫn Kubernetes.

---

## Phần 1: Vấn đề thực tế — khi "nó chạy trên máy tôi" không còn là lời bào chữa được chấp nhận

Hãy hình dung bạn vừa hoàn thành việc bảo mật toàn bộ hệ thống HealthCare ở chương trước — Gateway đã có xác thực OAuth 2.0, các route đã được gắn scope, BFF đã tách riêng cho web và mobile. Bạn tự tin deploy lên server QA. Và rồi mọi thứ vỡ trận: máy QA thiếu đúng phiên bản .NET runtime, biến môi trường đặt sai, một service thì chạy, service khác thì báo lỗi thư viện không tìm thấy. Bạn ngồi cả buổi chiều để "dựng lại y hệt máy dev" trên một máy hoàn toàn xa lạ.

Đây không phải là chuyện hiếm — đây là bản chất của việc vận hành nhiều service, mỗi service có thể viết bằng ngôn ngữ khác nhau, phụ thuộc runtime khác nhau, cần cấu hình hạ tầng khác nhau. Càng nhiều service, việc quản lý thủ công qua từng máy chủ càng dễ sinh ra "configuration drift" — tức là môi trường dần lệch nhau theo thời gian dù ban đầu được set up giống hệt.

Cách giải quyết truyền thống nhất là: mỗi ứng dụng một máy chủ riêng. Nhưng máy chủ vật lý không rẻ, còn phải tính chi phí bản quyền, điện năng, và nếu một máy hỏng, bạn phải cấu hình lại từ đầu đúng như bản gốc — một công việc tốn thời gian và dễ sai sót.

Bước tiến kế tiếp là ảo hóa (virtualization) — dùng máy ảo (VM) để tái sử dụng hạ tầng vật lý sẵn có, giảm chi phí máy móc, và có thể snapshot để khôi phục nhanh khi sự cố xảy ra. VMware, VirtualBox, Hyper-V là những cái tên quen thuộc ở giai đoạn này. Nhưng ảo hóa vẫn để lại vài vấn đề dai dẳng: máy chủ vật lý vẫn cần đủ mạnh để gánh nhiều VM cùng lúc, việc bảo trì hệ điều hành cho từng VM vẫn là công việc thủ công, và quan trọng nhất — dù cố gắng dựng lại "y hệt" môi trường trên các lần cài đặt khác nhau, bạn vẫn thường gặp sai khác không lường trước được.

### Analogy: Từ nhà trọ riêng biệt đến chung cư mini có vách ngăn

Hãy tưởng tượng mỗi service của bạn là một người thuê nhà. Mô hình máy chủ vật lý riêng biệt giống như xây hẳn một căn nhà riêng cho mỗi người thuê — tốn đất, tốn vật liệu, và nếu một căn nhà xuống cấp, bạn phải sửa lại từ móng. Mô hình máy ảo giống như xây một tòa nhà lớn (server) rồi ngăn thành các căn hộ riêng biệt hoàn chỉnh (VM) — mỗi căn hộ vẫn có bếp, phòng tắm, hệ thống điện nước riêng của nó (tức là cả một hệ điều hành đầy đủ bên trong), nên vẫn khá nặng nề và tốn tài nguyên trùng lặp.

Container thì giống như một chung cư mini: các phòng chia sẻ chung hệ thống điện nước cốt lõi của tòa nhà (kernel của hệ điều hành host), nhưng mỗi phòng vẫn có vách ngăn riêng, khóa cửa riêng, đồ đạc riêng — đủ cách ly để không ai đụng vào không gian của ai, nhưng nhẹ hơn hẳn vì không phải nhân bản toàn bộ hạ tầng điện nước cho từng phòng.

Container ra đời trên nền tảng tư duy của máy ảo, nhưng cắt giảm mạnh phần tài nguyên trùng lặp đó. Việc đóng gói một ứng dụng cùng toàn bộ dependency của nó thành một gói có thể lặp lại, kiểm thử được, và đáng tin cậy — gọi là **containerization**, và gói kết quả gọi là **image**. Image này triển khai ở đâu cũng cho ra hành vi giống hệt nhau, vì nó mang theo môi trường được tinh chỉnh riêng cho chính ứng dụng đó, chứ không phụ thuộc vào việc hệ điều hành nền có được cài đặt đồng nhất hay không.

**Bài học cốt lõi: Container không phải là "máy ảo nhẹ hơn" — nó là một cách tư duy lại hoàn toàn về việc đóng gói ứng dụng: đóng gói cùng với môi trường chạy của nó, thay vì hy vọng môi trường đích giống hệt môi trường nguồn.**

---

## Phần 2: Docker — công cụ biến containerization thành phổ cập

Docker không phải công cụ duy nhất trong mảng này. Podman, containerd, CRI-O đều tuân theo chuẩn **OCI (Open Container Initiative)** — một bộ tiêu chuẩn định nghĩa format image và runtime thống nhất, đảm bảo các công cụ này tương thích qua lại với nhau. Nhưng Docker vẫn là cái tên có ảnh hưởng lớn nhất và thân thiện nhất với developer, và chính Docker là công cụ đã đưa containerization từ một khái niệm học thuật trở thành thực hành phổ biến trong ngành.

Docker miễn phí cho mục đích phát triển và mã nguồn mở, chạy đa nền tảng, và bạn có thể thao tác qua giao diện đồ họa (Docker Desktop) hoặc dòng lệnh (CLI).

Về mặt kiến trúc, Docker theo mô hình **client-server**: Docker client (thường chính là CLI bạn gõ lệnh) giao tiếp với **Docker daemon** (`dockerd`) thông qua một REST API. Daemon này chạy nền trên máy host, chịu trách nhiệm quản lý vòng đời container, build và lưu trữ image, xử lý mạng, cấp phát volume, và thực thi container.

Ngoài container, Docker còn quản lý một số đối tượng hỗ trợ quan trọng:

- **Networks**: định nghĩa cách các container giao tiếp với nhau và với bên ngoài.
- **Volumes**: cho phép container lưu trữ và chia sẻ dữ liệu bền vững qua các lần khởi động lại.
- **Plugins**: mở rộng chức năng lõi của Docker, ví dụ hỗ trợ logging tùy chỉnh hay quản lý secret.

Còn về khái niệm container — nói ngắn gọn, một **container** là một môi trường cô lập, nhẹ, chạy một ứng dụng cùng toàn bộ dependency của nó. Nó được khởi tạo từ một **image** — bản thiết kế tĩnh, mô tả filesystem, cấu hình, và các dependency cần thiết để chạy ứng dụng bên trong. Image chính là những "bản chụp" môi trường có thể tái sử dụng nhất quán giữa các team và các môi trường triển khai khác nhau.

Để phân phối image, Docker cung cấp **Docker Hub** — một registry công khai khổng lồ, lưu hàng ngàn image dựng sẵn cho các stack phổ biến như Redis, NGINX, PostgreSQL, .NET. Ở cấp độ doanh nghiệp, các registry riêng như **Azure Container Registry (ACR)** hay **Amazon Elastic Container Registry (ECR)** còn cung cấp thêm quyền truy cập chi tiết, geo-replication, và tích hợp sâu với các dịch vụ cloud khác.

**Nhắc lại từ Chapter 11**: đây chính là lúc Docker mà bạn đã dùng để container hóa Ocelot Gateway thành nhiều instance BFF (mobile gateway, web gateway) trở nên rõ nghĩa hơn hẳn — lúc đó bạn mới dùng ở mức thao tác, còn giờ bạn hiểu vì sao nó hoạt động: image bất biến, daemon quản lý vòng đời, và registry là nơi lưu trữ trung tâm để mọi thành viên team pull về dùng nhất quán.

---

## Phần 3: Container image — bản thiết kế bất biến

Một container image là một gói tự chứa (self-contained), mang theo mã thực thi, runtime, thư viện, biến môi trường, và file cấu hình cần thiết để chạy ứng dụng. Hãy coi nó như bản thiết kế cho việc tạo container — khi bạn "chạy" một container, thực chất bạn đang khởi tạo một instance của image đó trong bộ nhớ, với toàn bộ hành vi và dependency đã được định nghĩa sẵn.

Đặc điểm quan trọng nhất của image là **tính bất biến (immutability)**. Một khi image đã được tạo, nó không thể thay đổi. Muốn cập nhật — ví dụ thêm một dependency mới — bạn buộc phải build lại image từ đầu. Chính tính bất biến này đảm bảo môi trường dev, staging, và production dùng cùng một image y hệt nhau, xóa sổ hoàn toàn kiểu bào chữa kinh điển "nó chạy trên máy tôi mà".

Image được cấu tạo theo dạng **layer** (lớp chồng lên nhau), và mọi image đều bắt đầu từ một điểm gốc — gọi là **base image** hoặc **parent image**.

- **Base image** là điểm khởi đầu để tạo image mới. Nó có thể trống hoàn toàn (image `scratch`) hoặc chỉ chứa những thành phần tối thiểu cần thiết. Image `scratch` thường dùng khi build image siêu nhẹ đi kèm các binary được compile tĩnh sẵn, ví dụ ứng dụng Go, hoặc microservice .NET dùng Ahead-Of-Time (AOT) compilation.
- **Parent image** là image đã chứa sẵn một hệ điều hành và có thể thêm các thành phần runtime. Ví dụ, image chính thức `mcr.microsoft.com/dotnet/aspnet:10.0` gồm một bản Linux tối giản cùng ASP.NET Core runtime — đây chính là điểm khởi đầu lý tưởng khi bạn build container cho một ứng dụng web ASP.NET Core.

Thực ra hai thuật ngữ này cùng chỉ một khái niệm: image nền tảng mà image mới của bạn kế thừa từ đó.

### Ví dụ thực chiến: chạy Redis mà không cần build gì cả

Giả sử bạn cần một Redis cache chạy local để test caching layer (nhắc lại: caching đã xuất hiện ở Chapter 10 khi bàn resilience). Bạn không cần build Redis từ mã nguồn hay cài thủ công — chỉ cần pull image chính thức từ Docker Hub, dựa trên một bản Linux nhẹ như Alpine:

```bash
docker pull redis
```

Lệnh này tải image Redis mới nhất từ Docker Hub về máy bạn. Sau đó, chạy container:

```bash
docker run --name my-redis-container -d redis
```

Trong đó:
- `--name my-redis-container` đặt tên dễ đọc cho container.
- `-d` chạy container ở chế độ nền (detached mode).
- `redis` là tên image dùng để khởi tạo container.

Vậy là chỉ với hai dòng lệnh, bạn đã có một instance Redis chạy độc lập, cô lập, sẵn sàng để các service khác hoặc công cụ quản lý (như RedisInsight) kết nối vào. Muốn dừng lại, chỉ cần:

```bash
docker stop my-redis-container
```

Đây chính là sức mạnh cốt lõi của image: cấp phát một dịch vụ bên thứ ba hoàn chỉnh chỉ trong vài giây, không cần cấu hình thủ công hay quản lý dependency phức tạp.

---

## Phần 4: Được và mất khi container hóa — không có bữa trưa miễn phí

Trước khi lao vào việc viết Dockerfile riêng cho từng service, hãy dừng lại đánh giá công bằng, vì đây là quyết định kiến trúc, và mọi quyết định kiến trúc đều có đánh đổi.

### Những gì bạn nhận được

**Tính nhất quán môi trường** là lợi ích lớn nhất. Image đóng gói không chỉ mã nguồn mà cả dependency, cấu hình, và runtime chính xác cần thiết — nghĩa là phần mềm hoạt động giống hệt nhau trên laptop developer, trong pipeline CI/CD, và trên cụm Kubernetes production. Vì container có thể được tạo, hủy, thay thế trong vài giây, chúng khớp hoàn hảo với các nguyên tắc cloud-native như auto-scaling, rolling update, blue/green deployment. Các nền tảng orchestration như Kubernetes tận dụng chính đặc tính này để quản lý hệ thống phân tán quy mô lớn với độ chính xác và tốc độ cao.

Khác với máy ảo phải nhân bản toàn bộ hệ điều hành (chi phí overhead lớn), container **chia sẻ chung kernel của host**, nên nhẹ hơn hẳn cả về dung lượng đĩa lẫn RAM. Khởi động một container mất vài giây hoặc ít hơn — cực kỳ phù hợp cho workload co giãn linh hoạt (elastic workload). Điều này cũng giúp tăng mật độ triển khai trên mỗi máy chủ — chạy được nhiều service hơn trên ít node hơn, giảm chi phí hạ tầng. Một khi image đã build xong, đẩy lên registry, nó có thể chạy trên bất kỳ runtime tương thích nào, ở bất kỳ môi trường nào — tách rời ứng dụng khỏi phần cứng hay nhà cung cấp cloud cụ thể, mở đường cho chiến lược multi-cloud và hybrid-cloud thực sự.

Hệ sinh thái quanh Docker cũng đã trưởng thành: Docker Compose cho việc test multi-container ở local, Docker Desktop cho quản lý qua GUI, và các registry như Docker Hub, ACR, GitHub Container Registry cho việc phân phối.

### Cái giá phải trả

Tất cả container trên cùng một host **chia sẻ chung kernel hệ điều hành**. Điều này có nghĩa: nếu một container bị chiếm quyền, kẻ tấn công về lý thuyết có thể khai thác lỗ hổng kernel để ảnh hưởng tới các container khác. Dù cách ly của container vẫn khá mạnh, nó không chặt chẽ bằng cách ly hoàn toàn của máy ảo. Đây là mối lo thực sự trong môi trường multi-tenant, nơi container của các đội/khách hàng khác nhau cùng chạy trên một host.

Container cũng thay đổi cả mô hình bảo mật bạn phải quan tâm:
- Bảo mật quy trình build image.
- Ngăn chặn việc dùng base image lỗi thời hoặc không an toàn.
- Áp dụng kiểm soát runtime (qua AppArmor, seccomp, hay PodSecurityPolicies).
- Đảm bảo network policy ngăn chặn di chuyển ngang (lateral movement) giữa các service.

Bất kỳ sơ suất nào ở đây đều có thể dẫn tới lỗ hổng nghiêm trọng. Việc dùng công cụ scan lỗ hổng như Trivy hay Anchore, cùng với ký và xác minh image, trở thành yêu cầu bắt buộc chứ không phải tùy chọn.

Việc **giám sát (monitoring)** môi trường container hóa cũng phức tạp hơn hệ thống VM truyền thống, vì container mang tính chất ephemeral (tồn tại ngắn hạn) — log và metric phải được tổng hợp tập trung ngay từ đầu, vì container không tồn tại đủ lâu để các agent thu thập log truyền thống hoạt động hiệu quả. Thiếu công cụ quan sát (observability) phù hợp, bạn sẽ rất khó truy vết sự cố trong production.

Cuối cùng, dù container mang lại một lớp trừu tượng gọn gàng, việc xử lý **lưu trữ bền vững (persistent storage)**, service discovery, và phân đoạn mạng vẫn đặt ra thách thức mới — container có trạng thái (stateful) cần được quản lý cẩn thận qua volume mount hoặc hệ thống lưu trữ ngoài, và traffic mạng thường cần overlay network hoặc service mesh để định tuyến đúng.

**Bài học cốt lõi: Container giải quyết bài toán "chạy nhất quán ở mọi nơi", nhưng đổi lại bạn phải chủ động thiết kế lại toàn bộ tư duy bảo mật, giám sát, và lưu trữ — container không tự động làm những việc đó thay bạn.**

---

## Phần 5: Viết Dockerfile cho service ASP.NET Core

Một **Dockerfile** là một file text mô tả từng bước để build ra image, dùng ngôn ngữ chỉ thị (directive) khá giống YAML. Một Dockerfile điển hình gồm các thành phần:

- Base/parent image làm nền cho image mới.
- Các lệnh cập nhật hệ điều hành, cài thêm phần mềm hoặc plugin cần thiết (nên chọn base image phù hợp ngay từ đầu để giảm thiểu việc phải thêm nhiều thứ về sau, vừa gọn vừa an toàn hơn).
- Các asset đã build sẵn của ứng dụng, được đưa vào image.
- Các asset khác liên quan tới lưu trữ và mạng.
- Lệnh chạy ứng dụng khi container khởi động.

Vì hệ thống HealthCare của chúng ta gồm nhiều web service, mỗi service cần một Dockerfile riêng. Vì tất cả đều dựa trên ASP.NET Core, ta có thể dùng chung một khuôn mẫu. Trong Visual Studio, chỉ cần right-click vào project trong Solution Explorer → Add → Docker Support…; trong VS Code, dùng Command Palette chọn "Docker: Add Docker Files to Workspace" — công cụ sẽ hỏi loại runtime và có cần hỗ trợ `docker-compose` hay không, rồi tự sinh file.

Một lưu ý quan trọng khi làm việc này: để ý tới **Container OS** (Windows hay Linux) và **Container Image Distro**. Vì mục tiêu cốt lõi của container là càng nhẹ càng tốt, việc chọn distro (như Alpine so với Debian) ảnh hưởng trực tiếp tới kích thước image cuối cùng — hãy chọn cái vừa đủ nhẹ vừa đáp ứng đúng nhu cầu ứng dụng.

Với service Appointments API, Dockerfile sinh ra trông như sau:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-alpine AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# This stage is used to build the service project
FROM mcr.microsoft.com/dotnet/sdk:9.0-alpine AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["AppointmentsApi.csproj", "."]
RUN dotnet restore "./AppointmentsApi.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "./AppointmentsApi.csproj" -c $BUILD_CONFIGURATION -o /app/build

# This stage is used to publish the service project to be copied to the final stage
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./AppointmentsApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# This stage is used in production or when running from VS in regular mode
# (Default when not using the Debug configuration)
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "AppointmentsApi.dll"]
```

Đây là kiểu **multi-stage build** — một kỹ thuật cực kỳ quan trọng cần hiểu rõ tại sao lại viết như vậy chứ không chỉ chép lại:

1. **Stage `base`**: dùng image `aspnet:10.0-alpine` — chỉ chứa runtime ASP.NET Core, không có SDK build. Đây là nền cho image cuối cùng, giữ cho nó nhẹ nhất có thể. Khai báo port 8080/8081 để expose ra ngoài.
2. **Stage `build`**: dùng image `sdk:9.0-alpine` — nặng hơn hẳn vì có đầy đủ công cụ build/compiler. Restore package, copy source code, và build. Stage này chỉ tồn tại tạm thời trong quá trình build, không có mặt trong image cuối.
3. **Stage `publish`**: kế thừa từ `build`, chạy `dotnet publish` để tạo ra output tối ưu cho production, `/p:UseAppHost=false` vì container tự cung cấp entrypoint chạy dotnet, không cần executable native đi kèm.
4. **Stage `final`**: quay lại dùng `base` (nhẹ, chỉ có runtime) làm nền, rồi **chỉ copy đúng phần đã publish** từ stage `publish` sang — không mang theo SDK, không mang theo source code gốc, không mang theo các file trung gian của quá trình build.

Đây chính là lý do tại sao multi-stage build lại quan trọng: nếu bạn build và chạy trong cùng một stage duy nhất, image cuối cùng sẽ phải mang theo toàn bộ SDK (nặng gấp nhiều lần runtime) dù production không hề cần đến compiler. Multi-stage build tách hẳn "công cụ dùng để build" khỏi "thứ thực sự cần chạy", giữ cho image production gọn nhẹ tối đa.

Kèm theo Dockerfile là file **`.dockerignore`**, liệt kê những gì không cần đưa vào image (thư mục `bin`, `obj`, file `.git`, `node_modules`, `README.md`...) — về cơ bản không cần chỉnh sửa file này trừ khi có nhu cầu đặc biệt.

Một điểm đáng chú ý: bản thân base image `mcr.microsoft.com/dotnet/aspnet:10.0` được xây trên những image nào phía sau nó, bạn hoàn toàn không cần biết — bạn chỉ cần dùng nó làm điểm khởi đầu để tạo ra image phái sinh của riêng mình. Cách tiếp cận theo layer này giúp bạn tận dụng công sức của những image đã có sẵn mà không phải tham chiếu trực tiếp hay làm phình to Dockerfile của mình.

---

## Phần 6: Chạy và debug container ngay trong Visual Studio

Sau khi có Dockerfile cho một service, việc lặp lại cho các service còn lại là điều tất yếu — làm xong toàn bộ, coi như bạn đã container hóa hoàn chỉnh hệ thống HealthCare.

Điều thú vị là Visual Studio/VS Code giờ hỗ trợ **launch trực tiếp trong container** mà vẫn giữ nguyên trải nghiệm debug real-time như chạy bình thường. Trong `Properties/launchSettings.json` của project, bạn sẽ thấy một profile mới được thêm vào:

```json
"profiles": {
  // Existing profiles
  "Container (Dockerfile)": {
    "commandName": "Docker",
    "launchUrl": "{Scheme}://{ServiceHost}:{ServicePort}",
    "environmentVariables": {
      "ASPNETCORE_HTTPS_PORTS": "8081",
      "ASPNETCORE_HTTP_PORTS": "8080"
    },
    "publishAllPorts": true,
    "useSSL": true
  }
}
```

Chọn profile này để launch sẽ thực thi toàn bộ chỉ thị trong Dockerfile — build image từ base Microsoft, restore, publish, copy file vào container mới, rồi chạy ứng dụng. Điểm khác biệt lớn nhất trong trải nghiệm là IDE giờ hiển thị thêm thông tin về container: health, version, port, biến môi trường, log, thậm chí cả filesystem bên trong container — giúp bạn theo dõi runtime sát sao hơn nhiều so với chạy trực tiếp trên máy dev.

Muốn xem danh sách container đang chạy qua CLI:

```bash
docker ps -a
```

Một vài lệnh Docker hữu ích khác cần nắm:

| Lệnh | Chức năng |
|---|---|
| `docker run` | Tạo/khởi động container mới, có thể đặt tên qua tham số `--name` |
| `docker pause` | Tạm dừng container, đình chỉ mọi hoạt động |
| `docker unpause` | Ngược lại với `pause` |
| `docker restart` | Gộp cả stop và start, khởi động lại container |
| `docker stop` | Gửi tín hiệu dừng tới container và các tiến trình bên trong |
| `docker rm` | Xóa hẳn container cùng dữ liệu liên quan |

Điểm mấu chốt ở đây: một khi đã container hóa, bạn không còn phải lo cấu hình đặc biệt trên từng server hay rủi ro server này chạy khác server kia — cùng một môi trường triển khai được lặp lại y hệt trên bất kỳ máy nào, và bạn luôn nhận được kết quả nhất quán.

### Container hóa không cần Dockerfile — native .NET container support

Kể từ .NET 7, bạn có thể container hóa ứng dụng mà **không cần viết Dockerfile** — thông qua package `Microsoft.NET.Build.Containers`:

```bash
dotnet add package Microsoft.NET.Build.Containers
```

Sau đó publish trực tiếp thành container và chạy bằng Docker:

```bash
dotnet publish --os linux --arch x64 -c Release -p:PublishProfile=DefaultContainer
docker run -it --rm -p 5010:80 healthcare-appointments-api:1.0.0
```

Container tự host sẽ lắng nghe traffic tại port 5010. Đây là một lựa chọn gọn nhẹ khi bạn muốn container hóa nhanh mà không cần quản lý Dockerfile thủ công, dù về bản chất cơ chế build phía sau vẫn tương tự.

---

## Phần 7: Container registry — nơi lưu trữ và chia sẻ image

Khi kiến trúc microservices phình to, khả năng chia sẻ, lưu trữ, và phân phối image một cách nhất quán trở nên thiết yếu. **Container registry** đóng vai trò trung tâm lưu trữ image có version, để cả team lẫn hệ thống tự động có thể pull đúng image nhất quán qua các môi trường dev, test, production.

Registry có hai loại:

- **Public registry** (như Docker Hub): chia sẻ image công khai với cộng đồng — bạn pull base image như `mcr.microsoft.com/dotnet/aspnet`, hoặc đóng góp image của riêng mình.
- **Private registry**: lưu trữ image nội bộ an toàn cấp doanh nghiệp, qua các dịch vụ như Google Container Registry, Azure Container Registry, Amazon ECR — mang lại kiểm soát chi tiết hơn về:
  - Xác thực (OAuth, Active Directory...).
  - Kiểm soát truy cập theo vai trò (RBAC) trên từng image.
  - Quét lỗ hổng bảo mật và quản lý version.
  - Audit hoạt động và log truy cập.

Bạn cũng có thể tự host registry on-premises qua Harbor, Nexus, hay tự dựng Docker registry riêng, để kiểm soát toàn bộ policy và tích hợp.

### Ví dụ thực chiến: đóng gói SQL Server đã seed sẵn dữ liệu

Giả sử team cần một image database dùng chung có sẵn schema khởi tạo. Đầu tiên, pull base image (nên chỉ định version cụ thể thay vì luôn dùng bản mới nhất, để đảm bảo tính nhất quán qua các lần triển khai):

```bash
docker pull mcr.microsoft.com/mssql/server:2022-latest
```

Chạy container, expose qua port 1434 (tránh đụng port 1433 mặc định nếu máy đã có SQL Server khác):

```bash
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=AStrongP@ssword1" -p 1434:1433 -d mcr.microsoft.com/mssql/server
```

Kết nối bằng SQL Server Management Studio hoặc Azure Data Studio, tạo database và bảng:

```sql
CREATE DATABASE PatientsDb
GO
USE PatientsDb
CREATE TABLE Patients(
    Id int primary key identity,
    FirstName varchar(50),
    LastName varchar(50),
    DateofBirth datetime
)
```

Có hai cách để đóng gói trạng thái này thành image mới:

**Cách 1 — commit trực tiếp container đang chạy:**

```bash
docker commit -m "Added Patients Database" -a "Your Name" adoring_boyd Username/patients-db:latest
docker push Username/patients-db
```

Cách này nhanh, giữ đúng trạng thái hiện tại của container, nhưng khó lặp lại và khó theo dõi thay đổi qua thời gian — vì bản chất nó "chụp ảnh" một trạng thái runtime, không phải một quy trình có thể review lại từng bước.

**Cách 2 — build từ Dockerfile + script khởi tạo (khuyến nghị):**

Tạo thư mục `sqlserver-db`, thêm file `patients_init.sql` chứa script tạo database, và một Dockerfile:

```dockerfile
FROM mcr.microsoft.com/mssql/server:2022-latest
ENV SA_PASSWORD="YourStrongP@ssword1" \
    ACCEPT_EULA=Y \
    MSSQL_PID=Developer
COPY patients_init.sql /patients_init.sql
CMD /bin/bash -c "
    /opt/mssql/bin/sqlservr &
    sleep 20 &&
    /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P YourStrongP@ssword1 -i /patients_init.sql &&
    wait
"
```

Build, tag, và push:

```bash
docker build -t healthcare/patients-db:v1 .
docker tag healthcare/patients-db:v1 yourdockerhub/patients-db:v1
docker push yourdockerhub/patients-db:v1
```

Cách này ưu việt hơn hẳn vì mọi bước đều được ghi lại tường minh dưới dạng code (script + Dockerfile), có thể review, version control, và tái tạo chính xác bất cứ lúc nào — thay vì phụ thuộc vào trạng thái "tình cờ" của một container đang chạy.

Sau khi có image này, việc thêm nó vào orchestration chỉ đơn giản là:

```yaml
patients_sql_db:
  image: Username/patients-db
  restart: always
  ports:
    1434:1433
```

**Bài học cốt lõi: `docker commit` nhanh nhưng mù mờ (bạn không biết chính xác trạng thái container ra sao khi commit); Dockerfile + script chậm hơn một chút nhưng minh bạch, review được, và tái lập chính xác — trong môi trường team, luôn ưu tiên cách thứ hai.**

---

## Phần 8: Docker Compose — điều phối nhiều container cho môi trường dev

Hệ thống HealthCare của chúng ta không chỉ có một service — nó còn có cache Redis, dịch vụ xác thực, API Gateway, và nhiều service nghiệp vụ khác. Chạy thủ công từng container, đúng thứ tự, đúng dependency, là việc gần như bất khả thi khi làm tay. Đây chính là lúc **container orchestration** phát huy tác dụng — cách tiếp cận tự động để khởi chạy container và các service liên quan theo đúng thứ tự khai báo.

Orchestration mang lại ba lợi ích rõ rệt:

- **Đơn giản hóa vận hành**: tự động hóa việc khởi chạy nhiều container theo đúng thứ tự và cấu hình định sẵn.
- **Khả năng phục hồi (resiliency)**: theo dõi health, tải hệ thống, hoặc nhu cầu scale để tự động điều chỉnh instance, đảm bảo tính ổn định.
- **Bảo mật**: giảm sai sót do thao tác thủ công của con người.

**Docker Compose** là hình thức orchestration đơn giản nhất — định nghĩa toàn bộ stack ứng dụng multi-container trong một file YAML duy nhất, rồi khởi chạy tất cả chỉ với một lệnh. Lợi thế lớn là mọi thứ nằm gọn trong một file, đặt ngay tại thư mục gốc, dễ version control và chia sẻ với team.

Trong Visual Studio, right-click một service project → Add → Container Orchestration Support… → chọn Docker Compose → chọn hệ điều hành target (Windows hoặc Linux, cả hai đều chạy tốt vì ASP.NET Core đa nền tảng). Lặp lại cho từng project cần đưa vào orchestration.

File `docker-compose.yml` sinh ra ban đầu chỉ có một service:

```yaml
version: '3.4'
services:
  healthcare.patients.api:
    image: ${DOCKER_REGISTRY-}healthcarepatientsapi
    build:
      context: .
      dockerfile: HealthCare.Patients.Api/Dockerfile
```

Sau khi thêm các service còn lại, file phát triển thành:

```yaml
version: '3.4'
services:
  healthcare.patients.api:
    image: ${DOCKER_REGISTRY-}healthcarepatientsapi
    build:
      context: .
      dockerfile: HealthCare.Patients.Api/Dockerfile
  healthcare.auth:
    image: ${DOCKER_REGISTRY-}healthcareauth
    build:
      context: .
      dockerfile: HealthCare.Auth/Dockerfile
  healthcare.appointments.api:
    image: ${DOCKER_REGISTRY-}healthcareappointmentsapi
    build:
      context: .
      dockerfile: HealthCare.Appointments.Api/Dockerfile
  healthcare.apigateway:
    image: ${DOCKER_REGISTRY-}healthcareapigateway
    build:
      context: .
      dockerfile: HealthCare.ApiGateway/Dockerfile
```

Muốn thêm Redis cache (nhắc lại: Redis đã dùng cho caching layer từ Chapter 10), chỉ cần thêm:

```yaml
services:
  redis:
    image: "redis:alpine"
  # … Other services …
```

### Khai báo thứ tự phụ thuộc bằng `depends_on`

Appointments service của chúng ta giao tiếp thường xuyên với Patients service — nên hợp lý là Patients service phải khởi động trước. Compose cho phép khai báo điều này tường minh:

```yaml
healthcare.appointments.api:
  image: ${DOCKER_REGISTRY-}healthcareappointmentsapi
  depends_on:
    - healthcare.patients.api
  build:
    context: .
    dockerfile: HealthCare.Appointments.Api/Dockerfile
```

### Ghi đè cấu hình theo môi trường với `docker-compose.override.yml`

File này dùng để lớp thêm (layer) cấu hình riêng cho từng môi trường — ví dụ môi trường development — lên trên file `docker-compose.yml` gốc, mà không cần lặp lại các chỉ thị Dockerfile:

```yaml
version: '3.4'
services:
  healthcare.patients.api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:443;http://+:80
    ports:
      - "80"
      - "443"
    volumes:
      - ${APPDATA}/Microsoft/UserSecrets:/root/.microsoft/usersecrets:ro
      - ${APPDATA}/ASP.NET/Https:/root/.aspnet/https:ro
    # ... Other overrides...
```

Nếu muốn kiểm soát tường minh hơn, bạn có thể bỏ hẳn file override này — miễn là các giá trị cần thiết đã được cấu hình sẵn trong Dockerfile.

### Callback quan trọng: BFF Pattern (Chapter 11) giờ triển khai gọn hơn nhờ container

Nhớ lại ở Chapter 11, để có nhiều BFF instance phục vụ riêng cho mobile và web, cách làm truyền thống là tạo nhiều project code riêng biệt. Giờ với container hóa, bạn có thể **tái sử dụng cùng một image gateway**, chỉ khác nhau về cấu hình khi khởi tạo container:

```yaml
mobileappapigw:
  image: ${DOCKER_REGISTRY-}healthcareapigateway
  build:
    context: .
    dockerfile: HealthCare.ApiGateway/Dockerfile
webappapigw:
  image: ${DOCKER_REGISTRY-}healthcareapigateway
  build:
    context: .
    dockerfile: HealthCare.ApiGateway/Dockerfile
```

Rồi dùng file override để trỏ tới đúng thư mục cấu hình cho từng instance:

```yaml
mobilegatewaygw:
  environment:
    - ASPNETCORE_ENVIRONMENT=Development
    - IdentityUrl=IDENTITY_URL
  volumes:
    - ./HealthCare.ApiGateway/mobile:/app/configuration
webhealthcaregw:
  environment:
    - ASPNETCORE_ENVIRONMENT=Development
    - IdentityUrl=IDENTITY_URL
  volumes:
    - ./HealthCare.ApiGateway/web:/app/configuration
```

Đây là một bước tiến hóa đẹp: **BFF Pattern (Ch11)** ban đầu giải quyết bài toán "mỗi client cần một gateway tối ưu riêng" bằng cách tách project code; giờ với container, bạn giải quyết đúng bài toán đó chỉ bằng cách tái sử dụng một image, khác biệt hoàn toàn ở lớp cấu hình runtime (volume + biến môi trường) — giảm hẳn effort duy trì nhiều codebase song song.

Khi mọi thứ đã sẵn sàng, chỉ cần một lệnh để khởi chạy toàn bộ hệ thống:

```bash
docker-compose up
```

**Bài học cốt lõi: Docker Compose biến việc khởi chạy hàng loạt service phụ thuộc lẫn nhau từ một chuỗi thao tác thủ công dễ sai sót thành một file khai báo (declarative) duy nhất, dễ đọc, dễ chia sẻ, và dễ tái lập.**

---

## Phần 9: Khi Docker Compose không đủ — bước sang Kubernetes

Docker Compose tuyệt vời cho môi trường dev và orchestration quy mô nhỏ, nhưng nó nhanh chóng chạm giới hạn khi hệ thống microservices lớn dần: quản lý vòng đời container, mạng, scaling, và khôi phục sự cố cần một giải pháp mạnh mẽ và linh hoạt hơn nhiều. Đây là lúc **Kubernetes** bước vào cuộc chơi.

Ban đầu do Google phát triển, nay do **Cloud Native Computing Foundation (CNCF)** duy trì, Kubernetes đã trở thành chuẩn công nghiệp cho container orchestration ở quy mô lớn. Nó cung cấp mô hình cấu hình khai báo (declarative) cùng một API mạnh mẽ để quản lý service với vòng đời, dependency, và policy vận hành phức tạp:

- Tự phục hồi (self-healing): tự khởi động lại, tái lập lịch, nhân bản khi cần.
- Service discovery và load balancing tích hợp sẵn.
- Quản lý secret và cấu hình.
- Auto-scaling (cả chiều dọc lẫn chiều ngang).
- Rollback và rolling update.
- Hỗ trợ triển khai multi-cloud và hybrid.

### Analogy: Từ "tự lái xe theo checklist" sang "hệ thống lái tự động có giám sát"

Docker Compose giống như bạn tự lái xe theo một checklist cố định — bạn biết chính xác việc gì cần làm, làm đúng thứ tự, nhưng nếu có sự cố giữa đường (một container chết), bạn phải tự nhận ra và tự xử lý. Kubernetes giống như một hệ thống lái tự động có giám sát liên tục: nó không chỉ chạy theo kịch bản định sẵn mà còn **liên tục theo dõi trạng thái thực tế** và tự điều chỉnh để trạng thái đó luôn khớp với trạng thái mong muốn bạn khai báo — pod chết thì tự khởi động lại, node hỏng thì tự lập lịch lại nơi khác, không cần ai can thiệp thủ công.

### Các khối xây dựng nền tảng của Kubernetes

| Thành phần | Vai trò |
|---|---|
| **Cluster** | Tập hợp các máy worker (node) được quản lý bởi control plane |
| **Pod** | Đơn vị triển khai nhỏ nhất trong Kubernetes, bọc một hoặc nhiều container, chia sẻ tài nguyên storage/network |
| **Service** | Lớp trừu tượng expose một tập Pod thành một network service ổn định |
| **Deployment** | Controller quản lý Pod, đảm bảo đúng số lượng replica và cho phép rolling update |
| **Ingress** | Tập luật cho phép traffic từ bên ngoài đi vào các service trong cluster |

### Thiết lập Kubernetes cục bộ

Nếu dùng Docker Desktop, việc bật Kubernetes rất đơn giản: vào Settings → Kubernetes → bật "Enable Kubernetes" → Apply & restart. Docker Desktop tự mang theo một cluster Kubernetes single-node chạy ngay bên trong Docker.

Với Linux hoặc macOS thích dùng CLI, **Minikube** là lựa chọn tốt để chạy Kubernetes local, kèm theo việc cài `kubectl` thủ công. Dùng `kubectl get nodes` để xác nhận node đã ở trạng thái `Ready`.

Muốn có giao diện dashboard trực quan:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
kubectl proxy
```

### Đưa image lên registry để Kubernetes truy cập được

Manifest deployment sẽ tham chiếu tới `appointments-api:1.0`, nhưng Kubernetes chỉ pull được image nếu nó nằm trong registry mà cluster có quyền truy cập. Nếu dùng Docker Desktop kèm Kubernetes, cluster có thể đọc trực tiếp từ Docker engine local — nhưng để triển khai thực tế và có tính di động, bạn nên đẩy image lên Docker Hub, ACR, hay một private registry:

```bash
docker build -t <dockerhub-username>/appointments-api:1.0 .
docker push <dockerhub-username>/appointments-api:1.0
```

### Viết manifest Deployment

Tạo thư mục `k8s` tại gốc solution, thêm file `appointments-api-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appointments-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: appointments-api
  template:
    metadata:
      labels:
        app: appointments-api
    spec:
      containers:
        - name: appointments-api
          image: appointments-api:1.0
          ports:
            - containerPort: 80
```

File này định nghĩa một resource **Deployment**, chịu trách nhiệm quản lý vòng đời của microservice AppointmentApi — số lượng instance cần chạy, image nào dùng, và cấu hình môi trường container ra sao trong cluster.

Trọng tâm của spec này là **Pod template**: yêu cầu Kubernetes tạo Pod gắn label `app: appointments-api`, chạy container tên `appointments-api` từ image `appointments-api:1.0`. Deployment controller liên tục giám sát trạng thái ứng dụng, đảm bảo số replica khai báo luôn được duy trì đúng — nếu Pod crash hoặc node gặp sự cố, Kubernetes tự động khởi động lại hoặc lập lịch lại Pod đó ở nơi khác.

Đây chính là nền tảng cho khả năng phục hồi và co giãn trong môi trường microservices, đồng thời là cơ sở để thực hiện update và rollback không downtime, giúp team DevOps quản lý version ứng dụng qua rolling deployment.

### Viết manifest Service

Thêm file `appointments-api-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appointments-api-service
spec:
  selector:
    app: appointments-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 32002
  type: NodePort
```

Vì Pod có bản chất ephemeral (có thể bị restart bất cứ lúc nào với IP khác), **Service** cung cấp một endpoint mạng ổn định để các thành phần khác hoặc client luôn kết nối được tới microservice này, bất kể Pod bên dưới thay đổi thế nào.

Service dùng `selector` để xác định Pod nào thuộc về nó (khớp label `app: appointments-api` với Deployment ở trên), mở port 80 trên service và định tuyến traffic vào tới `targetPort` 80 bên trong container — tạo một điểm truy cập nhất quán, tách biệt hoàn toàn giữa cấu hình container thực tế và giao diện bên ngoài.

Loại service `NodePort` nghĩa là service còn được expose ra một port tĩnh cao trên máy host (thường từ 30000–32767) — rất tiện cho việc test local qua Minikube hay Docker Desktop, truy cập trực tiếp qua `localhost:<nodePort>`. Service cũng là khối xây dựng nền tảng cho các lớp định tuyến phức tạp hơn về sau, như Ingress controller hay API Gateway (nhắc lại Chapter 11) — vốn cần Service làm đích định tuyến.

### Chạy và kiểm tra

```bash
kubectl apply -f appointments-api-deployment.yaml
kubectl apply -f appointments-api-service.yaml

kubectl get deployments
kubectl get pods
kubectl get services

kubectl get svc appointments-api-service
```

Sau đó truy cập `http://localhost:{nodePort}` để gọi tới service.

### Cái giá thực sự của Kubernetes

Kubernetes là nền tảng của phát triển ứng dụng cloud-native hiện đại — mạnh mẽ, có khả năng tự động hóa cao, tích hợp sâu với CI/CD, dịch vụ cloud, service mesh, và công cụ quan sát. Nhưng phải thực tế về chi phí vận hành nó. Đây là hệ thống phức tạp với rất nhiều bộ phận chuyển động, đòi hỏi đường cong học tập đáng kể và thường cần một đội DevOps/platform engineering chuyên trách. Việc debug resource cấu hình sai, quản lý YAML sprawl (số lượng file YAML phình to khó kiểm soát), và tinh chỉnh cấu hình production có thể áp đảo những team nhỏ hoặc dự án còn ở giai đoạn đầu.

**Kubernetes không phải là mặc định, mà là một đích đến** — quan trọng không chỉ vì năng lực kỹ thuật mà còn vì mức độ chuẩn hóa và trưởng thành vận hành nó mang lại cho tổ chức. Chỉ nên áp dụng khi bạn hiểu rõ và sẵn sàng chấp nhận cả lợi ích lẫn đánh đổi đi kèm.

**Bài học cốt lõi: Docker Compose trả lời câu hỏi "làm sao chạy nhiều container cùng lúc ở local"; Kubernetes trả lời câu hỏi hoàn toàn khác — "làm sao giữ cho hệ thống hàng trăm container luôn ở đúng trạng thái mong muốn, tự phục hồi, tự co giãn, trên một hạ tầng phân tán thực sự". Đừng nhảy sang Kubernetes chỉ vì nó "chuẩn công nghiệp" — nhảy sang khi độ phức tạp hệ thống thực sự cần nó.**

---

## Phần cuối: Best Practices khi container hóa microservices

1. **Luôn dùng multi-stage build cho Dockerfile production.** Tách rõ stage build (nặng, có SDK) khỏi stage final (nhẹ, chỉ runtime) — giữ image cuối cùng nhỏ gọn, giảm surface tấn công và tăng tốc độ pull/deploy.

2. **Chọn distro base image có chủ đích, đừng mặc định.** Alpine nhẹ hơn Debian đáng kể nhưng đôi khi thiếu một số thư viện native — cân nhắc đúng nhu cầu ứng dụng trước khi chọn, đừng chỉ vì "nhẹ là tốt".

3. **Luôn pin version cụ thể cho image, tránh dùng `latest` trong production.** `latest` có thể thay đổi bất cứ lúc nào, phá vỡ tính tái lập (reproducibility) — nguyên tắc cốt lõi mà container hóa hướng tới ngay từ đầu.

4. **Ưu tiên Dockerfile + script hơn `docker commit` khi cần image tái lập được.** Như đã thấy ở ví dụ SQL Server, cách build từ Dockerfile minh bạch, review được qua version control — trong khi `commit` là một "hộp đen" trạng thái runtime.

5. **Dùng `depends_on` trong Compose để khai báo tường minh thứ tự khởi động**, nhưng nhớ rằng đây chỉ đảm bảo thứ tự *khởi động container*, không đảm bảo service bên trong đã sẵn sàng nhận request (application ready) — cần kết hợp thêm health check hoặc cơ chế retry ở tầng ứng dụng (nhắc lại Polly/resilience từ Chapter 10) để xử lý khoảng trễ này.

6. **Đừng đưa secret thật vào Dockerfile hay image.** Dùng biến môi trường qua `docker-compose.override.yml`, Kubernetes Secrets, hoặc user secrets — image là artifact có thể bị leak hoặc chia sẻ nhầm, không nên chứa thông tin nhạy cảm cố định.

7. **Luôn scan image trước khi đẩy lên registry production**, dùng công cụ như Trivy hoặc Anchore — lỗ hổng trong base image lỗi thời là một trong những nguồn tấn công phổ biến nhất với hệ thống container hóa.

8. **Chỉ nhảy sang Kubernetes khi có lý do rõ ràng** (nhiều service, cần auto-scaling thực sự, cần self-healing ở mức hạ tầng, cần multi-cloud) — với dự án nhỏ hoặc team ít kinh nghiệm vận hành, Docker Compose vẫn là lựa chọn thực dụng hơn.

---

## Tổng kết chương

Chương này đánh dấu bước chuyển từ việc bảo mật microservices (Chapter 13) sang việc triển khai và vận hành chúng một cách thực tế bằng container. Chúng ta bắt đầu từ chính nỗi đau quen thuộc — sự thiếu nhất quán giữa các môi trường triển khai — để thấy vì sao containerization ra đời như một giải pháp tự nhiên, đóng gói ứng dụng cùng toàn bộ dependency thành các đơn vị portable, có thể tái lập, giúp loại bỏ configuration drift và giảm overhead so với máy ảo truyền thống.

Chúng ta đi sâu vào Docker — kiến trúc client-server qua daemon `dockerd`, cách image được xây theo layer từ base/parent image, và cách viết Dockerfile theo mô hình multi-stage build để giữ image production gọn nhẹ. Chúng ta cũng thực hành container hóa cả dịch vụ bên thứ ba (Redis, SQL Server) lẫn chính các microservice ASP.NET Core của hệ thống HealthCare, đồng thời tìm hiểu cách publish image lên registry công khai và riêng tư để phục vụ cộng tác nhóm và triển khai lặp lại.

Từ đó, chúng ta chuyển sang bài toán orchestration — bắt đầu với Docker Compose, công cụ nhẹ nhàng và thân thiện với developer để định nghĩa dependency giữa các service, quản lý biến môi trường, mount volume cấu hình, và điều phối thứ tự khởi động. Đặc biệt, chúng ta thấy được cách BFF Pattern từ Chapter 11 được tái hiện gọn gàng hơn hẳn nhờ container — tái sử dụng một image gateway duy nhất cho nhiều instance thay vì duy trì nhiều codebase song song.

Khi kiến trúc phức tạp dần vượt quá khả năng của Compose, chúng ta giới thiệu Kubernetes — nền tảng orchestration chuẩn công nghiệp cho workload quy mô production, với mô hình khai báo xoay quanh Cluster, Pod, Deployment, Service, và Ingress. Nhưng chúng ta cũng dừng lại để nhìn thẳng vào thực tế vận hành: Kubernetes mạnh mẽ nhưng không hề đơn giản, và việc áp dụng nó cần đi kèm sự chuẩn bị và chủ đích rõ ràng, không phải một lựa chọn mặc định.

**Bài học cốt lõi của chương: Container hóa giải quyết đúng bài toán "chạy nhất quán ở mọi nơi", nhưng bản thân nó chỉ là bước khởi đầu — orchestration mới là lớp biến một tập hợp container rời rạc thành một hệ thống phân tán thực sự có khả năng tự phục hồi và co giãn; và việc chọn công cụ orchestration nào (Compose hay Kubernetes) phải dựa trên độ trưởng thành và quy mô thực tế của hệ thống, không phải theo trào lưu.**

Ở chương tiếp theo, chúng ta sẽ mở rộng những khái niệm này sang các mô hình phát triển hướng cloud, giúp xây dựng các service hiệu quả và co giãn tốt hơn nữa.
