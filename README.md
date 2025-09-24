<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Download APK</title>
  <style>
    :root { --accent:#0b84ff; --bg:#f7f9fc; --card:#fff; }
    body{font-family:Inter, system-ui, Arial; background:var(--bg); margin:0; padding:24px; color:#122; display:flex; align-items:center; justify-content:center; min-height:100vh}
    .card{width:100%;max-width:760px;background:var(--card);border-radius:12px;box-shadow:0 6px 24px rgba(10,20,40,0.08);padding:22px}
    h1{margin:0 0 8px;font-size:20px}
    p.lead{margin:0 0 18px;color:#445;}
    .row{display:flex;gap:12px;flex-wrap:wrap;align-items:center}
    .meta{flex:1;min-width:220px}
    .btn{background:var(--accent);color:#fff;padding:10px 14px;border-radius:8px;border:0;cursor:pointer;font-weight:600}
    .btn:disabled{opacity:0.6;cursor:not-allowed}
    .progressWrap{background:#eef3ff;border-radius:10px;height:14px;overflow:hidden;margin-top:12px}
    .progress{height:100%;width:0%;background:linear-gradient(90deg,#0b84ff,#5fc0ff)}
    .info{margin-top:12px;color:#556;font-size:14px}
    .alert{padding:10px;border-radius:8px;background:#fff3cd;color:#664d03;border:1px solid #ffe8a8;margin-top:12px}
    a.direct{display:inline-block;margin-left:8px;color:var(--accent);text-decoration:none}
    small{color:#667;font-size:13px}
  </style>
</head>
<body>
  <div class="card" role="main">
    <h1>Tải APK</h1>
    <p class="lead">Trang đơn giản để tải file APK từ internet. Kiểm tra mạng, hiển thị tiến trình và cung cấp link trực tiếp.</p>

    <div class="row">
      <div class="meta">
        <label for="apkUrl"><small>URL APK</small></label><br />
        <input id="apkUrl" type="text" style="width:100%;padding:8px;border-radius:8px;border:1px solid #ddd" value="https://example.com/app.apk" />
        <div class="info" id="fileInfo">Tên: <b id="fileName">-</b> · Kích thước: <b id="fileSize">-</b></div>
      </div>

      <div style="display:flex;flex-direction:column;gap:8px;">
        <button class="btn" id="checkBtn">Kiểm tra & Hiển thị info</button>
        <button class="btn" id="downloadBtn">Tải về (Streaming)</button>
      </div>
    </div>

    <div class="progressWrap" aria-hidden="true" style="margin-top:18px;">
      <div class="progress" id="progressBar"></div>
    </div>

    <div id="status" class="info" aria-live="polite" style="margin-top:10px"></div>

    <div id="offlineNotice" class="alert" style="display:none;margin-top:12px">
      Bạn đang <b>mất kết nối</b>. Vui lòng kết nối Internet rồi thử lại.
    </div>

    <div style="margin-top:14px">
      <small>Ghi chú: nếu server không cho phép CORS (Cross-Origin Resource Sharing), một số thao tác (HEAD hoặc fetch streaming) có thể bị chặn. Trong trường hợp đó, dùng link tải trực tiếp hoặc đặt APK cùng domain với trang này.</small>
    </div>
  </div>

<script>
  // ---- Thay đổi mặc định ở đây nếu muốn
  // (Bạn vẫn có thể thay trực tiếp trong input trên giao diện)
  const defaultName = "app.apk";

  // ---- Elements
  const apkUrlInput = document.getElementById("apkUrl");
  const checkBtn = document.getElementById("checkBtn");
  const downloadBtn = document.getElementById("downloadBtn");
  const fileNameEl = document.getElementById("fileName");
  const fileSizeEl = document.getElementById("fileSize");
  const progressBar = document.getElementById("progressBar");
  const statusEl = document.getElementById("status");
  const offlineNotice = document.getElementById("offlineNotice");

  // ---- Network state
  function updateNetworkUI() {
    if (!navigator.onLine) {
      offlineNotice.style.display = "block";
      checkBtn.disabled = true;
      downloadBtn.disabled = true;
      statusEl.textContent = "Offline — kiểm tra kết nối mạng.";
    } else {
      offlineNotice.style.display = "none";
      checkBtn.disabled = false;
      downloadBtn.disabled = false;
      statusEl.textContent = "";
    }
  }
  window.addEventListener('online', updateNetworkUI);
  window.addEventListener('offline', updateNetworkUI);
  updateNetworkUI();

  // ---- Helper: format bytes
  function formatBytes(bytes) {
    if (!bytes || bytes === 0) return "-";
    const sizes = ['B','KB','MB','GB','TB'];
    const i = Math.floor(Math.log(bytes) / Math.log(1024));
    return parseFloat((bytes / Math.pow(1024, i)).toFixed(2)) + ' ' + sizes[i];
  }

  // ---- Check HEAD to get size and filename (CORS may block)
  checkBtn.addEventListener('click', async () => {
    const url = apkUrlInput.value.trim();
    if (!url) return alert("Nhập URL APK.");
    fileNameEl.textContent = defaultName;
    fileSizeEl.textContent = "Đang kiểm tra...";
    statusEl.textContent = "Gửi HEAD request để lấy thông tin...";
    try {
      const resp = await fetch(url, { method: 'HEAD' });
      if (!resp.ok) throw new Error("Server trả về " + resp.status);
      // tên file từ header content-disposition nếu có
      const cd = resp.headers.get('content-disposition');
      let name = defaultName;
      if (cd) {
        const m = /filename\*=UTF-8''(.+)$/.exec(cd) || /filename="?(.*?)"?($|;)/.exec(cd);
        if (m) name = decodeURIComponent(m[1] || m[0]);
      } else {
        // fallback: lấy từ URL
        try {
          const u = new URL(url);
          const last = u.pathname.split('/').filter(Boolean).pop();
          if (last) name = last;
        } catch(e) {}
      }
      const length = resp.headers.get('content-length');
      fileNameEl.textContent = name;
      fileSizeEl.textContent = length ? formatBytes(parseInt(length,10)) : "Không có thông tin";
      statusEl.textContent = "HEAD thành công.";
    } catch (err) {
      fileSizeEl.textContent = "-";
      statusEl.textContent = "Không lấy được thông tin (có thể do CORS hoặc server không trả HEAD): " + (err.message || err);
    }
  });

  // ---- Download bằng fetch streaming (cập nhật tiến trình)
  downloadBtn.addEventListener('click', async () => {
    const url = apkUrlInput.value.trim();
    if (!url) return alert("Nhập URL APK.");
    progressBar.style.width = '0%';
    statusEl.textContent = "Bắt đầu tải...";
    downloadBtn.disabled = true;
    checkBtn.disabled = true;

    try {
      const resp = await fetch(url);
      if (!resp.ok) throw new Error('Server trả về ' + resp.status);

      // nếu server cung cấp content-length ta có thể đo % tiến trình
      const contentLength = resp.headers.get('content-length');
      const total = contentLength ? parseInt(contentLength, 10) : null;

      // đọc stream
      const reader = resp.body && resp.body.getReader ? resp.body.getReader() : null;
      if (!reader) {
        // trình duyệt không hỗ trợ stream -> fallback: dùng redirect tới link trực tiếp
        statusEl.innerHTML = "Trình duyệt không hỗ trợ streaming. <a class='direct' href='" + encodeURI(url) + "' target='_blank'>Tải trực tiếp</a>";
        return;
      }
      const chunks = [];
      let received = 0;
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        chunks.push(value);
        received += value.length;
        if (total) {
          const pct = Math.round((received / total) * 100);
          progressBar.style.width = pct + '%';
          statusEl.textContent = `Đã tải ${formatBytes(received)} / ${formatBytes(total)} (${pct}%)`;
        } else {
          // unknown total
          progressBar.style.width = '60%';
          statusEl.textContent = `Đã tải ${formatBytes(received)} (tổng không xác định)`;
        }
      }

      // tạo blob và tự trigger download
      const blob = new Blob(chunks, { type: 'application/vnd.android.package-archive' });
      const downloadName = (fileNameEl.textContent && fileNameEl.textContent !== '-') ? fileNameEl.textContent : defaultName;
      const blobUrl = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = blobUrl;
      a.download = downloadName;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(blobUrl);
      progressBar.style.width = '100%';
      statusEl.textContent = `Hoàn tất — file: ${downloadName} (${formatBytes(blob.size)})`;
    } catch (err) {
      statusEl.textContent = "Lỗi khi tải: " + (err.message || err);
      // fallback: cung cấp link trực tiếp
      statusEl.innerHTML += " — thử <a class='direct' href='" + encodeURI(url) + "' target='_blank'>tải trực tiếp</a>";
    } finally {
      downloadBtn.disabled = false;
      checkBtn.disabled = false;
    }
  });

  // Tự cập nhật tên/url khi thay input
  apkUrlInput.addEventListener('input', () => {
    fileNameEl.textContent = '-';
    fileSizeEl.textContent = '-';
  });

</script>
</body>
</html>
