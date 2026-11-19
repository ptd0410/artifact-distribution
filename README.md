# Artifact Distribution

Repository lưu **expected metadata** cho việc phân phối resource package (artifact). Payload thực tế (ZIP, binary, assets) được phát hành qua GitHub Releases; repository này không lưu payload lớn.

## Cấu trúc

```text
artifact-distribution/
├── README.md
├── channels/
│   └── stable.json
└── manifests/
    └── main-ui/
        └── 1.0.0.json
```

- `channels/stable.json`: danh sách artifact/version **đang được công bố** trên kênh `stable`.
- `manifests/main-ui/1.0.0.json`: metadata cho version cụ thể (không sửa sau khi đã công bố; phát hành version mới khi có payload mới).
- GitHub Releases: nơi chứa `main-ui.zip` tương ứng.

## Trạng thái của bộ khởi tạo

**Chưa có payload thật.** File `manifests/main-ui/1.0.0.json` chỉ là mẫu: URL mang tính dự kiến, `sha256` và `size_bytes` chưa có giá trị thật. Vì vậy `channels/stable.json` ban đầu để `artifacts: []` để Android **không** nhận phiên bản chưa thể tải/cài đặt.

## Cách phát hành `main-ui` v1.0.0

1. Tạo file `main-ui.zip` thực tế, chạy kiểm tra bundle theo Core trước khi công bố.
2. Tính SHA-256 và kích thước **của đúng file được upload** (trên macOS):

   ```bash
   shasum -a 256 main-ui.zip
   stat -f%z main-ui.zip
   ```

3. Tạo GitHub Release tag `main-ui-v1.0.0`, upload asset `main-ui.zip`; kiểm tra URL có thể tải được.
4. Trong `manifests/main-ui/1.0.0.json`, điền `payload.url`, `payload.sha256` (64 ký tự hex) và `payload.size_bytes` (>0) bằng giá trị thật, rồi commit.
5. **Sau cùng** cập nhật `channels/stable.json` thành:

   ```json
   {
     "schema_version": 1,
     "channel": "stable",
     "artifacts": [
       {
         "kind": "web-bundle",
         "artifact_id": "main-ui",
         "version": "1.0.0",
         "manifest": "manifests/main-ui/1.0.0.json"
       }
     ]
   }
   ```

6. Android tải `stable.json` → manifest → payload, so SHA-256 trước khi install, rồi gọi Artifact Core `inspect()` để đối chiếu dữ liệu persist.

## Lưu ý về hợp đồng tích hợp

Đây là **schema metadata phân phối đề xuất tại tầng Android**, không phải schema `marker.json` trên thiết bị và cũng **chưa phải định dạng được Artifact Core tự động parse**. Khi tích hợp, Android phải kiểm tra `schema_version`, URL, version, `size_bytes`, checksum và các giá trị bị thiếu/sai định dạng; không được dùng metadata template để cài đặt.

SHA-256 đảm bảo đối chiếu bytes với metadata; nếu metadata bị thay đổi trái phép thì checksum đi kèm cũng có thể bị thay. Bản phát hành production nên xác thực chữ ký metadata bằng public key tin cậy có sẵn trong app. Không nhúng GitHub personal access token vào APK để truy cập private repo.
