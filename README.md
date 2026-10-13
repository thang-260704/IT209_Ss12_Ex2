# Session 12 - Bài 2: Tự động phát hành Docker Image khi gắn Git Tag `v*.*.*`

- Repository: <https://github.com/thang-260704/IT209_Ss12_Ex2>
- Workflow: [`.github/workflows/release.yml`](.github/workflows/release.yml)
- Dockerfile: [`Dockerfile`](Dockerfile)

## 1. Mục tiêu

Xây dựng luồng **CD (Continuous Delivery)**: mỗi khi gắn một Git Tag dạng `v*.*.*` và push lên GitHub, workflow tự động build Docker image của ứng dụng Spring Boot và đẩy lên **GitHub Container Registry (GHCR)** với tag trùng số phiên bản.

```text
git tag v1.0.0 ─► git push origin v1.0.0
                         │
                         ▼
         GitHub Actions: release.yml (on.push.tags: v*.*.*)
                         │
   checkout ─► buildx ─► login GHCR ─► metadata (tag) ─► build & push
                         │
                         ▼
     ghcr.io/thang-260704/it209_ss12_ex2:v1.0.0 (+ 1.0.0, 1.0, latest)
```

So với các bài trước: `ci-matrix.yml` (CI) chạy khi push lên `main` để kiểm tra code; `release.yml` (CD) chỉ chạy khi có tag phát hành, tạo ra **artifact có version** là Docker image.

## 2. Workflow `release.yml`

```yaml
name: Release Docker Image

on:
  push:
    tags:
      - 'v*.*.*'

# GITHUB_TOKEN cần quyền ghi Packages để push image lên GHCR
permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  release-image:
    name: Build & Push image ${{ github.ref_name }}
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Đăng nhập GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Trích xuất tag và label từ Git
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=tag
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build và push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Giải thích

| Thành phần | Ý nghĩa |
|---|---|
| `on.push.tags: ['v*.*.*']` | Chỉ chạy khi push tag dạng `v1.0.0`, `v2.3.1`; push code thường lên `main` không kích hoạt |
| `permissions.packages: write` | Cho `GITHUB_TOKEN` quyền push image lên GHCR. Không cần tạo Personal Access Token hay lưu mật khẩu |
| `docker/setup-buildx-action` | Bật BuildKit, cần cho cache `type=gha` |
| `docker/login-action` | Đăng nhập `ghcr.io` bằng `github.actor` và `secrets.GITHUB_TOKEN` (token tự sinh cho mỗi lần chạy, hết hạn khi job kết thúc) |
| `docker/metadata-action` | Từ Git tag sinh ra danh sách Docker tag và các OCI label (nguồn repo, commit, thời gian) |
| `type=ref,event=tag` | Tag giống hệt Git tag: **`v1.0.0`** |
| `type=semver,pattern={{version}}` | Tag semver không có chữ `v`: `1.0.0` |
| `type=semver,pattern={{major}}.{{minor}}` | Tag `1.0`, luôn trỏ tới bản vá mới nhất của dòng 1.0 |
| (tự động) `latest` | metadata-action tự thêm `latest` khi tag là semver |
| `docker/build-push-action` | Build theo `Dockerfile` và push tất cả tag lên GHCR |
| `cache-from/cache-to: type=gha` | Lưu cache layer Docker trên GitHub Actions, các lần release sau build nhanh hơn |

Tên image lấy từ `github.repository`; metadata-action tự chuyển về **chữ thường** (`it209_ss12_ex2`) vì Docker không chấp nhận chữ hoa.

## 3. Dockerfile (multi-stage)

```dockerfile
# ---------- Stage 1: build bootJar bằng Gradle Wrapper ----------
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /build

# Copy cấu hình Gradle trước để tận dụng layer cache cho dependency
COPY gradlew settings.gradle build.gradle ./
COPY gradle ./gradle
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon > /dev/null

# Copy mã nguồn và đóng gói (test đã chạy ở workflow CI)
COPY src ./src
RUN ./gradlew bootJar -x test --no-daemon

# ---------- Stage 2: runtime chỉ có JRE + app.jar ----------
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder /build/build/libs/*.jar app.jar
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Áp dụng các kỹ thuật từ Session 10: multi-stage, base image JRE Alpine, layer caching, user không phải root, `ENTRYPOINT` dạng exec form. `.dockerignore` loại `.git`, `.gradle`, `build`, `.github`, `*.md`, `images`.

## 4. Phát hành

```bash
git tag v1.0.0
git push origin v1.0.0
```

## 5. Kết quả

- Workflow run: <https://github.com/thang-260704/IT209_Ss12_Ex2/actions/runs/38356937550>
- Package: <https://github.com/thang-260704/IT209_Ss12_Ex2/pkgs/container/it209_ss12_ex2>

### 5.1 Image `v1.0.0` trên GitHub Packages

Package **`it209_ss12_ex2`** (Public), liên kết với repo `IT209_Ss12_Ex2`. Phiên bản mới nhất mang đủ 4 tag: **`latest`, `v1.0.0`, `1.0`, `1.0.0`**; mục OS / Arch có 2 mục (image `linux/amd64` và bản attestation do BuildKit sinh kèm).

```bash
docker pull ghcr.io/thang-260704/it209_ss12_ex2:v1.0.0
```

![GHCR package](images/04-ghcr-package.png)

### 5.2 Workflow Release được kích hoạt bởi tag

Push tag `v1.0.0` (commit `3e8c0d1`) kích hoạt run **#1** của `Release Docker Image`: Status **Success**, tổng **2m 36s**, job **Build & Push image v1.0.0** mất 2m 33s.

![Release run](images/01-release-run.png)

![Git tag v1.0.0](images/03-git-tag.png)

### 5.3 Các tag do `docker/metadata-action` sinh ra

```text
ghcr.io/thang-260704/it209_ss12_ex2:1.0.0
ghcr.io/thang-260704/it209_ss12_ex2:1.0
ghcr.io/thang-260704/it209_ss12_ex2:v1.0.0
ghcr.io/thang-260704/it209_ss12_ex2:latest
```

![Docker tags](images/05-docker-tags.png)

| Step | Thời gian |
|---|---:|
| Thiết lập Docker Buildx | 7s |
| Đăng nhập GitHub Container Registry | 0s |
| Trích xuất tag và label từ Git | 1s |
| **Build và push Docker image** | **2m 15s** |

Bước build chiếm gần toàn bộ thời gian vì lần đầu phải tải image `eclipse-temurin`, Gradle và dependency. Nhờ `cache-from/cache-to: type=gha`, các lần release sau sẽ tái sử dụng layer đã có.

![Các step của job](images/02-job-steps.png)

## 6. Kết luận

- Workflow `release.yml` chỉ chạy khi push tag khớp `v*.*.*`; lệnh `git tag v1.0.0 && git push origin v1.0.0` đã kích hoạt nó thành công.
- `docker/metadata-action` tự chuyển Git tag thành 4 Docker tag (`v1.0.0`, `1.0.0`, `1.0`, `latest`) và gắn OCI label nguồn gốc.
- Image được build multi-stage và đẩy lên `ghcr.io/thang-260704/it209_ss12_ex2`, hiển thị trong GitHub Packages, có thể `docker pull` để triển khai lên VPS.
- Xác thực dùng `GITHUB_TOKEN` với `permissions.packages: write`, không cần lưu mật khẩu hay token cá nhân trong repo.
