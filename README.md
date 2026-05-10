import http from "node:http";
import fs from "node:fs";
import path from "node:path";
import { fileURLToPath } from "node:url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const PORT = process.env.PORT || 3000;
const GEMINI_API_KEY = process.env.GEMINI_API_KEY || "";
const API_FOOTBALL_KEY = process.env.API_FOOTBALL_KEY || "";
const API_SECRET = process.env.API_SECRET || "";
const ALLOWED_ORIGINS_RAW = (process.env.ALLOWED_ORIGINS || "*").split(",").map(o => o.trim());
const ALLOWED_ORIGINS = ALLOWED_ORIGINS_RAW.map(o => {
  if (o === "*") return "*";
  if (o.startsWith("*.")) return new RegExp("^https://.+\\." + o.slice(2).replace(/\./g, "\\.") + "$");
  return o;
});

const SYSTEM_PROMPT = `Você é BetAnalyst Pro, um agente especialista em análise estatística de apostas de futebol.
Sua missão é gerar palpites com confiança mínima de 85%, baseados em dados reais.
Siga estas regras:
1. Nunca emita palpite com confiança abaixo de 85%
2. Sempre calcule o Valor Esperado (EV) — só recomende se EV positivo
3. Analise: forma recente, H2H, lesões, motivação, mando de campo, odds
4. Para cada palpite informe: mercado, odds recomendada, confiança %, EV, raciocínio e riscos
5. Responda sempre em português brasileiro
6. Se dados forem insuficientes, peça mais informações

Formato obrigatório de resposta:
═══════════════════════════════
🏆 [COMPETIÇÃO] | [FASE]
⚽ [TIME A] vs [TIME B]
═══════════════════════════════
📊 ANÁLISE RÁPIDA
- Forma [Time A]: ...
- Forma [Time B]: ...
- H2H: ...
- Lesões: ...

🎯 PALPITE PRINCIPAL
Mercado: ...
Odds recomendadas: X.XX ou superior
Probabilidade real estimada: X%
EV: +X.XX ✅
Confiança: X%

💡 RACIOCÍNIO
...

⚠️ RISCOS
1. ...
2. ...

🎯 PALPITES SECUNDÁRIOS
1. ...
2. ...

💰 Stake: X% da banca
═══════════════════════════════`;

const rateLimitMap = new Map();
function checkRateLimit(ip) {
  const now = Date.now();
  const entry = rateLimitMap.get(ip) || { count: 0, start: now };
  if (now - entry.start > 15 * 60 * 1000) { rateLimitMap.set(ip, { count: 1, start: now }); return true; }
  if (entry.count >= 60) return false;
  entry.count++;
  rateLimitMap.set(ip, entry);
  return true;
}

async function callGemini(userMessage) {
  if (!GEMINI_API_KEY) throw new Error("GEMINI_API_KEY não configurada.");
  const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${GEMINI_API_KEY}`;
  const res = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      system_instruction: { parts: [{ text: SYSTEM_PROMPT }] },
      contents: [{ role: "user", parts: [{ text: userMessage }] }],
      generationConfig: { temperature: 0.3, maxOutputTokens: 4096 }
    })
  });
  if (!res.ok) { const e = await res.text(); throw new Error(`Gemini error ${res.status}: ${e}`); }
  const data = await res.json();
  return data.candidates?.[0]?.content?.parts?.[0]?.text || "Sem resposta do modelo.";
}

async function getFixtures() {
  if (!API_FOOTBALL_KEY) throw new Error("API_FOOTBALL_KEY não configurada.");
  const today = new Date().toISOString().split("T")[0];
  const res = await fetch(`https://v3.football.api-sports.io/fixtures?date=${today}&timezone=America/Sao_Paulo`, {
    headers: { "x-apisports-key": API_FOOTBALL_KEY }
  });
  if (!res.ok) throw new Error(`API-Football error ${res.status}`);
  const data = await res.json();
  return data.response || [];
}

async function getTeamStats(teamId, leagueId, season) {
  if (!API_FOOTBALL_KEY) return null;
  const res = await fetch(`https://v3.football.api-sports.io/teams/statistics?team=${teamId}&league=${leagueId}&season=${season}`, {
    headers: { "x-apisports-key": API_FOOTBALL_KEY }
  });
  if (!res.ok) return null;
  const data = await res.json();
  return data.response || null;
}

async function getH2H(h2h) {
  if (!API_FOOTBALL_KEY) return [];
  const res = await fetch(`https://v3.football.api-sports.io/fixtures?h2h=${h2h}&last=5`, {
    headers: { "x-apisports-key": API_FOOTBALL_KEY }
  });
  if (!res.ok) return [];
  const data = await res.json();
  return data.response || [];
}

function readBody(req) {
  return new Promise((resolve, reject) => {
    let data = "";
    req.on("data", c => { data += c; });
    req.on("end", () => { try { resolve(JSON.parse(data || "{}")); } catch { reject(new Error("JSON inválido.")); } });
    req.on("error", reject);
  });
}

function getAllowedOrigin(req) {
  const origin = req.headers.origin || "";
  for (const allowed of ALLOWED_ORIGINS) {
    if (allowed === "*") return "*";
    if (typeof allowed === "string" && allowed === origin) return origin;
    if (allowed instanceof RegExp && allowed.test(origin)) return origin;
  }
  const first = ALLOWED_ORIGINS.find(o => typeof o === "string" && o !== "*");
  return first || "null";
}

