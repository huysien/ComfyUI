# Cài đặt ComfyUI cho máy này

Ghi lại các bước để dựng lại ComfyUI từ đầu, ví dụ khi phải cài lại hệ điều hành.

## Máy

| Thành phần | Giá trị |
|---|---|
| OS | Ubuntu 26.04 LTS, kernel 7.0 |
| GPU | AMD Radeon RX 9070 XT (RDNA4, `gfx1201`) |
| RAM | 32 GB |
| Python | 3.13 (cài bằng uv; Python hệ thống 3.14 không dùng) |
| PyTorch | 2.15.0 + ROCm 10.0 (HIP 7.15) |

Kernel 7.0 đã có sẵn driver `amdgpu` cho RDNA4, nên **không cần cài driver hay ROCm vào hệ thống**. Gói PyTorch ROCm đã kèm sẵn thư viện ROCm.

## 1. Quyền dùng GPU

```bash
sudo usermod -aG render,video $USER
```

Đăng xuất rồi đăng nhập lại (hoặc khởi động lại máy). Kiểm tra bằng `groups`: kết quả phải có `render` và `video`.

## 2. Tải nhanh hơn với Cloudflare WARP (nên làm)

Gói PyTorch được tải từ `download-r2.pytorch.org`, nằm trên mạng của Cloudflare. Nếu không có WARP, kết nối đi qua máy chủ ở Hồng Kông và khá chậm. Bật WARP thì nhanh hơn đáng kể.

```bash
warp-cli connect
```

Không bật hoặc tắt WARP khi đang tải, vì kết nối sẽ bị đứt và uv phải tải lại gói đang dở.

## 3. Cài uv và Python 3.13

```bash
curl -LsSf https://astral.sh/uv/install.sh | UV_NO_MODIFY_PATH=1 sh   # cài vào ~/.local/bin
uv python install 3.13
```

## 4. Lấy mã nguồn

```bash
git clone -b local https://github.com/huysien/ComfyUI.git ~/Works/ComfyUI
cd ~/Works/ComfyUI
git remote add upstream https://github.com/Comfy-Org/ComfyUI.git
```

## 5. Tạo venv và cài thư viện

```bash
cd ~/Works/ComfyUI
uv venv --python 3.13

# PyTorch ROCm: phải cài TRƯỚC requirements.txt, nếu không pip sẽ kéo bản cho NVIDIA
uv pip install torch torchvision "torchaudio==2.11.1" \
  --index-url https://download.pytorch.org/whl/rocm10.0

uv pip install -r requirements.txt -r manager_requirements.txt
```

Ghi chú:
- README chính thức ghi `rocm7.2` vì chưa được cập nhật. `rocm10.0` là bản stable mới nhất và hỗ trợ RDNA4 tốt hơn.
- Phải ghi rõ `torchaudio==2.11.1`. Kho `rocm10.0` liệt kê cả các bản torchaudio rất cũ, nên nếu không ghi phiên bản, uv có thể dò rất lâu.
- `.venv` nặng khoảng 18 GB, chủ yếu là thư viện ROCm.
- Nếu cài lỗi kiểu hash mismatch hoặc corrupt, chạy `uv cache clean torch torchvision torchaudio` rồi cài lại.

## 6. Kiểm tra

```bash
uv run python -c "import torch, torchaudio; print(torch.__version__, torch.version.hip, torchaudio.__version__)"
# 2.15.0+rocm10.0 7.15.xxxxx 2.11.1+rocm10.0

uv run python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
# True AMD Radeon RX 9070 XT
```

Trên bản ROCm, PyTorch vẫn dùng tên `torch.cuda` và thiết bị `"cuda"`, nhưng thực chất chạy bằng HIP/ROCm. `torch.version.hip` khác `None` nghĩa là đang dùng bản ROCm.

## 7. Chạy

```bash
cd ~/Works/ComfyUI
uv run python main.py --enable-manager
```

Mở http://127.0.0.1:8188 trong trình duyệt.

