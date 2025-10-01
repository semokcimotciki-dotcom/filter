<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8" />
  <title>Kamera Filter - Auto Switch 1.5s</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <style>
    html, body {
      margin:0; padding:0; height:100%; width:100%;
      background:#000; overflow:hidden;
    }
    canvas {
      position:absolute;
      top:0; left:0;
      width:100%; height:100%;
      object-fit:cover;
    }
    .controls {
      position:absolute;
      bottom:20px; left:50%;
      transform:translateX(-50%);
      display:flex; gap:8px; flex-wrap:wrap; justify-content:center;
      background:rgba(0,0,0,0.5);
      padding:10px; border-radius:10px;
    }
    button {
      padding:10px 16px; border:none; border-radius:6px; cursor:pointer;
      font-weight:bold; background:#333; color:#eee; font-size:15px;
    }
    button:hover { background:#555; }
    .toast {
      position: fixed;
      bottom: 80px;
      left: 50%;
      transform: translateX(-50%);
      background: rgba(0,0,0,0.8);
      color: #fff;
      padding: 10px 16px;
      border-radius: 8px;
      font-size: 14px;
      opacity: 0;
      pointer-events: none;
      transition: opacity 0.5s ease;
    }
    .toast.show { opacity: 1; }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>

  <div class="controls">
    <button onclick="switchCamera()">Putar Kamera</button>
    <button onclick="takePhoto()">📸 Foto</button>
    <button id="recordBtn" onclick="toggleRecording()">🎥 Mulai Rekam</button>
  </div>

  <div id="toast" class="toast"></div>

  <script>
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d', { willReadFrequently: true });
    let currentFilter = 'normal';

    let video = document.createElement('video');
    video.autoplay = true;
    video.playsInline = true;
    let currentStream = null;
    let facingNow = "environment";

    let recorder = null;
    let recordedChunks = [];
    let isRecording = false;

    // Daftar filter yang akan diputar otomatis
    const autoFilters = ["blue", "green", "redStrong"];
    let autoIndex = 0;
    let autoSwitch = true; // bisa dimatikan kalau tidak mau otomatis

    function setFilter(f) { currentFilter = f; }

    async function startCamera(facing = "environment") {
      if (currentStream) {
        currentStream.getTracks().forEach(track => track.stop());
      }
      try {
        const stream = await navigator.mediaDevices.getUserMedia({
          video: { facingMode: facing },
          audio: false
        });
        video.srcObject = stream;
        currentStream = stream;
        video.onloadedmetadata = () => {
          resizeCanvas();
          requestAnimationFrame(drawFrame);
        };
      } catch (err) {
        alert("Tidak bisa mengakses kamera: " + err.name + " - " + err.message);
      }
    }

    async function switchCamera() {
      try {
        const devices = await navigator.mediaDevices.enumerateDevices();
        const cams = devices.filter(d => d.kind === "videoinput");
        if (cams.length < 2) {
          alert("Hanya ada 1 kamera di perangkat ini");
          return;
        }
        facingNow = (facingNow === "user") ? "environment" : "user";
        startCamera(facingNow);
      } catch (err) {
        alert("Gagal mengecek kamera: " + err.name + " - " + err.message);
      }
    }

    function resizeCanvas() {
      canvas.width = window.innerWidth;
      canvas.height = window.innerHeight;
    }
    window.addEventListener('resize', resizeCanvas);

    function drawFrame() {
      if (video.readyState >= 2) {
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
        let frame = ctx.getImageData(0, 0, canvas.width, canvas.height);
        const data = frame.data;

        for (let i = 0; i < data.length; i += 4) {
          if (currentFilter === 'blue') {
            data[i]   = Math.round(data[i] * 0.2);
            data[i+1] = Math.round(data[i+1] * 0.2);
            data[i+2] = Math.min(255, data[i+2] * 1.5);
          } else if (currentFilter === 'green') {
            data[i]   = 0;
            data[i+2] = 0;
          } else if (currentFilter === 'redStrong') {
            data[i]   = Math.min(255, data[i] * 2.0);
            data[i+1] = 0;
            data[i+2] = 0;
          }
        }

        ctx.putImageData(frame, 0, 0);
      }
      requestAnimationFrame(drawFrame);
    }

    function takePhoto() {
      const image = canvas.toDataURL("image/png");
      const link = document.createElement('a');
      link.href = image;
      link.download = "foto_filter.png";
      link.click();
      showToast("Foto berhasil disimpan 📸");
    }

    function toggleRecording() {
      const recordBtn = document.getElementById("recordBtn");
      if (!isRecording) {
        recordedChunks = [];
        const stream = canvas.captureStream(30);
        recorder = new MediaRecorder(stream, { mimeType: "video/webm" });

        recorder.ondataavailable = e => {
          if (e.data.size > 0) recordedChunks.push(e.data);
        };
        recorder.onstop = saveWebm;

        recorder.start();
        isRecording = true;
        recordBtn.textContent = "⏹ Stop Rekam";
      } else {
        recorder.stop();
        isRecording = false;
        recordBtn.textContent = "🎥 Mulai Rekam";
      }
    }

    function saveWebm() {
      const blob = new Blob(recordedChunks, { type: "video/webm" });
      const url = URL.createObjectURL(blob);
      const link = document.createElement("a");
      link.href = url;
      link.download = "video_filter.webm";
      link.click();
      URL.revokeObjectURL(url);
      showToast("Video berhasil disimpan 🎉");
    }

    function showToast(msg) {
      const toast = document.getElementById("toast");
      toast.textContent = msg;
      toast.classList.add("show");
      setTimeout(() => toast.classList.remove("show"), 3000);
    }

    // 🔄 fungsi ganti filter otomatis setiap 1,5 detik
    function autoSwitchFilter() {
      if (!autoSwitch) return;
      setFilter(autoFilters[autoIndex]);
      autoIndex = (autoIndex + 1) % autoFilters.length;
      setTimeout(autoSwitchFilter, 1500); // ⏱️ ubah ke 1.5 detik
    }

    startCamera("environment");
    autoSwitchFilter(); // mulai otomatis
  </script>
</body>
</html>
