<div id="friction-demo">
  <style>
    #friction-demo {
      width: 100%;
      max-width: 1000px;
      margin: auto;
      font-family: Arial, sans-serif;
      color: #333;
    }

    #friction-demo .stage {
      width: 100%;
      background: #fff;
      border: 1px solid #ddd;
      border-radius: 12px;
      overflow: hidden;
    }

    #friction-demo canvas {
      width: 100%;
      height: auto;
      display: block;
      touch-action: none;
      cursor: grab;
    }

    #friction-demo .controls {
      display: flex;
      justify-content: center;
      flex-wrap: wrap;
      gap: 10px;
      padding: 14px;
      background: #f5f5f5;
    }

    #friction-demo button {
      border: 0;
      border-radius: 8px;
      padding: 11px 20px;
      font-size: 15px;
      font-weight: bold;
      cursor: pointer;
      background: #2675bd;
      color: white;
    }

    #friction-demo button:hover {
      filter: brightness(0.9);
    }

    #friction-demo button.secondary {
      background: #777;
    }

    #friction-demo button.active {
      background: #e28b26;
    }

    #friction-demo .description {
      text-align: center;
      padding: 12px;
      line-height: 1.6;
      font-size: 15px;
      background: #fff;
    }

    #friction-demo .hint {
      color: #777;
      font-size: 13px;
    }
  </style>

  <div class="stage">
    <canvas id="frictionCanvas" width="900" height="480"></canvas>

    <div class="controls">
      <button id="startFriction">▶ Bắt đầu đẩy</button>
      <button id="lensFriction" class="active">🔍 Bật kính lúp</button>
      <button id="resetFriction" class="secondary">↻ Làm lại</button>
    </div>

    <div class="description">
      <div id="frictionStatus">
        Thanh gỗ đang đứng yên trên mặt bàn.
      </div>
      <div class="hint">
        Có thể kéo kính lúp để thay đổi vị trí quan sát.
      </div>
    </div>
  </div>

  <script>
  (() => {
    const root = document.getElementById("friction-demo");
    const canvas = root.querySelector("#frictionCanvas");
    const ctx = canvas.getContext("2d");

    const W = 900, H = 480;
    const tableY = 360;
    const blockW = 145, blockH = 68;
    const startX = 150;

    let blockX = startX;
    let velocity = 0;
    const friction = 135;
    let moving = false;
    let elapsed = 0;
    let lastTime = 0;

    let showLens = false;
    let lensX = 550;
    let lensY = 165;
    const lensR = 92;
    let dragging = false;
    let pointerOffsetX = 0;
    let pointerOffsetY = 0;

    const status = root.querySelector("#frictionStatus");
    const startBtn = root.querySelector("#startFriction");
    const lensBtn = root.querySelector("#lensFriction");
    const resetBtn = root.querySelector("#resetFriction");

    function roundedRect(x, y, w, h, r) {
      ctx.beginPath();
      ctx.roundRect(x, y, w, h, r);
      ctx.fill();
    }

    // Vẽ mặt bàn
    function drawTable() {
      ctx.fillStyle = "#ad6942";
      ctx.fillRect(50, tableY, 800, 72);

      ctx.fillStyle = "#d49a73";
      ctx.fillRect(50, tableY, 800, 5);

      ctx.fillStyle = "#f1dfd1";
      ctx.fillRect(50, tableY + 72, 800, 12);

      // Đường bề mặt nhìn ở tỉ lệ thông thường
      ctx.strokeStyle = "#e0b493";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(50, tableY);
      ctx.lineTo(850, tableY);
      ctx.stroke();
    }

    // Vẽ thanh gỗ
    function drawBlock() {
      const y = tableY - blockH;

      ctx.fillStyle = "rgba(0,0,0,0.12)";
      ctx.fillRect(blockX + 5, tableY, blockW, 5);

      ctx.fillStyle = "#bd916f";
      ctx.fillRect(blockX, y, blockW, blockH);

      ctx.fillStyle = "#dfb99f";
      ctx.fillRect(blockX, y, blockW, 7);

      ctx.strokeStyle = "#87654f";
      ctx.lineWidth = 2;
      ctx.strokeRect(blockX, y, blockW, blockH);

      // Vân gỗ nhẹ
      ctx.strokeStyle = "rgba(110,70,45,0.22)";
      ctx.lineWidth = 1;
      ctx.beginPath();
      ctx.moveTo(blockX + 15, y + 25);
      ctx.lineTo(blockX + 115, y + 25);
      ctx.moveTo(blockX + 30, y + 48);
      ctx.lineTo(blockX + 130, y + 48);
      ctx.stroke();
    }

    // Vẽ bàn tay đang đẩy thanh gỗ
    function drawHand() {
      if (!moving || elapsed > 0.5) return;

      const hx = blockX - 12;
      const hy = tableY - blockH + 12;

      ctx.save();
      ctx.translate(hx, hy);

      ctx.fillStyle = "#efc1a7";
      ctx.strokeStyle = "#bd927c";
      ctx.lineWidth = 2;

      // Cánh tay
      ctx.beginPath();
      ctx.moveTo(-85, -72);
      ctx.lineTo(-57, -88);
      ctx.lineTo(-20, -24);
      ctx.lineTo(-44, -7);
      ctx.closePath();
      ctx.fill();
      ctx.stroke();

      // Bàn tay
      ctx.beginPath();
      ctx.moveTo(-60, -20);
      ctx.lineTo(-43, -35);
      ctx.lineTo(-20, -19);
      ctx.lineTo(-7, -3);
      ctx.lineTo(-15, 8);
      ctx.lineTo(-49, 2);
      ctx.closePath();
      ctx.fill();
      ctx.stroke();

      // Ngón tay
      for (let i = 0; i < 3; i++) {
        ctx.beginPath();
        ctx.moveTo(-47 + i * 10, -31 + i * 5);
        ctx.lineTo(-29 + i * 10, -13 + i * 4);
        ctx.stroke();
      }

      ctx.restore();
    }

    // Tạo bề mặt lồi lõm trong kính lúp
    function drawRoughSurface(cx, cy, r) {
      ctx.save();

      ctx.beginPath();
      ctx.arc(cx, cy, r - 3, 0, Math.PI * 2);
      ctx.clip();

      // Phần gỗ phía trên
      ctx.fillStyle = "#e9c4a8";
      ctx.fillRect(cx - r, cy - r, 2 * r, r + 10);

      // Phần mặt bàn phía dưới
      ctx.fillStyle = "#a96c45";
      ctx.fillRect(cx - r, cy, 2 * r, r);

      // Các điểm lồi lõm ở bề mặt thanh gỗ
      ctx.fillStyle = "#c18d68";
      ctx.beginPath();
      ctx.moveTo(cx - r, cy - 5);

      for (let x = -r; x <= r; x += 5) {
        const y = cy - 7
          + Math.sin(x * 0.19) * 5
          + Math.sin(x * 0.43) * 3;
        ctx.lineTo(cx + x, y);
      }

      ctx.lineTo(cx + r, cy + 4);
      ctx.lineTo(cx - r, cy + 4);
      ctx.closePath();
      ctx.fill();

      // Các điểm lồi lõm của mặt bàn
      ctx.fillStyle = "#885333";
      ctx.beginPath();
      ctx.moveTo(cx - r, cy + 9);

      for (let x = -r; x <= r; x += 5) {
        const y = cy + 12
          + Math.sin(x * 0.22 + 1) * 6
          + Math.cos(x * 0.47) * 3;
        ctx.lineTo(cx + x, y);
      }

      ctx.lineTo(cx + r, cy + r);
      ctx.lineTo(cx - r, cy + r);
      ctx.closePath();
      ctx.fill();

      // Làm nổi bật các điểm tiếp xúc
      ctx.fillStyle = "#70452f";
      for (let i = -75; i <= 75; i += 20) {
        const x = cx + i;
        const y = cy + Math.sin(i * 0.2) * 4 + 3;

        ctx.beginPath();
        ctx.arc(x, y, 3.2, 0, Math.PI * 2);
        ctx.fill();
      }

      ctx.restore();
    }

    // Vẽ kính lúp
    function drawLens() {
      if (!showLens) return;

      const targetX = blockX + blockW / 2;
      const targetY = tableY;

      // Hai đường nối vùng tiếp xúc với kính lúp
      ctx.strokeStyle = "#555";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(targetX, targetY);
      ctx.lineTo(lensX - lensR * 0.68, lensY + lensR * 0.72);
      ctx.moveTo(targetX, targetY);
      ctx.lineTo(lensX + lensR * 0.68, lensY + lensR * 0.72);
      ctx.stroke();

      // Vùng phóng đại
      drawRoughSurface(lensX, lensY, lensR);

      // Viền kính
      ctx.beginPath();
      ctx.arc(lensX, lensY, lensR, 0, Math.PI * 2);
      ctx.strokeStyle = "#333";
      ctx.lineWidth = 4;
      ctx.stroke();

      // Ánh sáng kính
      ctx.strokeStyle = "rgba(255,255,255,0.8)";
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.arc(
        lensX - 10, lensY - 10,
        lensR - 7,
        Math.PI * 1.05,
        Math.PI * 1.42
      );
      ctx.stroke();

      // Chú thích
      ctx.fillStyle = "#333";
      ctx.font = "bold 14px Arial";
      ctx.textAlign = "center";
      ctx.fillText(
        "Bề mặt phóng đại",
        lensX,
        lensY - lensR - 13
      );
    }

    function drawLabels() {
      ctx.fillStyle = "#333";
      ctx.font = "bold 17px Arial";
      ctx.textAlign = "center";
      ctx.fillText("THANH GỖ", blockX + blockW / 2, tableY - blockH - 15);

      ctx.font = "16px Arial";
      ctx.fillText("MẶT BÀN", 760, tableY + 40);

      ctx.textAlign = "left";
      ctx.font = "14px Arial";
      ctx.fillStyle = "#777";
      ctx.fillText(
        "Quan sát thông thường: bề mặt có vẻ nhẵn",
        55, 30
      );
    }

    function render() {
      ctx.clearRect(0, 0, W, H);

      // Nền
      ctx.fillStyle = "#ffffff";
      ctx.fillRect(0, 0, W, H);

      drawTable();
      drawBlock();
      drawHand();
      drawLabels();
      drawLens();
    }

    function animate(timestamp) {
      if (!moving) return;

      if (!lastTime) lastTime = timestamp;
      const dt = Math.min((timestamp - lastTime) / 1000, 0.04);
      lastTime = timestamp;

      elapsed += dt;

      // Chuyển động chậm dần do lực ma sát
      velocity = Math.max(0, velocity - friction * dt);
      blockX += velocity * dt;

      if (velocity <= 0) {
        velocity = 0;
        moving = false;
        lastTime = 0;
        status.textContent =
          "Thanh gỗ đã dừng lại do tác dụng của lực ma sát.";
      } else {
        status.textContent =
          "Thanh gỗ đang trượt và chậm dần do lực ma sát.";
      }

      render();

      if (moving) {
        requestAnimationFrame(animate);
      }
    }

    // Bắt đầu đẩy
    startBtn.addEventListener("click", () => {
      if (moving) return;

      if (blockX > startX + 5) {
        blockX = startX;
      }

      velocity = 360;
      elapsed = 0;
      moving = true;
      lastTime = 0;

      status.textContent =
        "Tay tác dụng lực đẩy, thanh gỗ bắt đầu chuyển động.";

      requestAnimationFrame(animate);
    });

    // Bật / tắt kính lúp
    lensBtn.addEventListener("click", () => {
      showLens = !showLens;
      lensBtn.classList.toggle("active", showLens);
      lensBtn.textContent = showLens
        ? "🔍 Tắt kính lúp"
        : "🔍 Bật kính lúp";

      status.textContent = showLens
        ? "Quan sát bề mặt tiếp xúc: hai bề mặt có nhiều chỗ lồi lõm."
        : "Bề mặt nhìn bằng mắt thường có vẻ nhẵn.";

      render();
    });

    // Đặt lại thí nghiệm
    resetBtn.addEventListener("click", () => {
      moving = false;
      velocity = 0;
      elapsed = 0;
      lastTime = 0;
      blockX = startX;
      lensX = 550;
      lensY = 165;

      status.textContent =
        "Thanh gỗ đang đứng yên trên mặt bàn.";

      render();
    });

    // Kéo kính lúp
    function getPointerPosition(e) {
      const rect = canvas.getBoundingClientRect();
      return {
        x: (e.clientX - rect.left) * W / rect.width,
        y: (e.clientY - rect.top) * H / rect.height
      };
    }

    canvas.addEventListener("pointerdown", e => {
      if (!showLens) return;

      const p = getPointerPosition(e);
      const distance = Math.hypot(p.x - lensX, p.y - lensY);

      if (distance < lensR) {
        dragging = true;
        pointerOffsetX = p.x - lensX;
        pointerOffsetY = p.y - lensY;
        canvas.setPointerCapture(e.pointerId);
        canvas.style.cursor = "grabbing";
      }
    });

    canvas.addEventListener("pointermove", e => {
      if (!dragging) return;

      const p = getPointerPosition(e);

      lensX = Math.max(lensR, Math.min(W - lensR, p.x - pointerOffsetX));
      lensY = Math.max(lensR + 20, Math.min(285, p.y - pointerOffsetY));

      render();
    });

    function stopDragging() {
      dragging = false;
      canvas.style.cursor = "grab";
    }

    canvas.addEventListener("pointerup", stopDragging);
    canvas.addEventListener("pointercancel", stopDragging);

    render();
  })();
  </script>
</div>
# KHBD-ICT
