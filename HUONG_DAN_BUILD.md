# Hướng dẫn build & chạy (AMD64, headless, Docker)

## 0. Việc BẮT BUỘC phải làm trước khi build

1. **Đổi mật khẩu MongoDB.** URI cũ (`baoan_dev` / `103.159.51.61`) đã bị
   dán vào chat với AI nên coi như đã lộ ra ngoài. Đổi mật khẩu trên
   MongoDB thật trước, không chỉ xoá dòng code là đủ.
2. **Copy `detectors/yolov8n.pt`** từ project gốc vào đúng vị trí
   `Project_new/detectors/yolov8n.pt` trong bộ file này (`yolo11n.pt` không
   được dùng ở đâu cả nên không cần copy).
3. **Copy `manage_cameras.py` chạy 1 lần ngoài Docker** (không cần đóng gói
   vào container) để thêm camera vào MongoDB trước khi container chạy —
   container không tự tạo camera, chỉ đọc camera có sẵn trong DB
   (`status=True`).
4. Tạo file `.env` từ `.env.example`, điền `MONGO_URI` thật.

## 1. Cấu trúc file đã chuẩn bị

```
Project_new/
├── multi_main.py            # ĐÃ SỬA: headless, thêm MJPEG stream, tắt bằng SIGTERM
├── api_server.py            # giữ nguyên
├── camera/CameraThread.py
├── database/task_db.py      # ĐÃ SỬA: bỏ hardcode Mongo URI, bắt buộc qua ENV
├── detectors/{detect_face.py, motion_detector.py, PersonDetector.py, yolov8n.pt}
├── trackers/{person_tracker.py, zone_counter.py}
├── recognition/{person_db_recognizer.py, unknown_gallery.py}
├── requirements-pipeline.txt
├── requirements-api.txt
├── Dockerfile.pipeline
├── Dockerfile.api
├── docker-compose.yml
├── .env.example
└── .dockerignore
```

Lưu ý: `build_system_regconition.py`, `face_identifier.py` (DEPRECATED,
không được `multi_main.py`/`api_server.py` import) và
`configure_zone_camera.py`, `manage_cameras.py`, `manage_persons.py`
(công cụ CLI chạy tay, cần GUI hoặc chạy 1 lần) **không** được đóng vào
Docker image — chạy trực tiếp bằng Python trên máy có sẵn khi cần cấu
hình.

## 2. Build

```bash
cd Project_new
cp .env.example .env
nano .env   # điền MONGO_URI thật

docker compose build
```

Build lần đầu sẽ khá lâu (torch + insightface + onnxruntime + faiss tải
về, vài trăm MB - 1GB). Máy build cần internet, KHÔNG cần GPU.

## 3. Chạy

```bash
docker compose up -d
docker compose logs -f camera-pipeline
```

- Xem video (nếu muốn): mở trình duyệt `http://<ip-server>:8090/stream`
  (toàn bộ lưới camera) hoặc `http://<ip-server>:8090/stream/<channel>`
  (1 camera riêng, `channel` lấy từ MongoDB `cameras.channel`).
- API tra cứu event / quản lý person, camera: `http://<ip-server>:8000/docs`
  (Swagger UI tự sinh).
- Health check: `http://<ip-server>:8090/health` và
  `http://<ip-server>:8000/health`.

Lần chạy đầu tiên, `insightface` sẽ tự tải model nhận diện khuôn mặt
(buffalo_l, ~350MB) từ GitHub về `/root/.insightface` bên trong container -
**cần internet lúc chạy**, không chỉ lúc build. Model được lưu vào volume
`insightface-cache` nên chỉ tải 1 lần, các lần restart sau không tải lại.

## 4. Dừng / restart gọn gàng

```bash
docker compose down       # gửi SIGTERM, multi_main.py đã xử lý để thoát sạch
docker compose restart camera-pipeline
```

## 5. Những điểm cần biết / rủi ro còn lại

- **CPU-only**: `ultralytics.YOLO.track()` và `insightface` mặc định chạy
  CPU (đã set `ctx_id=-1` cho face, YOLO không chỉ định device nên tự dùng
  CPU khi không có CUDA). Số lượng camera chạy đồng thời bị giới hạn bởi
  CPU thật của server - nếu FPS tụt/CPU 100% liên tục, cân nhắc giảm số
  camera/container hoặc giảm `YOLO_IMGSZ`, `det_size` trong code.
- **`ML_INFERENCE_LOCK`** ép mọi inference (mọi camera + face) chạy TUẦN
  TỰ trong 1 container để tránh segfault (đã có sẵn trong code gốc) - đây
  là lý do không nên nhồi quá nhiều camera vào 1 container
  `camera-pipeline`; nếu cần scale nhiều camera hơn khả năng 1 CPU, cân
  nhắc chạy nhiều container `camera-pipeline` (mỗi container 1 nhóm
  camera), API server dùng chung 1 MongoDB.
- **MJPEG stream** hiện phát nguyên lưới ghép hoặc từng camera raw, CHƯA
  có xác thực (ai có URL cũng xem được) - nếu chạy "online" thật, nên đặt
  sau reverse proxy (nginx/Caddy) có auth hoặc VPN, không expose thẳng ra
  internet.
- File `.pt` YOLO (~6MB) được build thẳng vào image - image sẽ không nhỏ
  (ước tính image `camera-pipeline` ~2-3GB do torch+insightface+onnxruntime).