function sendJson(res, status, obj, origin) {
  res.writeHead(status, {
    "Content-Type": "application/json",
    "Access-Control-Allow-Origin": origin || "*",
    "Access-Control-Allow-Headers": "Content-Type, Authorization",
    "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
  });
  res.end(JSON.stringify(obj, null, 2));
}

function checkAuth(req) {
  if (!API_SECRET) return true;
  return req.headers.authorization?.replace("Bearer ", "") === API_SECRET;
}

const server = http.createServer(async (req, res) => {
  const origin = getAllowedOrigin(req);
  const ip = req.socket.remoteAddress || "unknown";
  const url = new URL(req.url, `http://localhost:${PORT}`);

  if (req.method === "OPTIONS") {
    res.writeHead(204, {
      "Access-Control-Allow-Origin": origin,
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
      "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
    });
    return res.end();
  }

  if (req.method === "GET" && url.pathname === "/health") {
    return sendJson(res, 200, { status: "ok", agent: "BetAnalyst Pro", version: "3.0", timestamp: new Date().toISOString() }, origin);
  }

  if (!checkRateLimit(ip)) return sendJson(res, 429, { error: "Muitas requisições. Tente em 15 minutos." }, origin);
  if (!checkAuth(req)) return sendJson(res, 401, { error: "Não autorizado." }, origin);

  // POST /tips — chat livre
  if (req.method === "POST" && url.pathname === "/tips") {
    let payload;
    try { payload = await readBody(req); } catch (e) { return sendJson(res, 400, { error: e.message }, origin); }
    if (!payload.message) return sendJson(res, 400, { error: "'message' é obrigatório." }, origin);
    try {
      const tip = await callGemini(payload.message);
      return sendJson(res, 200, { success: true, tip }, origin);
    } catch (e) { return sendJson(res, 500, { error: e.message }, origin); }
  }

  // POST /analyze — análise estruturada
  if (req.method === "POST" && url.pathname === "/analyze") {
    let payload;
    try { payload = await readBody(req); } catch (e) { return sendJson(res, 400, { error: e.message }, origin); }
    if (!payload.message && !payload.match) return sendJson(res, 400, { error: "Forneça 'message' ou 'match'." }, origin);
    try {
      const msg = payload.message || `Analise: ${payload.match.home_team} vs ${payload.match.away_team} — ${payload.match.competition || ""}`;
      const analysis = await callGemini(msg);
      return sendJson(res, 200, { success: true, analysis }, origin);
    } catch (e) { return sendJson(res, 500, { error: e.message }, origin); }
  }

  // GET /fixtures — jogos do dia com análise automática
  if (req.method === "GET" && url.pathname === "/fixtures") {
    try {
      const fixtures = await getFixtures();
      if (!fixtures.length) return sendJson(res, 200, { success: true, fixtures: [], message: "Nenhum jogo encontrado para hoje." }, origin);

      const top = fixtures.slice(0, 5);
      const results = await Promise.all(top.map(async (f) => {
        const home = f.teams.home;
        const away = f.teams.away;
        const league = f.league;
        const h2hKey = `${home.id}-${away.id}`;
        const [statsHome, statsAway, h2h] = await Promise.all([
          getTeamStats(home.id, league.id, league.season),
          getTeamStats(away.id, league.id, league.season),
          getH2H(h2hKey)
        ]);

        const prompt = `
Analise este jogo e gere palpites com confiança ≥ 85%:

PARTIDA: ${home.name} vs ${away.name}
COMPETIÇÃO: ${league.name} (${league.country})
DATA: ${f.fixture.date}

ESTATÍSTICAS ${home.name}:
${statsHome ? `- Jogos: ${statsHome.fixtures?.played?.total || "N/A"} | Vitórias: ${statsHome.fixtures?.wins?.total || "N/A"} | Gols pró: ${statsHome.goals?.for?.total?.total || "N/A"} | Gols contra: ${statsHome.goals?.against?.total?.total || "N/A"}` : "Não disponível"}

ESTATÍSTICAS ${away.name}:
${statsAway ? `- Jogos: ${statsAway.fixtures?.played?.total || "N/A"} | Vitórias: ${statsAway.fixtures?.wins?.total || "N/A"} | Gols pró: ${statsAway.goals?.for?.total?.total || "N/A"} | Gols contra: ${statsAway.goals?.against?.total?.total || "N/A"}` : "Não disponível"}

H2H (últimos 5 jogos):
${h2h.length ? h2h.map(g => `${g.teams.home.name} ${g.goals.home}-${g.goals.away} ${g.teams.away.name}`).join("\n") : "Sem histórico disponível"}
        `.trim();

        const analysis = await callGemini(prompt);
        return {
          fixture_id: f.fixture.id,
          home: home.name,
          away: away.name,
          league: league.name,
          date: f.fixture.date,
          analysis
        };
      }));

      return sendJson(res, 200, { success: true, date: new Date().toISOString().split("T")[0], fixtures: results }, origin);
    } catch (e) { return sendJson(res, 500, { error: e.message }, origin); }
  }

  sendJson(res, 404, { error: `Rota não encontrada: ${req.method} ${url.pathname}` }, origin);
});

server.listen(PORT, () => {
  console.log(`✅ BetAnalyst Pro v3.0 rodando na porta ${PORT}`);
  console.log(`   Gemini: ${GEMINI_API_KEY ? "✅" : "❌ não configurado"}`);
  console.log(`   API-Football: ${API_FOOTBALL_KEY ? "✅" : "❌ não configurado"}`);
});
