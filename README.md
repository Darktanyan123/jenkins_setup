# Resume Analyser — chạy trên Jenkins

Pipeline ở `Jenkinsfile` kiểm thử frontend/backend, build và push ba Docker images lên Docker Hub, sau đó deploy bằng Docker Compose. Pipeline hỗ trợ Jenkins agent Windows hoặc Linux. Jenkins phải có agent online với executor khả dụng; nếu có nhiều agent, chỉ chọn agent được cài đủ các yêu cầu dưới đây.

## 1. Chuẩn bị Jenkins agent

Agent cần:

- JDK 21 và Node.js 20.
- Docker Engine/CLI và Docker Compose v2 (`docker compose`).
- Quyền chạy Docker từ tài khoản dịch vụ Jenkins.
- Kết nối mạng tới GitHub, Docker Hub và (nếu bật) SonarQube.

Jenkins controller có thể chạy trên hệ điều hành khác agent. Pipeline tự chọn `sh` cho Linux/macOS và `bat` cho Windows.

## 2. Cấu hình Jenkins

1. Cài các plugin **Pipeline**, **Git**, **NodeJS**, và **Docker Pipeline**. Chỉ khi bật SonarQube, cài thêm **SonarQube Scanner for Jenkins**.
2. Trong **Manage Jenkins → Tools**, cấu hình:
   - JDK installation tên `jdk21`, trỏ tới JDK 21.
   - NodeJS installation tên `NodeJS-20`, trỏ tới Node.js 20.
   - Nếu cần, bật tự cài đặt các tool trên.
3. Trong **Manage Jenkins → Credentials**, thêm Docker Hub credential dạng **Username with password**, ID chính xác `dockerhub-credentials`. Tài khoản này cần quyền push vào repository của Docker Hub.
4. Tạo job kiểu **Pipeline** → **Pipeline script from SCM**:
   - SCM: Git.
   - Repository URL: `https://github.com/Darktanyan123/jenkins_setup.git`
   - Branch: `*/jenkins_new` (hoặc branch có Jenkinsfile).
   - Script Path: `Jenkinsfile`.
5. Đảm bảo job được chạy trên agent đã chuẩn bị ở bước 1. Nếu Jenkins có nhiều agent, gán label cho agent phù hợp và thêm label đó vào `agent any` trong Jenkinsfile.

## 3. Chạy pipeline

Chọn **Build with Parameters**:

- `BRANCH_NAME`: nhánh cần build, mặc định `jenkins_new`.
- `DOCKERHUB_USERNAME`: tài khoản sở hữu các image `resume-frontend`, `resume-backend`, `resume-mongodb`; mặc định `hoang2207`. Giá trị này phải khớp với credential Docker Hub.
- `ENABLE_SONAR`: để `false` nếu chưa thiết lập SonarQube. Khi bật, Jenkins cần có tool `sonar-scanner`, SonarQube server tên `SonarQube`, và webhook từ SonarQube về Jenkins cho Quality Gate.

Pipeline chỉ deploy sau khi test, build và push thành công. Docker Compose sử dụng network do Compose tự tạo và giữ dữ liệu MongoDB trong volume `mongo-data`.

## 4. Kiểm tra kết quả

Sau khi stage **Deploy Containers** thành công, trên Docker host của agent:

- Mở frontend tại `http://localhost:3000`.
- Backend được publish tại `http://localhost:8081`.
- Kiểm tra trạng thái bằng `docker compose ps` trong workspace của job; xem log bằng `docker compose logs`.

Các cổng `3000`, `8081`, `27017` phải còn trống trên Docker host. Nếu agent không phải máy cần phục vụ ứng dụng, cần cấu hình deploy từ xa thay vì dùng `agent any`.

## Khắc phục nhanh

- **`No such DSL method: nodejs`**: cài plugin NodeJS và cấu hình tool `NodeJS-20`.
- **Không tìm thấy JDK/tool**: kiểm tra tên trong **Manage Jenkins → Tools** khớp chính xác `jdk21` và `NodeJS-20`.
- **Docker permission/daemon error**: đảm bảo Docker đang chạy và tài khoản dịch vụ Jenkins có quyền truy cập.
- **Docker Hub push bị từ chối**: kiểm tra credential ID, quyền push và `DOCKERHUB_USERNAME`.
- **`docker compose` không tồn tại**: cài Docker Compose v2 plugin trên agent.
- **Frontend gọi API lỗi**: frontend Nginx chuyển `/api/` tới backend qua cổng container `8080`; kiểm tra backend và MongoDB bằng `docker compose logs`.
