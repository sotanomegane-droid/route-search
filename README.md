# route-search
駅から駅、バス停かバス停、そこを案内する旅行案内アプリです。
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>ルート検索（東京 → 高尾）</title>
  <style>
    :root { color-scheme: light; }
    body { font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans JP", sans-serif; margin: 0; background:#f6f7fb; }
    header { padding: 18px 16px; background: white; border-bottom: 1px solid #e6e8ef; position: sticky; top:0; z-index: 10; }
    header h1 { margin: 0; font-size: 18px; }
    main { max-width: 980px; margin: 0 auto; padding: 16px; }
    .card { background: white; border: 1px solid #e6e8ef; border-radius: 14px; padding: 14px; box-shadow: 0 4px 14px rgba(0,0,0,.05); }
    .grid { display: grid; gap: 12px; }
    @media (min-width: 760px){ .grid2 { grid-template-columns: 1fr 1fr; } }
    label { font-size: 12px; color:#5b6172; display:block; margin-bottom: 6px; }
    input, select, button { width: 100%; padding: 12px; border-radius: 12px; border: 1px solid #d8dbea; font-size: 14px; background: white; }
    button { border: none; background: #1a73e8; color: white; font-weight: 800; cursor: pointer; }
    button:hover { filter: brightness(0.96); }
    .row { display:flex; gap:10px; align-items:center; }
    .row > * { flex: 1; }
    .muted { color:#6b7280; font-size: 12px; line-height: 1.55; }
    .warn { color:#8a1c1c; font-weight: 800; }
    .results { margin-top: 12px; }
    .result { border: 1px solid #e6e8ef; border-radius: 14px; padding: 12px; margin-top: 10px; }
    .result h3 { margin: 0 0 6px; font-size: 15px; }
    .meta { display:flex; gap: 10px; flex-wrap: wrap; font-size: 12px; color:#4b5563; }
    .pill { background:#eef3ff; color:#1a3f9b; padding: 4px 8px; border-radius: 999px; font-weight:800; }
    .steps { margin-top: 10px; padding-left: 18px; }
    .steps li { margin: 6px 0; }
    .footer { margin-top: 12px; font-size: 12px; color:#6b7280; }
    .swap { max-width: 140px; background: #0f172a; }
    .kpi { display:flex; gap:10px; flex-wrap: wrap; margin-top: 8px; }
    .kpi span { background:#f2f4f8; border:1px solid #e6e8ef; padding: 6px 10px; border-radius: 999px; font-size: 12px; color:#374151; }
    .tiny { font-size: 11px; color:#6b7280; }
  </style>
</head>

<body>
  <header>
    <h1>ルート検索（東京 → 高尾 / 試作）</h1>
  </header>

  <main class="grid" style="gap:14px;">
    <section class="card grid grid2">
      <div>
        <label>出発</label>
        <input id="from" value="東京" placeholder="例）東京" />
      </div>
      <div>
        <label>到着</label>
        <input id="to" value="高尾" placeholder="例）高尾" />
      </div>

      <div>
        <label>日時</label>
        <div class="row">
          <input id="date" type="date" />
          <input id="time" type="time" />
        </div>
      </div>

      <div>
        <label>検索条件</label>
        <select id="mode">
          <option value="fast">最短（推定）</option>
          <option value="few">乗換少なめ</option>
          <option value="cheap">最安（未対応）</option>
        </select>
      </div>

      <div class="row" style="grid-column: 1 / -1;">
        <button id="search">検索</button>
        <button id="swap" class="swap" type="button">⇄ 入替</button>
      </div>

      <p class="muted" style="grid-column: 1 / -1; margin: 0;">
        ※この版は「東京→高尾」の<strong>東京駅発時刻</strong>を使って候補を出します（運賃は後で追加できます）。
      </p>
    </section>

    <section class="card">
      <h2 style="margin:0 0 8px; font-size:16px;">検索結果</h2>
      <div id="results" class="results"></div>
      <div class="footer">
        次やるなら：高尾の着時刻を“駅停車表”で精密化 / 途中乗換（例：立川）対応 / 運賃追加。
      </div>
    </section>
  </main>

  <script>
    // =========================
    // 初期日時を「今」にセット
    // =========================
    const now = new Date();
    const pad = (n) => String(n).padStart(2, "0");
    document.getElementById("date").value =
      `${now.getFullYear()}-${pad(now.getMonth()+1)}-${pad(now.getDate())}`;
    document.getElementById("time").value = `${pad(now.getHours())}:${pad(now.getMinutes())}`;

    // =========================
    // 東京駅：中央線快速（下り）時刻（スクショの分）
    // ※平日想定。まずは weekday に全部入れて、holiday は weekday を流用します。
    // =========================
    const TIMETABLE = {
      weekday: {
        4:  [39,59],
        5:  [18,29,44,54],
        6:  [4,11,16,22,28,32,37,46,51,56],
        7:  [1,3,7,14,18,22,28,31,33,36,40,43,45,49,54,56,59],
        8:  [1,3,6,8,11,13,15,17,19,22,24,26,28,30,33,35,37,40,42,44,46,48,51,53,55,57],
        9:  [2,6,8,10,13,15,17,19,22,24,27,31,36,38,41,43,48,53,55,57],
        10: [0,5,10,15,20,25,27,31,34,38,42,49,53,56],
        11: [1,7,10,13,18,26,29,34,38,43,50,54,58],
        12: [3,5,9,16,22,25,29,34,37,43,49,55,58],
        13: [3,5,10,17,21,26,29,34,39,45,50,55,59],
        14: [4,9,14,18,24,28,32,37,42,48,52,55,59],
        15: [2,5,9,15,17,21,24,29,33,37,45,51,55],
        16: [0,4,8,15,17,20,25,29,33,36,40,45,48,50,53,56],
        17: [0,2,5,8,12,15,17,20,22,25,27,29,32,34,36,41,45,47,49,52,55,58],
        18: [1,3,6,8,12,15,17,19,22,25,27,31,33,36,41,45,47,49,52,54,57],
        19: [0,5,8,12,15,17,19,22,26,29,32,35,38,43,45,48,51,54,57],
        20: [1,4,9,15,19,22,25,28,32,36,39,45,47,50,52,56],
        21: [1,5,10,14,18,21,25,31,35,38,43,45,48,50,55],
        22: [1,6,10,14,18,21,26,31,36,39,43,45,48,52,55],
        23: [1,7,14,19,22,27,31,36,39,45,49,56],
        0:  [5],
      },
      holiday: null,
    };

    // holiday が空なら weekday を流用
    TIMETABLE.holiday = TIMETABLE.weekday;

    // =========================
    // 所要時間（ざっくり）
    // ※正確化したい場合は、時間帯別・種別別に調整できる
    // =========================
    const BASE_DURATION_MIN = 72;        // 通常想定
    const RUSH_BUFFER_MIN = 10;          // 7〜8時台は遅れやすい → 到着見込みに余裕

    // =========================
    // ユーティリティ
    // =========================
    function label(mode){
      if(mode==="fast") return "最短";
      if(mode==="cheap") return "最安";
      if(mode==="few") return "乗換少なめ";
      return mode;
    }

    function escapeHtml(str){
      return str.replace(/[&<>"']/g, (c) => ({
        "&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"
      }[c]));
    }

    function parseDateTime(dateStr, timeStr){
      const [y,m,d] = dateStr.split("-").map(Number);
      const [hh,mm] = timeStr.split(":").map(Number);
      return new Date(y, m-1, d, hh, mm, 0, 0);
    }

    function addMinutes(dt, mins){
      return new Date(dt.getTime() + mins*60000);
    }

    function fmtHM(dt){
      return `${pad(dt.getHours())}:${pad(dt.getMinutes())}`;
    }

    // 土日をholiday扱い（祝日は未対応：必要なら追加できる）
    function dayType(dt){
      const w = dt.getDay(); // 0 Sun ... 6 Sat
      return (w === 0 || w === 6) ? "holiday" : "weekday";
    }

    // 次の出発をN本取得（当日分だけ）
    function nextDepartures(dt, tableForDay, limit=5){
      const res = [];
      const h0 = dt.getHours();
      const m0 = dt.getMinutes();

      for(let h = h0; h <= 23; h++){
        const mins = tableForDay[h] || [];
        for(const m of mins){
          if(h === h0 && m < m0) continue;
          res.push(new Date(dt.getFullYear(), dt.getMonth(), dt.getDate(), h, m, 0, 0));
          if(res.length >= limit) return res;
        }
      }
      return res;
    }

    // =========================
    // 画面描画
    // =========================
    function render() {
      const from = document.getElementById("from").value.trim();
      const to = document.getElementById("to").value.trim();
      const mode = document.getElementById("mode").value;
      const date = document.getElementById("date").value;
      const time = document.getElementById("time").value;

      const box = document.getElementById("results");
      const dt = parseDateTime(date, time);

      const isTokyoTakao = (from === "東京" && to === "高尾");
      const type = dayType(dt);
      const table = TIMETABLE[type];

      // 7〜8時台の遅延注意（あなたの情報を反映）
      const h = dt.getHours();
      const isRush = (h === 7 || h === 8);
      const rushNote = isRush
        ? "⚠️ 7〜8時台は遅延しやすい時間帯：到着見込みに+10分の余裕を自動で加えています。"
        : "";

      // 東京→高尾以外は、丁寧に案内（この版は東京→高尾専用）
      if(!isTokyoTakao){
        box.innerHTML = `
          <div class="muted">
            検索：<b>${escapeHtml(from)}</b> → <b>${escapeHtml(to)}</b>（${date} ${time} / ${type === "weekday" ? "平日" : "土日"} / 条件：${label(mode)}）<br>
            ※この版は「東京→高尾」専用です。出発/到着を東京・高尾にして検索してね。
          </div>
        `;
        return;
      }

      const deps = nextDepartures(dt, table, 5);
      if(deps.length === 0){
        box.innerHTML = `
          <div class="muted">
            検索：<b>東京</b> → <b>高尾</b>（${date} ${time} / ${type === "weekday" ? "平日" : "土日"} / 条件：${label(mode)}）
          </div>
          <div class="result">
            <h3>本日の候補が見つかりません</h3>
            <div class="muted">当日分の時刻表が尽きました。時間を早めるか、翌日の検索にしてね。</div>
          </div>
        `;
        return;
      }

      // 所要時間（朝は遅れやすいのでバッファを足す）
      const duration = BASE_DURATION_MIN + (isRush ? RUSH_BUFFER_MIN : 0);

      const cards = deps.map((dep, i) => {
        const arr = addMinutes(dep, duration);
        return `
          <div class="result">
            <h3>候補 ${i+1}：中央線（快速）想定</h3>
            <div class="meta">
              <span class="pill">${escapeHtml(fmtHM(dep))} 発 → ${escapeHtml(fmtHM(arr))} 着</span>
              <span>所要：${duration}分（推定）</span>
              <span>乗換：0回</span>
              <span>徒歩：少</span>
              <span>運賃：未設定</span>
            </div>
            <div class="kpi">
              <span>出発：東京</span>
              <span>到着：高尾</span>
              <span>${type === "weekday" ? "平日" : "土日"}</span>
            </div>
            <ol class="steps">
              <li>東京（${escapeHtml(fmtHM(dep))} 発）</li>
              <li>JR 中央線（快速）</li>
              <li>高尾（${escapeHtml(fmtHM(arr))} 着：推定）</li>
            </ol>
            <div class="tiny">※東京駅の発車分から候補を作っています（着時刻は推定）。</div>
          </div>
        `;
      }).join("");

      box.innerHTML = `
        <div class="muted">
          検索：<b>東京</b> → <b>高尾</b>（${date} ${time} / ${type === "weekday" ? "平日" : "土日"} / 条件：${label(mode)}）<br>
          ${rushNote ? `<span class="warn">${escapeHtml(rushNote)}</span><br>` : ""}
        </div>
        ${cards}
      `;
    }

    document.getElementById("search").addEventListener("click", render);
    document.getElementById("swap").addEventListener("click", () => {
      const a = document.getElementById("from");
      const b = document.getElementById("to");
      [a.value, b.value] = [b.value, a.value];
      render();
    });

    render();
  </script>
</body>
</html>
