index.html
<!doctype html>
<html lang="ro">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Fațadă – editor simplu (A)</title>
  <style>
    :root { --bg:#0b0c10; --panel:#11131a; --ink:#e9eef7; --muted:#aab4c3; --line:#222738; }
    * { box-sizing: border-box; }
    body { margin:0; font-family:-apple-system,system-ui,Segoe UI,Roboto,Arial; background:var(--bg); color:var(--ink); }
    .bar{ position:sticky; top:0; z-index:5; display:flex; gap:12px; align-items:center; justify-content:space-between;
      padding:12px 14px; background:#0f1118; border-bottom:1px solid var(--line); }
    .title{ font-weight:800; }
    .actions{ display:flex; gap:8px; flex-wrap:wrap; justify-content:flex-end; }
    .btn{ border:1px solid var(--line); background:#171a24; color:var(--ink); padding:10px 12px; border-radius:10px; font-weight:700; }
    .btn:active{ transform: translateY(1px); }
    .btn-danger{ background:#2a161a; border-color:#4a232b; }
    .btn input{ display:none; }

    .layout{ display:grid; grid-template-columns: 340px 1fr; gap:12px; padding:12px; }
    @media (max-width: 900px){ .layout{ grid-template-columns: 1fr; } }

    .panel{ background:var(--panel); border:1px solid var(--line); border-radius:14px; padding:12px; }
    .panel h2{ margin:6px 0 10px; font-size:16px; }
    .panel hr{ border:none; border-top:1px solid var(--line); margin:12px 0; }
    .grid{ display:grid; grid-template-columns: 1fr 1fr; gap:10px; }
    .card{ border:1px solid var(--line); background:#151826; color:var(--ink); border-radius:14px; padding:10px; text-align:left; }
    .thumb{ height:56px; border-radius:10px; border:1px dashed #38405a; background:rgba(255,255,255,0.03); position:relative; }
    .name{ margin-top:8px; font-weight:800; font-size:14px; }

    .t-anc::after{ content:""; position:absolute; inset:10px; border:3px solid #cfd6e5; border-radius:8px; }
    .t-brau::after{ content:""; position:absolute; left:8px; right:8px; top:24px; height:10px; background:#cfd6e5; border-radius:6px; }
    .t-cor::after{ content:""; position:absolute; left:8px; right:8px; top:18px; height:16px; background:#cfd6e5; border-radius:6px; box-shadow: 0 10px 0 rgba(207,214,229,0.5); }
    .t-col::after{ content:""; position:absolute; left:22px; right:22px; top:10px; bottom:10px; border:3px solid #cfd6e5; border-radius:30px; }

    .row{ display:flex; gap:8px; flex-wrap:wrap; }
    .help{ margin:10px 2px 0; color:var(--muted); line-height:1.35; font-size:13px; }

    .stage{ background:var(--panel); border:1px solid var(--line); border-radius:14px; padding:12px; position:relative; min-height: 60vh; }
    canvas{ width:100%; height:auto; display:block; border-radius:12px; background:#0c0f18; border:1px solid #1f2435; touch-action:none; }
    .hint{ position:absolute; inset:12px; display:flex; align-items:center; justify-content:center; color:var(--muted); pointer-events:none; font-weight:800; }
    .small{ font-size:12px; color:var(--muted); margin-top:8px; }
  </style>
</head>
<body>
  <header class="bar">
    <div class="title">Fațadă – editor simplu (A)</div>
    <div class="actions">
      <label class="btn">
        Încarcă poză
        <input id="file" type="file" accept="image/*" />
      </label>
      <button id="exportBtn" class="btn">Salvează / Share</button>
      <button id="clearBtn" class="btn btn-danger">Șterge tot</button>
    </div>
  </header>

  <main class="layout">
    <section class="panel">
      <h2>Profile preset</h2>
      <div class="grid">
        <button class="card" data-add="ancadrament"><div class="thumb t-anc"></div><div class="name">Ancadrament</div></button>
        <button class="card" data-add="brau"><div class="thumb t-brau"></div><div class="name">Brau</div></button>
        <button class="card" data-add="cornisa"><div class="thumb t-cor"></div><div class="name">Cornisă</div></button>
        <button class="card" data-add="coloana"><div class="thumb t-col"></div><div class="name">Coloană</div></button>
      </div>

      <hr />

      <h2>Element selectat</h2>
      <div class="row">
        <button id="scaleDown" class="btn">- mic</button>
        <button id="scaleUp" class="btn">+ mare</button>
        <button id="deleteSel" class="btn btn-danger">Șterge</button>
      </div>

      <p class="help">
        • Tap pe un element ca să-l selectezi.<br/>
        • Trage cu degetul ca să-l muți.<br/>
        • Folosește + / - pentru mărime.<br/>
        • „Salvează / Share” îți deschide/trimite imaginea pe iPhone.
      </p>

      <div class="small" id="status"></div>
    </section>

    <section class="stage">
      <canvas id="c"></canvas>
      <div class="hint" id="hint">Încarcă o poză ca să începi.</div>
    </section>
  </main>

<script>
  const fileEl = document.getElementById("file");
  const canvas = document.getElementById("c");
  const ctx = canvas.getContext("2d");
  const hint = document.getElementById("hint");
  const statusEl = document.getElementById("status");

  const exportBtn = document.getElementById("exportBtn");
  const clearBtn = document.getElementById("clearBtn");
  const scaleUp = document.getElementById("scaleUp");
  const scaleDown = document.getElementById("scaleDown");
  const deleteSel = document.getElementById("deleteSel");

  let bgImg = null;
  let items = [];
  let selectedId = null;

  let dragging = false;
  let dragOffset = { x:0, y:0 };

  function setStatus(t){ statusEl.textContent = t || ""; }

  function uid(){ return Math.random().toString(16).slice(2) + Date.now().toString(16); }
  function setCanvasSize(w,h){ canvas.width = w; canvas.height = h; }

  function drawRoundedRect(x,y,w,h,r){
    r = Math.min(r, w/2, h/2);
    ctx.beginPath();
    ctx.moveTo(x+r, y);
    ctx.arcTo(x+w, y, x+w, y+h, r);
    ctx.arcTo(x+w, y+h, x, y+h, r);
    ctx.arcTo(x, y+h, x, y, r);
    ctx.arcTo(x, y, x+w, y, r);
    ctx.closePath();
  }

  function pointInRect(px,py, it){
    return px >= it.x && px <= it.x + it.w && py >= it.y && py <= it.y + it.h;
  }

  function getPointerPos(e){
    const rect = canvas.getBoundingClientRect();
    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const clientY = e.touches ? e.touches[0].clientY : e.clientY;
    const x = (clientX - rect.left) * (canvas.width / rect.width);
    const y = (clientY - rect.top) * (canvas.height / rect.height);
    return { x, y };
  }

  function bringToFront(id){
    const idx = items.findIndex(i => i.id === id);
    if(idx >= 0){
      const [it] = items.splice(idx,1);
      items.push(it);
    }
  }

  function draw(){
    ctx.clearRect(0,0,canvas.width,canvas.height);

    if(bgImg){
      ctx.drawImage(bgImg, 0, 0, canvas.width, canvas.height);
      hint.style.display = "none";
    } else {
      hint.style.display = "flex";
    }

    for(const it of items){
      ctx.save();
      ctx.globalAlpha = 0.92;
      ctx.fillStyle = "rgba(255,255,255,0.12)";
      ctx.strokeStyle = "rgba(255,255,255,0.90)";
      ctx.lineWidth = Math.max(2, canvas.width * 0.003);

      if(it.type === "brau"){
        drawRoundedRect(it.x, it.y, it.w, it.h, Math.min(12, it.h/2));
        ctx.fill(); ctx.globalAlpha = 1; ctx.stroke();
      }

      if(it.type === "cornisa"){
        const h1 = it.h * 0.55;
        const h2 = it.h * 0.35;
        drawRoundedRect(it.x, it.y, it.w, h1, 10);
        ctx.fill(); ctx.globalAlpha = 1; ctx.stroke();
        ctx.globalAlpha = 0.85;
        drawRoundedRect(it.x, it.y + h1, it.w, h2, 10);
        ctx.fill(); ctx.globalAlpha = 1; ctx.stroke();
      }

      if(it.type === "ancadrament"){
        ctx.globalAlpha = 0.9;
        drawRoundedRect(it.x, it.y, it.w, it.h, 16);
        ctx.stroke();
        const pad = Math.max(10, Math.min(it.w,it.h)*0.12);
        drawRoundedRect(it.x+pad, it.y+pad, it.w-2*pad, it.h-2*pad, 12);
        ctx.stroke();
      }

      if(it.type === "coloana"){
        const baseH = it.h*0.14;
        const capH  = it.h*0.18;
        const shaftY = it.y + capH;
        const shaftH = it.h - capH - baseH;

        ctx.globalAlpha = 0.15;
        drawRoundedRect(it.x, shaftY, it.w, shaftH, it.w/2);
        ctx.fill(); ctx.globalAlpha = 1; ctx.stroke();

        ctx.globalAlpha = 0.2;
        drawRoundedRect(it.x- it.w*0.18, it.y, it.w*1.36, capH, 12);
        ctx.fill(); ctx.globalAlpha = 1; ctx.stroke();

        ctx.globalAlpha = 0.2;
        drawRoundedRect(it.x- it.w*0.10, it.y + capH + shaftH, it.w*1.20, baseH, 12);
        ctx.fill(); ctx.globalAlpha = 1; ctx.stroke();
      }

      if(it.id === selectedId){
        ctx.setLineDash([10,6]);
        ctx.strokeStyle = "rgba(0,190,255,0.95)";
        ctx.lineWidth = Math.max(3, canvas.width * 0.004);
        ctx.strokeRect(it.x-2, it.y-2, it.w+4, it.h+4);
        ctx.setLineDash([]);
      }

      ctx.restore();
    }
  }

  function addItem(type){
    if(!bgImg){ alert("Întâi încarcă o poză."); return; }
    const base = Math.min(canvas.width, canvas.height);
    let w = base*0.25, h = base*0.25;
    if(type === "brau"){ w = canvas.width*0.75; h = base*0.06; }
    if(type === "cornisa"){ w = canvas.width*0.82; h = base*0.08; }
    if(type === "ancadrament"){ w = base*0.30; h = base*0.38; }
    if(type === "coloana"){ w = base*0.16; h = base*0.52; }
    const it = { id: uid(), type, x:(canvas.width-w)/2, y:(canvas.height-h)/2, w, h };
    items.push(it);
    selectedId = it.id;
    draw();
  }

  function deleteSelected(){
    if(!selectedId) return;
    items = items.filter(i => i.id !== selectedId);
    selectedId = null;
    draw();
  }

  function scaleSelected(factor){
    if(!selectedId) return;
    const it = items.find(i => i.id === selectedId);
    if(!it) return;
    const cx = it.x + it.w/2;
    const cy = it.y + it.h/2;
    it.w *= factor; it.h *= factor;
    it.w = Math.max(20, Math.min(it.w, canvas.width*1.2));
    it.h = Math.max(20, Math.min(it.h, canvas.height*1.2));
    it.x = cx - it.w/2;
    it.y = cy - it.h/2;
    draw();
  }

  function onDown(e){
    e.preventDefault();
    const p = getPointerPos(e);
    for(let i = items.length - 1; i >= 0; i--){
      const it = items[i];
      if(pointInRect(p.x,p.y,it)){
        selectedId = it.id;
        bringToFront(it.id);
        dragging = true;
        dragOffset.x = p.x - it.x;
        dragOffset.y = p.y - it.y;
        draw();
        return;
      }
    }
    selectedId = null;
    draw();
  }

  function onMove(e){
    if(!dragging || !selectedId) return;
    e.preventDefault();
    const p = getPointerPos(e);
    const it = items.find(i => i.id === selectedId);
    if(!it) return;
    it.x = p.x - dragOffset.x;
    it.y = p.y - dragOffset.y;
    draw();
  }

  function onUp(){ dragging = false; }

  async function exportImage(){
    if(!bgImg){ alert("Nu ai încă poză."); return; }

    setStatus("Export…");

    // toBlob merge mai bine decât dataURL pe iOS
    canvas.toBlob(async (blob) => {
      if(!blob){ setStatus("Export eșuat."); alert("Export eșuat."); return; }

      // iOS: încearcă Share Sheet (cel mai stabil)
      if(navigator.canShare && navigator.canShare({ files: [new File([blob], "fatada.png", {type:"image/png"})] })){
        const file = new File([blob], "fatada.png", { type:"image/png" });
        try{
          await navigator.share({ files:[file], title:"Fațadă", text:"Imagine exportată" });
          setStatus("Trimis prin Share.");
        }catch(e){
          setStatus("Share anulat.");
        }
        return;
      }

      // fallback: deschide imaginea într-un tab nou (de acolo Save Image)
      const url = URL.createObjectURL(blob);
      window.open(url, "_blank");
      setStatus("S-a deschis imaginea într-un tab nou (Save Image).");
      setTimeout(()=>URL.revokeObjectURL(url), 60000);
    }, "image/png");
  }

  fileEl.addEventListener("change", (e) => {
    const f = e.target.files?.[0];
    if(!f) return;

    const reader = new FileReader();
    reader.onload = () => {
      const img = new Image();
      img.onload = () => {
        bgImg = img;
        // limită OK pe mobil
        const maxSide = 1600;
        const scale = Math.min(1, maxSide / Math.max(img.width, img.height));
        setCanvasSize(Math.round(img.width*scale), Math.round(img.height*scale));
        items = [];
        selectedId = null;
        setStatus("Poză încărcată.");
        draw();
      };
      img.src = reader.result;
    };
    reader.readAsDataURL(f);
  });

  document.querySelectorAll("[data-add]").forEach(btn => {
    btn.addEventListener("click", () => addItem(btn.dataset.add));
  });

  scaleUp.addEventListener("click", () => scaleSelected(1.08));
  scaleDown.addEventListener("click", () => scaleSelected(0.92));
  deleteSel.addEventListener("click", deleteSelected);

  clearBtn.addEventListener("click", () => {
    items = [];
    selectedId = null;
    bgImg = null;
    setCanvasSize(900, 600);
    setStatus("Reset.");
    draw();
  });

  exportBtn.addEventListener("click", exportImage);

  canvas.addEventListener("mousedown", onDown);
  canvas.addEventListener("mousemove", onMove);
  window.addEventListener("mouseup", onUp);
  canvas.addEventListener("touchstart", onDown, { passive:false });
  canvas.addEventListener("touchmove", onMove, { passive:false });
  window.addEventListener("touchend", onUp);

  setCanvasSize(900, 600);
  draw();
</script>
</body>
</html>. 