- Có thể thêm `PYTORCH_TUNABLEOP_ENABLED=1` vào trước lệnh để tăng tốc. Lần chạy đầu sẽ rất chậm vì ROCm phải dò cấu hình tối ưu.
- **Không** dùng `HSA_OVERRIDE_GFX_VERSION`. Biến này chỉ dành cho card ROCm không hỗ trợ chính thức, còn RX 9070 XT được hỗ trợ.

## Sau khi lắp hoặc thay card đồ hoạ

1. Khởi động lại máy, rồi kiểm tra máy đã nhận card:
   ```bash
   lspci | grep -iE 'vga|display'
   # phải thấy Navi 48 / RX 9070 XT
   ```
2. Kiểm tra quyền: `groups` phải có `render` và `video`. Nếu chưa có, làm lại mục 1.
3. Kiểm tra PyTorch nhận card (lệnh thứ hai ở mục 6). Kết quả phải là `True` cùng tên card.
4. Không chạy `main.py` khi máy vẫn đang lắp card cũ không được ROCm hỗ trợ, như dòng RX 4xx/5xx (Polaris). ComfyUI sẽ báo lỗi hoặc tự chuyển sang chạy CPU.
5. Nếu đổi sang card thuộc kiến trúc khác (ví dụ một card NVIDIA), phải cài lại PyTorch với `--index-url` phù hợp. Card NVIDIA dùng kho `cu130`, xem README.

## Sự cố đã gặp và cách xử lý

| Sự cố | Nguyên nhân | Cách xử lý |
|---|---|---|
| Tải PyTorch rất chậm | Kết nối tới Cloudflare đi qua máy chủ Hồng Kông, đường quốc tế của nhà mạng bị nghẽn | `warp-cli connect` (mục 2) |
| Bật hoặc tắt WARP giữa chừng làm hỏng lần tải đang chạy | Đổi đường mạng nên kết nối bị đứt | Chạy lại lệnh cài. Gói đã tải xong vẫn còn trong cache. Nếu báo hash mismatch hoặc corrupt: `uv cache clean torch torchvision torchaudio` |
| `uv pip install` hoặc `--dry-run` đứng im rất lâu | Có một lệnh uv khác đang chạy và giữ khoá cache, hoặc chưa ghi rõ phiên bản torchaudio | Chờ lệnh uv kia xong. Luôn ghi `torchaudio==2.11.1` |
| Gõ `pip install` báo `externally-managed-environment` | Ubuntu chặn cài thư viện vào Python hệ thống | Luôn dùng `uv pip` trong `~/Works/ComfyUI`, vì uv tự tìm `.venv` |
| `torch.version.hip` là `None`, ComfyUI chạy bằng CPU | Cài nhầm bản PyTorch cho NVIDIA hoặc CPU, thường do chạy `requirements.txt` trước khi cài PyTorch | `uv pip uninstall torch torchvision torchaudio`, rồi làm lại mục 5 theo đúng thứ tự |
| `source .venv/bin/activate` chạy trong Claude Code (gõ `!`) nhưng không có tác dụng | Mỗi lệnh `!` chạy trong một shell riêng | Dùng `uv run python ...` thay cho `python ...` |
| `torch.cuda.is_available()` là `False` dù đã lắp card | Chưa có quyền `render`/`video`, hoặc chưa khởi động lại máy sau khi thêm quyền | Mục 1, rồi đăng xuất và đăng nhập lại |

## Sao lưu trước khi cài lại hệ điều hành

Các thư mục sau không nằm trong git (đã bị `.gitignore` loại trừ), nên phải tự sao lưu:

| Thư mục | Nội dung |
|---|---|
| `models/` | Checkpoint, LoRA, VAE… (nặng nhất, tải lại rất lâu) |
| `custom_nodes/` | Custom node đã cài |
| `user/` | Cài đặt và workflow đã lưu |
| `input/`, `output/` | Ảnh đầu vào và ảnh đã tạo |

Không cần sao lưu `.venv`: cứ tạo lại bằng bước 5.

## Cập nhật từ upstream

```bash
git fetch upstream
git merge upstream/master        # hoặc: git rebase upstream/master
uv pip install -r requirements.txt -r manager_requirements.txt
```
