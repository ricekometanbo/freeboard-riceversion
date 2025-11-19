<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>曲線補間＋ピンチズームホワイトボード</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <style>
    body { margin:0; font-family:sans-serif; }
    .controls { text-align:center; padding:10px; display:flex; gap:8px; flex-wrap:wrap; justify-content:center; align-items:center; }
    #board { border:1px solid #ccc; display:block; margin:10px auto; background:#fff; touch-action:none; }
    .pen-button { width:34px; height:34px; border:1px solid #bbb; border-radius:6px; background:#fff; cursor:pointer; }
    .color-swatch { width:24px; height:24px; border-radius:4px; margin:auto; }
    .active { background:#1976d2; color:#fff; }
    .btn { padding:6px 8px; border-radius:4px; cursor:pointer; }
    .btn.ghost { background:#fff; border:1px solid #1976d2; color:#1976d2; }
  </style>
</head>
<body>
  <div class="controls">
    <input type="color" id="colorPicker" value="#000000">
    <input type="range" id="sizePicker" min="1" max="40" value="5">
    <button id="addPreset" class="btn ghost">プリセット追加</button>
    <button id="eraserBtn" class="btn ghost">消しゴム</button>
    <button id="undoBtn" class="btn ghost">戻る</button>
    <button id="clearBtn" class="btn ghost">全削除</button>
    <div id="presetContainer" style="display:flex; gap:6px;"></div>
  </div>

  <canvas id="board" width="1000" height="600"></canvas>

  <script>
    const canvas = document.getElementById('board');
    const ctx = canvas.getContext('2d');

    const colorPicker = document.getElementById('colorPicker');
    const sizePicker  = document.getElementById('sizePicker');
    const addPresetBtn = document.getElementById('addPreset');
    const presetContainer = document.getElementById('presetContainer');
    const eraserBtn = document.getElementById('eraserBtn');
    const clearBtn  = document.getElementById('clearBtn');
    const undoBtn   = document.getElementById('undoBtn');

    let penColor = colorPicker.value;
    let penSize  = Number(sizePicker.value);
    let eraserMode = false;

    let isDrawing = false;
    let isZooming = false;
    let points = [];
    let activePresetIndex = null;

    // ズーム（内部座標系。canvas要素サイズは変更しない）
    let scale = 1;
    let lastDistance = null;

    // 履歴
    let history = [];

    // プリセット
    const presets = [
      { color:'#000000', size:5 },
      { color:'#ff0000', size:10 },
      { color:'#00aa00', size:3 }
    ];

    function renderPresets(){
      presetContainer.innerHTML = '';
      presets.forEach((p, idx) => {
        const btn = document.createElement('button');
        btn.className = 'pen-button';
        if (idx === activePresetIndex) btn.classList.add('active');
        btn.innerHTML = `<div class="color-swatch" style="background:${p.color}"></div>`;
        btn.addEventListener('click', () => {
          activePresetIndex = idx;
          penColor = p.color;
          penSize  = p.size;
          eraserMode = false;
          colorPicker.value = p.color;
          sizePicker.value  = p.size;
          updateActiveUI();
        });
        presetContainer.appendChild(btn);
      });
    }

    function updateActiveUI(){
      Array.from(presetContainer.children).forEach((b,i) => b.classList.toggle('active', i === activePresetIndex));
      eraserBtn.classList.toggle('active', eraserMode);
    }

    addPresetBtn.addEventListener('click', () => {
      presets.push({ color: colorPicker.value, size: Number(sizePicker.value) });
      renderPresets();
      updateActiveUI();
    });

    colorPicker.addEventListener('input', () => {
      penColor = colorPicker.value;
      eraserMode = false;
      if (activePresetIndex !== null) {
        presets[activePresetIndex].color = penColor;
        renderPresets();
        updateActiveUI();
      }
    });

    sizePicker.addEventListener('input', () => {
      penSize = Number(sizePicker.value);
      if (activePresetIndex !== null) {
        presets[activePresetIndex].size = penSize;
        renderPresets();
        updateActiveUI();
      }
    });

    eraserBtn.addEventListener('click', () => {
      eraserMode = !eraserMode;
      if (eraserMode) activePresetIndex = null;
      updateActiveUI();
    });

    clearBtn.addEventListener('click', () => {
      ctx.setTransform(1,0,0,1,0,0);
      ctx.clearRect(0,0,canvas.width,canvas.height);
      history = [];
    });

    undoBtn.addEventListener('click', () => { undo(); });

    function undo(){
      if (history.length > 0) {
        history.pop();
        ctx.setTransform(1,0,0,1,0,0);
        ctx.clearRect(0,0,canvas.width,canvas.height);
        if (history.length > 0) {
          ctx.putImageData(history[history.length-1], 0, 0);
        }
      }
    }

    // 曲線補間（Catmull-Rom の Bezier近似）でノードエッジ描画
    function drawEdges(){
      if (points.length < 2) return;

      // 中心に向けた拡縮（内部座標系）
      const tx = canvas.width  / 2 * (1 - scale);
      const ty = canvas.height / 2 * (1 - scale);
      ctx.save();
      ctx.setTransform(scale, 0, 0, scale, tx, ty);

      ctx.lineCap = 'round';
      ctx.lineJoin = 'round';
      ctx.strokeStyle = eraserMode ? '#fff' : penColor;
      ctx.lineWidth = penSize;

      ctx.beginPath();
      ctx.moveTo(points[0].x, points[0].y);

      if (points.length < 4) {
        // 点が少ないときは直線でつなぐ
        for (let i = 1; i < points.length; i++) {
          ctx.lineTo(points[i].x, points[i].y);
        }
      } else {
        for (let i = 1; i < points.length - 2; i++) {
          const p0 = points[i - 1];
          const p1 = points[i];
          const p2 = points[i + 1];
          const p3 = points[i + 2];
          const cp1x = p1.x + (p2.x - p0.x) / 6;
          const cp1y = p1.y + (p2.y - p0.y) / 6;
          const cp2x = p2.x - (p3.x - p1.x) / 6;
          const cp2y = p2.y - (p3.y - p1.y) / 6;
          ctx.bezierCurveTo(cp1x, cp1y, cp2x, cp2y, p2.x, p2.y);
        }
      }

      ctx.stroke();
      ctx.restore();
    }

    function getCoords(e){
      const rect = canvas.getBoundingClientRect();
      return { x: e.clientX - rect.left, y: e.clientY - rect.top };
    }

    // マウス描画
    canvas.addEventListener('mousedown', (e) => {
      if (isZooming) return;
      isDrawing = true;
      points = [ getCoords(e) ];
    });

    canvas.addEventListener('mousemove', (e) => {
      if (isZooming || !isDrawing) return;
      points.push(getCoords(e));
      // クリアして再描画（現在のストロークのみ）
      ctx.setTransform(1,0,0,1,0,0);
      ctx.clearRect(0,0,canvas.width,canvas.height);
      drawEdges();
    });

    canvas.addEventListener('mouseup', () => {
      if (isDrawing) {
        // 完了したストロークを履歴へ
        history.push(ctx.getImageData(0,0,canvas.width,canvas.height));
      }
      isDrawing = false;
      points = [];
    });

    // タッチ描画＋二本指ズーム（ズーム中は描画停止）
    canvas.addEventListener('touchstart', (e) => {
      if (e.touches.length === 1) {
        isZooming = false;
        const t = e.touches[0];
        const rect = canvas.getBoundingClientRect();
        points = [{ x: t.clientX - rect.left, y: t.clientY - rect.top }];
        isDrawing = true;
      } else if (e.touches.length === 2) {
        // 二本指でズームモードへ
        isZooming = true;
        isDrawing = false;
        lastDistance = null; // 新規ピンチ開始
      }
    });

    canvas.addEventListener('touchmove', (e) => {
      if (isZooming && e.touches.length === 2) {
        e.preventDefault();
        const dx = e.touches[0].clientX - e.touches[1].clientX;
        const dy = e.touches[0].clientY - e.touches[1].clientY;
        const dist = Math.sqrt(dx*dx + dy*dy);

        if (lastDistance) {
          if (dist > lastDistance + 5) {
            scale *= 1.05;
          } else if (dist < lastDistance - 5) {
            scale *= 0.95;
          }
          // 最小 75%
          scale = Math.max(scale, 0.75);

          // ズーム中は現在のストロークは消して座標系だけ更新
          ctx.setTransform(1,0,0,1,0,0);
          ctx.clearRect(0,0,canvas.width,canvas.height);
        }
        lastDistance = dist;
      } else if (!isZooming && e.touches.length === 1 && isDrawing) {
        const t = e.touches[0];
        const rect = canvas.getBoundingClientRect();
        points.push({ x: t.clientX - rect.left, y: t.clientY - rect.top });
        ctx.setTransform(1,0,0,1,0,0);
        ctx.clearRect(0,0,canvas.width,canvas.height);
        drawEdges();
      }
    });

    canvas.addEventListener('touchend', (e) => {
      if (e.touches.length < 2) {
        // 二本指でなくなったらズーム解除
        lastDistance = null;
        isZooming = false;
        // タッチ終了時、描画中なら履歴へ
        if (isDrawing) {
          history.push(ctx.getImageData(0,0,canvas.width,canvas.height));
        }
        isDrawing = false;
        points = [];
      }
    });

    // 初期UI
    renderPresets();
    updateActiveUI();
  </script>
</body>
</html>
