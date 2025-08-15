<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>3×3 海龜華容道</title>
<style>
  :root { --gap: 3px; }
  body{font-family: system-ui, -apple-system, "Segoe UI", Arial; margin:0; background:#0e2a3b; color:#fff}
  .wrap{max-width:780px;margin:0 auto;padding:16px}
  h1{font-size:20px;margin:0 0 6px}
  .subtitle{opacity:.9;margin:6px 0 10px}
  .toolbar{display:flex;gap:8px;flex-wrap:wrap;align-items:center;margin-bottom:8px}
  select,button{background:#1f6f8b;color:#fff;border:none;padding:10px 14px;border-radius:10px;cursor:pointer}
  select{appearance:none}
  button:hover{filter:brightness(1.1)}
  .stats{margin-left:auto;display:flex;gap:12px;align-items:center;font-variant-numeric:tabular-nums}
  .board{width:min(92vw,600px);aspect-ratio:1/1;position:relative;border-radius:14px;background:#102534;
         box-shadow:0 12px 36px rgba(0,0,0,.4);margin:10px auto;touch-action:manipulation;user-select:none;overflow:hidden}
  .tile{position:absolute;width:calc((100% - 2*var(--gap))/3);height:calc((100% - 2*var(--gap))/3);
        background-size:300% 300%;background-repeat:no-repeat;background-position:center;
        border-radius:12px;box-shadow:inset 0 0 0 1px rgba(255,255,255,.06),0 6px 14px rgba(0,0,0,.35);
        transition:transform .12s ease;will-change:transform;margin:calc(var(--gap)/3)}
  .tile::after{content:"";position:absolute;inset:0;border-radius:12px;box-shadow:inset 0 0 0 1px rgba(0,0,0,.25)}
  .tile.empty{opacity:0;pointer-events:none}
  .gridlines{position:absolute;inset:0;border-radius:14px;pointer-events:none;
    background:linear-gradient(#ffffff26,#ffffff26) 0 33.333%/100% 1px no-repeat,
               linear-gradient(#ffffff26,#ffffff26) 0 66.666%/100% 1px no-repeat,
               linear-gradient(90deg,#ffffff26,#ffffff26) 33.333% 0/1px 100% no-repeat,
               linear-gradient(90deg,#ffffff26,#ffffff26) 66.666% 0/1px 100% no-repeat}
  .win{position:absolute;inset:0;display:none;place-items:center;background:rgba(0,0,0,.25);backdrop-filter:blur(2px);
  z-index:10; /* 加這行確保解說卡在最上層 */}
  .win.show{display:grid}
  .win .card{background:#fff;color:#123;padding:18px 22px;border-radius:12px;max-width:min(90vw,520px)}
  .win .card img{max-width:100%;border-radius:8px;margin-bottom:10px}
  .win .card h2{margin:.2em 0 .4em;font-size:18px}
  .win .card p{margin:.3em 0 1em;line-height:1.5}
   .small{font-size:12px;opacity:.8}

/* 讓解說卡在棋盤內完整顯示，字不被裁掉 */
.win{ padding:12px; }
.win .card{
  max-height: calc(100% - 24px);
  overflow: auto;
  -webkit-overflow-scrolling: touch;
}
.win .card img{
  width:100%;
  height:auto;
  display:block;
}
</style>
</head>
<body>
<div class="wrap">
  <h1>3×3 海龜華容道</h1>

  <div class="subtitle">
    選擇海龜種類：
    <select id="turtleSelect" aria-label="選擇海龜種類">
      <option value="leatherback">革龜</option>
      <option value="green">綠蠵龜</option>
      <option value="olive">欖蠵龜</option>
      <option value="hawksbill">玳瑁</option>
      <option value="loggerhead">赤蠵龜</option>
    </select>
  </div>

  <div class="subtitle">點擊或觸控相鄰空格的拼圖塊，使圖片還原。</div>

  <div class="toolbar">
    <button id="shuffle">重新洗牌</button>
    <button id="reset">還原</button>
    <div class="stats"><span>步數：<strong id="moves">0</strong></span><span>時間：<strong id="time">00:00</strong></span></div>
  </div>

  <div class="board" id="board" aria-label="Sliding puzzle board" aria-live="polite">
    <div class="gridlines" aria-hidden="true"></div>
    <div class="win" id="win"><div class="card"><strong>完成！</strong></div></div>
  </div>
</div>

<script>
(function(){
  const N = 3;
  const total = N * N;
  const emptyGoal = total - 1;

  // 物種資料（圖片、解說）
  const turtleData = {
    leatherback: {
      img: 'leatherback.jpg',
      title: '革龜（Dermochelys coriacea）',
      desc: '世界上最大的海龜，體長常見 1.5–2.0 公尺以上、體重可逾 500–900 公斤。背甲無硬鱗，為革質外皮覆蓋骨板而得名。台灣曾於東部外海、澎湖、東沙海域有紀錄。',
      infoImg: 'leatherback_info.jpg'
    },
    green: {
      img: 'green.jpg',
      title: '綠蠵龜（Chelonia mydas）',
      desc: '成龜以海草、海藻為主食，體色偏綠。台灣常見於蘭嶼、綠島與墾丁海域，亦有上岸產卵紀錄點。',
      infoImg: 'green_info.jpg'
    },
    olive: {
      img: 'olive.jpg',
      title: '欖蠵龜（Lepidochelys olivacea）',
      desc: '體型較小，背甲橄欖綠色，常見於熱帶及亞熱帶沿海。台灣海域偶有出現紀錄。',
      infoImg: 'olive_info.jpg'
    },
    hawksbill: {
      img: 'hawksbill.jpg',
      title: '玳瑁（Eretmochelys imbricata）',
      desc: '背甲鱗片重疊、龜甲邊緣具鋸齒，常活動於珊瑚礁。台灣南部及離島珊瑚礁區較易見。',
      infoImg: 'hawksbill_info.jpg'
    },
    loggerhead: {
      img: 'loggerhead.jpg',
      title: '赤蠵龜（Caretta caretta）',
      desc: '頭部寬大、背甲紅褐色，偏好硬殼無脊椎動物。台灣周邊海域有洄游與偶見紀錄。',
      infoImg: 'loggerhead_info.jpg'
    }
  };

  let currentTurtle = 'leatherback';
  const board  = document.getElementById('board');
  const movesEl = document.getElementById('moves');
  const timeEl  = document.getElementById('time');
  const winEl   = document.getElementById('win');
  const selectEl= document.getElementById('turtleSelect');

  let tiles = [];
  let moves = 0, timer = null, startTime = null;

function setBoardAspect(src){
  const img = new Image();
  img.onload = () => {
    board.style.aspectRatio = `${img.naturalWidth} / ${img.naturalHeight}`;
    draw(); // 尺寸改了，重畫
  };
  img.src = src;
}

  const p2xy = p => ({ x: p % N, y: Math.floor(p / N) });
  const xy2p = (x,y) => y * N + x;

  // 預載圖片避免切換時閃爍
  function preload(src){ const i = new Image(); i.src = src; }

  function draw(){
  // 清空舊的
  board.querySelectorAll('.tile').forEach(t => t.remove());

  // 依目前棋盤實際寬高計算每格位移（避免拉伸變形）
  const cellX = board.clientWidth  / N;
  const cellY = board.clientHeight / N;

  tiles.forEach((val, idx) => {
    const {x, y} = p2xy(idx);
    const tile = document.createElement('div');
    tile.className = 'tile' + (val === emptyGoal ? ' empty' : '');

    // 位置
    tile.style.transform = `translate(${x * cellX}px, ${y * cellY}px)`;

    // 圖片與該塊對應區域
    tile.style.backgroundImage = `url('${turtleData[currentTurtle].img}')`;
    const bx = (100 / (N - 1)) * p2xy(val).x;
    const by = (100 / (N - 1)) * p2xy(val).y;
    tile.style.backgroundPosition = `${bx}% ${by}%`;

    tile.addEventListener('click', () => move(idx));
    tile.addEventListener('touchstart', (e) => { e.preventDefault(); move(idx); }, { passive:false });
    board.appendChild(tile);
  });
}

  // 由完成狀態隨機合法步驟打亂，確保可解
  function shuffle(){
    tiles = Array.from({length: total}, (_,i) => i);
    for(let i=0; i<240; i++){
      const e = tiles.indexOf(emptyGoal);
      const {x,y} = p2xy(e);
      const ops = [];
      if(x>0)     ops.push(xy2p(x-1,y));
      if(x<N-1)   ops.push(xy2p(x+1,y));
      if(y>0)     ops.push(xy2p(x,y-1));
      if(y<N-1)   ops.push(xy2p(x,y+1));
      const s = ops[Math.floor(Math.random()*ops.length)];
      [tiles[e], tiles[s]] = [tiles[s], tiles[e]];
    }
    moves = 0; movesEl.textContent = moves;
    startTimer(); winEl.classList.remove('show');
    draw();
  }

  function move(idx){
    const e = tiles.indexOf(emptyGoal);
    const a = p2xy(idx), b = p2xy(e);
    if(Math.abs(a.x - b.x) + Math.abs(a.y - b.y) === 1){
      [tiles[e], tiles[idx]] = [tiles[idx], tiles[e]];
      moves++; movesEl.textContent = moves;
      draw();
      if(tiles.every((v,i)=>v===i)){
        stopTimer();
        showWinCard();
      }
    }
  }

  function startTimer(){
    clearInterval(timer); startTime = Date.now(); timeEl.textContent = '00:00';
    timer = setInterval(()=>{
      const s = Math.floor((Date.now() - startTime)/1000);
      const m = Math.floor(s/60);
      timeEl.textContent = `${String(m).padStart(2,'0')}:${String(s%60).padStart(2,'0')}`;
    }, 1000);
  }
  function stopTimer(){ clearInterval(timer); timer = null; }

  function showWinCard(){
    const data = turtleData[currentTurtle];
    winEl.innerHTML = `
      <div class="card" role="dialog" aria-label="完成解說卡">
        <img src="${data.infoImg}" alt="${data.title}" onerror="this.src='${data.img}'">
        <h2>${data.title}</h2>
        <p style="text-align:left">${data.desc}</p>
        <p class="small">完成步數：${movesEl.textContent}｜用時：${timeEl.textContent}</p>
<p class="small" style="margin-top:6px">
資料來源：海洋保育署、IUCN（國際自然保護聯盟）、NOAA Photo Library 等。<br>
※ 本遊戲內容僅供教育推廣參考，物種資訊可能隨研究更新。
</p>
        <button onclick="document.getElementById('win').classList.remove('show')">關閉</button>
      </div>
    `;
    winEl.classList.add('show');
  }

  // 事件
  document.getElementById('shuffle').addEventListener('click', shuffle);
  document.getElementById('reset').addEventListener('click', ()=>{
    tiles = Array.from({length: total}, (_,i)=>i);
    moves = 0; movesEl.textContent = moves; stopTimer(); draw();
  });
  selectEl.addEventListener('change', (e)=>{
  currentTurtle = e.target.value;
  preload(turtleData[currentTurtle].img);
  preload(turtleData[currentTurtle].infoImg);
  setBoardAspect(turtleData[currentTurtle].img);
  shuffle();
});

  // 初始化
  Object.values(turtleData).forEach(t => { preload(t.img); preload(t.infoImg); });
setBoardAspect(turtleData[currentTurtle].img);
shuffle();
  window.addEventListener('resize', draw);
})();
</script>
</body>
</html>
