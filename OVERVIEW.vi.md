# Tổng quan repo AX (Tiếng Việt)

> **Lưu ý:** Repo này (**AX** của Google) **không** chứa 10 AI agent, plugin cho Claude Cowork hay Vertical Plugins. AX là **hạ tầng** để *chạy* các AI agent một cách an toàn và ở quy mô lớn. Những khái niệm tương tự như agent, skills và connector (MCP) vẫn xuất hiện trong AX. Mục cuối có bảng so sánh.

## Repo có gì?

AX là một **orchestrator** (bộ điều phối) cho AI agent. Bạn mô tả công việc của agent trong một file YAML: chạy lệnh gì, cần repo và công cụ nào, được phép truy cập mạng ở đâu, dùng mô hình AI nào. AX sẽ tự động:

1. **Cô lập agent trong sandbox.** Mỗi agent chạy trong một "hộp cát" riêng, có giới hạn CPU/RAM. Nếu agent chạy code lỗi hoặc code không tin cậy thì cũng không ảnh hưởng đến hệ thống khác.
2. **Chuẩn bị môi trường làm việc (workspace).** AX clone sẵn Git repo, kết nối MCP server (công cụ) và tải skills, để agent bắt đầu làm việc ngay mà không tốn thời gian (và token) cài đặt.
3. **Rào mạng.** Agent chỉ được kết nối tới những địa chỉ nằm trong danh sách cho phép (allowlist).
4. **Chạy ở quy mô lớn.** Trạng thái được lưu trong Redis thay vì etcd của Kubernetes, nên AX có thể quản lý hàng triệu tác vụ ngắn hạn.

Cách dùng AX giống Kubernetes: viết manifest YAML rồi dùng lệnh `ax` (tương tự `kubectl`).

> ⚠️ Dự án đang ở phiên bản `v1alpha1` và có thể thay đổi lớn trước khi ổn định.

## Các khái niệm chính

| Khái niệm | Giải thích |
|---|---|
| **Agent (AI agent)** | Chương trình AI có thể tự lập kế hoạch, gọi công cụ và thực hiện nhiều bước để hoàn thành mục tiêu. Khác với chatbot chỉ trả lời một câu hỏi, agent chạy lâu, tích lũy trạng thái, có thể chia nhỏ việc và có thể tiêu tốn nhiều tiền nếu không được giám sát. |
| **`Task`** | Đơn vị nhỏ nhất: một sandbox chạy agent, gồm container image, lệnh, biến môi trường và giới hạn tài nguyên. Một agent có thể tạo ra hàng loạt Task con để chia việc. |
| **`Workspace`** | "Bàn làm việc" dựng sẵn cho agent: Git repo, MCP server, skills. Khai báo một lần, dùng cho nhiều Task. Có thể thêm `goal` (mục tiêu bằng ngôn ngữ tự nhiên, ví dụ "đảm bảo có Go toolchain"); khi đó một agent sẽ tự cài đặt phần còn lại trước khi Task chạy. |
| **`Gateway`** | Ranh giới mạng: Task mở cổng nào (listener) và được phép gọi ra host/port nào (egress allowlist). Ví dụ: chỉ cho gọi API của nhà cung cấp LLM và Git host của công ty. |
| **`Model`** | Cấu hình mô hình AI dùng chung: nhà cung cấp (mặc định Google), tên model (mặc định `gemini-3.8-flash`), tham số như `temperature`, và **Kubernetes secret** chứa API key. Muốn đổi key hoặc đổi model thì chỉ cần một lệnh `ax apply`, không phải sửa từng agent. |
| **Atespace** | Không gian tên (tương tự namespace) để nhóm tài nguyên. Mặc định là `default`. |
| **Sandbox / Agent Substrate** | Nền tảng thực thi bên dưới, do AX gọi tới để tạo "actor" (máy ảo/container cô lập), gán worker và áp chính sách mạng. |
| **MCP (Model Context Protocol)** | Giao thức chuẩn để AI kết nối với công cụ và dữ liệu bên ngoài (API, database, phần mềm). Đây chính là vai trò của **connector**. |
| **Skills** | Gói hướng dẫn (thường là file Markdown có các bước rõ ràng) để agent làm theo. AI hiểu văn bản có tiêu đề và các bước tốt hơn code, và dùng skill có sẵn giúp tiết kiệm token vì agent không phải tự mò cách làm. Trong AX, skills được tải từ **skill registry** vào đường dẫn như `/.agents/skills`. |
| **Suspend / Resume** | Tạm dừng agent đang rảnh (lưu checkpoint trạng thái bộ nhớ), giải phóng tài nguyên, rồi tiếp tục đúng chỗ cũ khi cần. |
| **Phase & Condition** | Trạng thái của Task: phase là tóm tắt một từ (`Running`, `Suspended`, `Failed`, `Terminating`); condition là chi tiết (`WorkspaceReady`, `GatewayReady`, `Ready`). Nên chờ `Ready`. |

