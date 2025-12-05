<!doctype html>
<html lang="pt-br">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Dashboard de Visitas — EXCELLENT CV</title>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
  <style>
    body{font-family:Inter, system-ui, Arial; background:#f5f7fb; color:#111; margin:0; padding:20px;}
    .container{max-width:1100px;margin:0 auto;}
    header{display:flex;justify-content:space-between;align-items:center;margin-bottom:18px;}
    .card{background:#fff;padding:14px;border-radius:10px;box-shadow:0 8px 30px rgba(0,0,0,0.06);}
    .grid{display:grid;grid-template-columns:repeat(3,1fr);gap:12px;margin-bottom:12px;}
    .grid .card{padding:18px;}
    table{width:100%;border-collapse:collapse;}
    th,td{padding:8px 6px;border-bottom:1px solid #eef2f7;text-align:left;font-size:14px;}
    .small{font-size:13px;color:#666}
    @media(max-width:800px){ .grid{grid-template-columns:repeat(1,1fr);} }
    .btn{display:inline-flex;align-items:center;gap:8px;padding:8px 12px;border-radius:8px;border:0;cursor:pointer;}
    .btn-primary{background:linear-gradient(135deg,#1a237e,#283593);color:white;}
    .quality-pill{display:inline-flex;align-items:center;gap:8px;padding:6px 10px;border-radius:999px;background:#eef6ff;border:1px solid #dbeeff;font-weight:700}
  </style>
</head>
<body>
  <div class="container">
    <header>
      <div>
        <h2>Dashboard de Visitas</h2>
        <div class="small">Visualize visitas ao site (dados em visitors.json)</div>
      </div>
      <div>
        <button id="refreshBtn" class="btn btn-primary"><i class="fas fa-sync"></i> Atualizar</button>
      </div>
    </header>

    <div id="authBox" class="card" style="margin-bottom:12px;">
      <form id="authForm" style="display:flex;gap:8px;flex-wrap:wrap;align-items:center;">
        <label class="small">Usuário</label>
        <input id="user" type="text" value="admin" />
        <label class="small">Senha</label>
        <input id="pass" type="password" value="admin" />
        <button id="fetchBtn" type="submit" class="btn btn-primary">Conectar</button>
        <div id="authMsg" class="small" style="margin-left:8px;color:#666"></div>
      </form>
    </div>

    <div id="stats" class="grid" style="display:none;">
      <div class="card">
        <div class="small">Visitas totais</div>
        <h3 id="total">—</h3>
      </div>
      <div class="card">
        <div class="small">IPs únicos</div>
        <h3 id="unique">—</h3>
      </div>
      <div class="card">
        <div class="small">Última atualização</div>
        <h3 id="updated">—</h3>
      </div>
    </div>

    <div id="qualityCard" style="display:none; margin-bottom:12px;">
      <div class="card">
        <div style="display:flex; justify-content:space-between; align-items:center;">
          <div>
            <div class="small">Qualidade do Site (calculada)</div>
            <div id="siteQualityDisplay" style="margin-top:6px; font-weight:800;">—</div>
            <div id="siteQualityDetails" class="small" style="margin-top:6px;color:#666;"></div>
          </div>
          <div>
            <button id="refreshQuality" class="btn btn-primary"><i class="fas fa-sync"></i> Recalcular</button>
          </div>
        </div>
      </div>
    </div>

    <div id="charts" style="display:none;">
      <div class="card" style="margin-bottom:12px;">
        <h4 style="margin:0 0 10px 0;">Visitas por caminho</h4>
        <div id="byPath"></div>
      </div>

      <div class="card" style="margin-bottom:12px;">
        <h4 style="margin:0 0 10px 0;">Visitas por dia</h4>
        <div id="byDay"></div>
      </div>

      <div class="card">
        <h4 style="margin:0 0 10px 0;">Entradas recentes (últimas 200)</h4>
        <div style="max-height:360px; overflow:auto;">
          <table id="recentTable">
            <thead><tr><th>Ts</th><th>Path</th><th>Referrer</th><th>IP</th><th class="small">UA</th></tr></thead>
            <tbody></tbody>
          </table>
        </div>
      </div>
    </div>

    <div class="small" style="margin-top:10px;color:#666;">
      Nota: este dashboard usa basic auth para chamar o endpoint /api/visitors — por favor altere as credenciais por defeito em production.
    </div>
  </div>

  <script>
    const authForm = document.getElementById('authForm');
    const userInput = document.getElementById('user');
    const passInput = document.getElementById('pass');
    const authMsg = document.getElementById('authMsg');
    const stats = document.getElementById('stats');
    const charts = document.getElementById('charts');
    const totalEl = document.getElementById('total');
    const uniqueEl = document.getElementById('unique');
    const updatedEl = document.getElementById('updated');
    const byPathEl = document.getElementById('byPath');
    const byDayEl = document.getElementById('byDay');
    const recentBody = document.querySelector('#recentTable tbody');
    const refreshBtn = document.getElementById('refreshBtn');
    const qualityCard = document.getElementById('qualityCard');
    const siteQualityDisplay = document.getElementById('siteQualityDisplay');
    const siteQualityDetails = document.getElementById('siteQualityDetails');
    const refreshQuality = document.getElementById('refreshQuality');

    let authHeader = null;

    authForm.addEventListener('submit', async (e) => {
      e.preventDefault();
      authMsg.textContent = 'A carregar…';
      authHeader = 'Basic ' + btoa(userInput.value + ':' + passInput.value);
      await fetchStats();
    });

    refreshBtn.addEventListener('click', async () => {
      if (!authHeader) return alert('Conecte com credenciais primeiro.');
      await fetchStats();
    });

    refreshQuality && refreshQuality.addEventListener('click', async () => {
      if (!authHeader) return alert('Conecte com credenciais primeiro.');
      await fetchSiteQuality(true);
    });

    async function fetchStats() {
      try {
        const resp = await fetch('/api/visitors', { headers: { Authorization: authHeader } });
        if (resp.status === 401) {
          authMsg.textContent = 'Credenciais inválidas.';
          stats.style.display = 'none';
          charts.style.display = 'none';
          qualityCard.style.display = 'none';
          return;
        }
        const data = await resp.json();
        authMsg.textContent = 'Conectado';
        stats.style.display = 'grid';
        charts.style.display = 'block';
        qualityCard.style.display = 'block';

        totalEl.textContent = data.total || 0;
        uniqueEl.textContent = data.uniqueIPs || 0;
        updatedEl.textContent = new Date().toLocaleString();

        // byPath
        byPathEl.innerHTML = '';
        const byPath = data.byPath || {};
        Object.keys(byPath).sort((a,b)=>byPath[b]-byPath[a]).forEach(p=>{
          const el = document.createElement('div');
          el.style.display='flex';
          el.style.justifyContent='space-between';
          el.style.padding='6px 0';
          el.innerHTML = `<div class="small">${p}</div><div style="font-weight:700">${byPath[p]}</div>`;
          byPathEl.appendChild(el);
        });

        // byDay
        byDayEl.innerHTML = '';
        const byDay = data.byDay || {};
        Object.keys(byDay).sort().forEach(d=>{
          const el = document.createElement('div');
          el.style.display='flex';
          el.style.justifyContent='space-between';
          el.style.padding='6px 0';
          el.innerHTML = `<div class="small">${d}</div><div style="font-weight:700">${byDay[d]}</div>`;
          byDayEl.appendChild(el);
        });

        // latest
        recentBody.innerHTML = '';
        (data.latest || []).forEach(row=>{
          const tr = document.createElement('tr');
          tr.innerHTML = `<td class="small">${new Date(row.ts).toLocaleString()}</td><td>${row.path}</td><td class="small">${row.referrer || '-'}</td><td class="small">${row.ip || '-'}</td><td class="small">${row.ua ? (row.ua.length>60? row.ua.slice(0,60)+'…' : row.ua) : ''}</td>`;
          recentBody.appendChild(tr);
        });

        // also fetch site quality (admin-protected)
        await fetchSiteQuality(false);

      } catch(err) {
        console.error(err);
        authMsg.textContent = 'Erro ao buscar dados.';
      }
    }

    async function fetchSiteQuality(force=false) {
      try {
        const resp = await fetch('/api/site-quality', { headers: { Authorization: authHeader } });
        if (resp.status === 401) {
          siteQualityDisplay.textContent = '—';
          siteQualityDetails.textContent = 'Sem acesso';
          return;
        }
        const data = await resp.json();
        if (data && typeof data.score !== 'undefined') {
          siteQualityDisplay.innerHTML = `<span class="quality-pill">${data.score} / 100</span>`;
          siteQualityDetails.textContent = `Últimos 7 dias: ${data.details.recent7} visitas — IPs únicos: ${data.details.uniqueIPs} — avg/dia: ${data.details.avgLast7}`;
        } else {
          siteQualityDisplay.textContent = '—';
          siteQualityDetails.textContent = 'Sem dados';
        }
      } catch (err) {
        console.error('Erro ao buscar site-quality', err);
        siteQualityDisplay.textContent = 'Erro';
      }
    }
  </script>
</body>
</html>
