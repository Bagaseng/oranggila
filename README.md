<!-- <!DOCTYPE html> -->
<html lang="id">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>QRIS Manual Tool + CRC</title>
    <script src="https://cdn.jsdelivr.net/npm/jsqr@1.4.0/dist/jsQR.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/qrcode/build/qrcode.min.js"></script>
    <style>
      :root {
        --blue: #1976d2;
        --blue-dark: #0d47a1;
      }
      body {
        margin: 0;
        font-family: "Segoe UI", sans-serif;
        background: linear-gradient(to bottom, var(--blue) 45%, #fff 45%);
        color: #0d47a1;
        display: flex;
        flex-direction: column;
        align-items: center;
        min-height: 100vh;
      }
      header {
        color: white;
        font-weight: bold;
        font-size: 22px;
        margin: 20px 0;
        text-align: center;
      }
      main {
        background: white;
        width: 90%;
        max-width: 420px;
        border-radius: 16px;
        padding: 20px;
        box-shadow: 0 4px 14px rgba(0, 0, 0, 0.1);
      }
      section {
        margin-bottom: 24px;
      }
      input[type="file"] {
        display: block;
        margin: 10px auto;
      }
      textarea {
        width: 100%;
        height: 100px;
        border: 2px solid #bbdefb;
        border-radius: 8px;
        padding: 10px;
        font-size: 14px;
        color: #0d47a1;
        resize: none;
      }
      button {
        display: block;
        width: 100%;
        border: none;
        border-radius: 8px;
        padding: 12px;
        font-weight: bold;
        margin-top: 10px;
        background: var(--blue);
        color: white;
        font-size: 15px;
      }
      button:hover {
        background: var(--blue-dark);
      }
      #status {
        margin-top: 10px;
        font-weight: bold;
        text-align: center;
      }
      #preview {
        text-align: center;
        margin-top: 16px;
      }
      hr {
        border: none;
        border-top: 1px solid #bbdefb;
        margin: 24px 0;
      }
    </style>
  </head>
  <body>
    <header>QRIS Manual Tool</header>
    <main>
      <section id="decodeSection">
        <h3 style="text-align: center">📥 Upload Gambar QR</h3>
        <input type="file" accept="image/*" id="fileInput" />
        <div id="status"></div>
        <textarea
          id="decodedText"
          placeholder="Hasil decode muncul otomatis..."
          readonly
        ></textarea>
        <button id="copyBtn">📋 Copy Decode</button>
      </section>

      <hr />

      <section id="generateSection">
        <h3 style="text-align: center">🔲 Tempel & Buat QR</h3>
        <textarea
          id="qrisText"
          placeholder="Tempel atau edit kode QRIS di sini..."
        ></textarea>
        <button id="crcBtn">🔧 Generate</button>
        <button id="generateBtn">Buat QR dari Teks</button>
        <div id="preview"></div>
        <button id="downloadBtn" style="display: none">⬇ Download QR</button>
      </section>
    </main>

    <script>
      let currentCanvas = null;

      // --- CRC16-CCITT (EMVCo Standard) ---
      function crc16ccitt(str) {
        let crc = 0xffff;
        for (let i = 0; i < str.length; i++) {
          crc ^= str.charCodeAt(i) << 8;
          for (let j = 0; j < 8; j++) {
            if (crc & 0x8000) crc = (crc << 1) ^ 0x1021;
            else crc <<= 1;
            crc &= 0xffff;
          }
        }
        return crc.toString(16).toUpperCase().padStart(4, "0");
      }

      // --- Decode otomatis setelah upload ---
      document.getElementById("fileInput").addEventListener("change", (e) => {
        const file = e.target.files[0];
        if (!file) return;
        const reader = new FileReader();
        reader.onload = function (ev) {
          const img = new Image();
          img.onload = function () {
            const canvas = document.createElement("canvas");
            const ctx = canvas.getContext("2d");
            canvas.width = img.width;
            canvas.height = img.height;
            ctx.drawImage(img, 0, 0);
            const imageData = ctx.getImageData(
              0,
              0,
              canvas.width,
              canvas.height
            );
            const code = jsQR(imageData.data, canvas.width, canvas.height);
            if (code) {
              document.getElementById("decodedText").value = code.data;
              document.getElementById("status").innerText =
                "✅ QR berhasil didecode otomatis!";
            } else {
              document.getElementById("status").innerText =
                "❌ Gagal membaca QR!";
            }
          };
          img.src = ev.target.result;
        };
        reader.readAsDataURL(file);
      });

      // --- Copy hasil decode ---
      document.getElementById("copyBtn").addEventListener("click", () => {
        const text = document.getElementById("decodedText").value;
        if (!text) return alert("Belum ada teks untuk disalin!");
        navigator.clipboard.writeText(text);
        alert("✅ Teks QR berhasil disalin!");
      });

      // --- Generate CRC valid EMVCo ---
      document.getElementById("crcBtn").addEventListener("click", () => {
        let text = document.getElementById("qrisText").value.trim();
        if (!text)
          return alert("Masukkan atau tempel teks QR terlebih dahulu!");
        if (!text.includes("6304"))
          return alert("Format tidak valid: tag 6304 tidak ditemukan!");
        const base = text.substring(0, text.indexOf("6304") + 4);
        const crc = crc16ccitt(base);
        const valid = base + crc;
        document.getElementById("qrisText").value = valid;
        alert("✅ CRC diperbarui: " + crc);
      });

      // --- Generate QR dari teks ---
      document.getElementById("generateBtn").addEventListener("click", () => {
        const text = document.getElementById("qrisText").value.trim();
        if (!text) return alert("Tempel teks QR dulu!");
        const preview = document.getElementById("preview");
        preview.innerHTML = "";
        QRCode.toCanvas(text, { width: 260 }, (err, canvas) => {
          if (err) return alert("Gagal membuat QR!");
          preview.appendChild(canvas);
          currentCanvas = canvas;
          document.getElementById("downloadBtn").style.display = "block";
        });
      });

      // --- Download QR ---
      document.getElementById("downloadBtn").addEventListener("click", () => {
        if (!currentCanvas) return;
        const link = document.createElement("a");
        link.download = "QR_Generated.png";
        link.href = currentCanvas.toDataURL("image/png");
        link.click();
      });
    </script>
  </body>
</html>