## Có thể ứng dụng như thế nào?

- **Chạy agent tự động trong thời gian dài và ở quy mô lớn**, ví dụ hàng nghìn agent sửa bug hoặc phân tích dữ liệu song song, mỗi agent trong một sandbox riêng.
- **Bảo mật:** giới hạn chính xác những gì agent được kết nối tới (Gateway), và giữ API key trong secret thay vì để lộ trong prompt hoặc code.
- **Chuẩn hóa môi trường:** mọi agent trong công ty dùng chung Workspace (repo, công cụ MCP, skills) và chung cấu hình Model.
- **Tiết kiệm chi phí:** tự động suspend agent khi rảnh, rồi resume khi cần.
- **Debug:** `ax ssh <task>` để vào sandbox xem agent đang làm gì.
- **Tùy biến:** tự build image riêng có nhúng package `runner` để chạy agent framework của bạn.

## Cấu trúc repo

```
cmd/ax/               → CLI cho người dùng (apply, get, describe, watch, suspend, resume, ssh…)
cmd/ax-server/        → API server (gRPC, cổng 8080): kiểm tra manifest, lưu vào Redis, phát sự kiện
cmd/ax-controller/    → Worker đọc hàng đợi Redis và tạo/điều khiển sandbox trên Agent Substrate
cmd/ax-task-runner/   → Tiến trình đầu tiên trong mỗi container: dựng workspace, chạy lệnh của agent
runner/               → Logic chạy trong sandbox (có thể nhúng vào image tự build)
pkg/apis/v1alpha1/    → Định nghĩa API (ax.proto) và kiểu dữ liệu Go
internal/             → server, controller, substrate client, store (Redis/in-memory),
                        workspace (git/MCP/skills + planner dùng LLM), model (LLM client), tunnel
deploy/               → Manifest Kubernetes (Redis, server, controller)
examples/             → YAML mẫu: simple.yaml, task.yaml
docs/                 → Tài liệu khái niệm, manifest, sandbox, networking, roadmap
```

**Luồng hoạt động:** `ax apply` → `ax-server` → Redis (hàng đợi) → `ax-controller` → Agent Substrate tạo sandbox → `ax-task-runner` dựng workspace và chạy agent.

## Thư viện và công cụ phụ thuộc

- **Go 1.27.1**
- **Agent Substrate**: nền tảng sandbox
- **Redis** (`go-redis/v9`): lưu trạng thái và hàng đợi
- **gRPC/Protobuf**: giao tiếp giữa các thành phần
- **yaml.v3**: đọc manifest
- Để deploy: cụm Kubernetes, `kubectl`, `ko`, Docker/Podman, container registry
- Image mặc định của Task: Python 3.12 và `google-antigravity` (agent dùng để dựng workspace theo `goal`)

## Cách chạy

```bash
# Build và test trên máy (không cần cluster)
make build && make test

# Cài CLI
go install github.com/google/ax/cmd/ax@latest

# Deploy lên Kubernetes (namespace ax-system)
make deploy AX_IMAGE_REPO=<registry-của-bạn>

# Sử dụng
ax apply -f examples/task.yaml
ax get tasks
ax watch task <tên>
ax ssh <tên> -- ls -la /workspace     # cần spec.debug: true
ax suspend task <tên>
ax resume task <tên>
./demo.sh                             # demo toàn bộ vòng đời
```

## So sánh với các khái niệm trong đoạn tham khảo

| Khái niệm tham khảo | Tương ứng trong AX | Ghi chú |
|---|---|---|
| **Agents** (Market Researcher, KYC Screener…) | Không có sẵn agent nghiệp vụ. AX là nơi **chạy** agent: mỗi agent là một `Task`. | Bạn đưa agent của mình (image + lệnh) vào, AX lo sandbox, mạng và vòng đời. |
| **Deploy dạng Managed Agent cho việc chạy lâu dài** | Đây đúng là vai trò của AX | Suspend/resume, watch, chạy song song hàng triệu task. |
| **Tinh chỉnh prompt / ngữ cảnh công ty** | `Workspace.goal`, tham số của `Model` (có thể đặt system instruction mặc định) | Thêm ngữ cảnh để AI hiểu yêu cầu sâu hơn. |
| **Skills (file Markdown)** | `Workspace.spec.skills`, tải từ skill registry | Cùng ý tưởng: hướng dẫn dạng văn bản giúp agent làm đúng và tiết kiệm token. |
| **Connectors (API, database, Jira, Salesforce)** | `Workspace.spec.mcp` (MCP server/registry) cùng với `Gateway` | MCP kết nối công cụ; Gateway khóa những nơi agent được và không được kết nối. API key nằm trong Kubernetes secret. |
| **Vertical Plugins** (skills, connectors, commands nhưng không tự suy nghĩ) | Gần giống `Workspace`: cung cấp repo, công cụ và skills, nhưng tự nó không chạy hay ra quyết định | Chỉ khi được gắn vào một `Task` (agent) thì mới được sử dụng. |
