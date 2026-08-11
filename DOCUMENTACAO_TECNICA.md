# Análise e Explicação Linha por Linha do Código do Dashboard

Este documento descreve detalhadamente o funcionamento técnico das rotinas em JavaScript executadas dentro do plugin **Volkov Labs ECharts (Business Charts)** no Grafana, mapeando a estrutura dos dados do Zabbix e sua transformação em elementos visuais.

## 1. Análise da Estrutura Geral da Especificação (JSON)
O arquivo JSON segue o esquema de declaração `dashboard.grafana.app/v2` (Grafana Dashboard as Code).

* **`metadata`**:
  * `name`: Nome do recurso (`gerenciamento-incidentes-community-businesscharts`).
  * `uid`: Identificador único do dashboard (`721d3cb7-48b1-4724-89af-dba9858fdc82`).
  * `annotations`: Define os metadados de criação (`Grafana v12.4.2`).

* **`spec.elements`**:
  Contém o mapeamento de cada painel e suas respectivas consultas (*Queries*) e scripts de visualização (*VizConfig*).

## 2. Análise do Painel: Incidentes Ativos (`panel-1`)
Abaixo está a explicação linha por linha da opção de renderização em JavaScript (`getOption`) que processa os problemas ativos retornados pela API/plugin do Zabbix.

```javascript
// Linhas 1-2: Largura e altura dinâmicas do container do painel no Grafana
const W = Math.max(1, context.panel.chart.getWidth());
const H = Math.max(1, context.panel.chart.getHeight());

// Linha 3: Instância global da biblioteca Apache ECharts fornecida pelo plugin
const echarts = context.echarts;

// Linhas 5-10: Mapeamento de paleta de cores para o tema escuro (Dark Theme)
const C = {
  bg0: '#050B16', bg1: '#08111F', card: '#0D1B2E', card2: '#12243A',
  border: 'rgba(174,186,213,0.16)', text: '#F4F7FC', sub: '#AEBAD5', mute: '#7F8DA8',
  cyan: '#17C7D5', purple: '#7C3AED', green: '#14B8A6', orange: '#F59E0B',
  red: '#EF4444', blue: '#38BDF8', gray: '#64748B', yellow: '#EAB308', deepRed: '#DC2626'
};

// Linhas 12-18: Função utilitária para extrair arrays independentemente da estrutura do buffer/DataFrame
function vals(f) {
  if (!f || f.values == null) return [];
  if (Array.isArray(f.values)) return f.values;
  if (typeof f.values.toArray === 'function') return f.values.toArray();
  if (f.values.buffer && Array.isArray(f.values.buffer)) return f.values.buffer;
  try { return Array.from(f.values); } catch (e) { return []; }
}

// Linhas 20-35: Normaliza múltiplos formatos de data/timestamp para milissegundos
function normalizeTime(v) {
  if (v === null || v === undefined || v === '') return null;
  if (v instanceof Date) return v.getTime();
  if (typeof v === 'object') {
    if (v._d instanceof Date) return v._d.getTime();
    if (typeof v.valueOf === 'function') {
      const vv = v.valueOf();
      if (vv !== v && Number.isFinite(Number(vv))) {
        const n = Number(vv);
        return Math.abs(n) < 1e12 ? n * 1000 : n;
      }
    }
  }
  const n = Number(v);
  if (Number.isFinite(n)) return Math.abs(n) < 1e12 ? n * 1000 : n;
  const parsed = Date.parse(String(v));
  return Number.isFinite(parsed) ? parsed : null;
}

// Linhas 37-44: Formata timestamps no padrão brasileiro DD/MM/AAAA HH:mm
function fmtDateTime(v) {
  const ts = normalizeTime(v);
  if (ts === null) return '—';
  const d = new Date(ts);
  if (Number.isNaN(d.getTime())) return String(v ?? '—');
  const z = n => String(n).padStart(2, '0');
  return `${z(d.getDate())}/${z(d.getMonth() + 1)}/${d.getFullYear()} ${z(d.getHours())}:${z(d.getMinutes())}`;
}

// Linhas 46-56: Calcula o tempo decorrido desde a ocorrência do incidente (ex: "15 min", "2h 10m", "3d 4h")
function ageText(v) {
  const ts = normalizeTime(v);
  if (ts === null) return '—';
  let sec = Math.max(0, Math.floor((Date.now() - ts) / 1000));
  if (sec < 60) return `${sec}s`;
  const min = Math.floor(sec / 60);
  if (min < 60) return `${min} min`;
  const h = Math.floor(min / 60);
  if (h < 24) return `${h}h ${min % 60}m`;
  const days = Math.floor(h / 24);
  return `${days}d ${h % 24}h`;
}

// Linhas 58-65: Cria elemento de texto vetorial no canvas do ECharts
function txt(x, y, text, size, color, weight, align = 'left', extra = {}) {
  return {
    type: 'text', silent: true,
    style: {
      x, y, text: String(text), font: `${weight || 500} ${size}px Inter, sans-serif`,
      fill: color || C.text, textAlign: align, textVerticalAlign: 'middle', ...extra
    }
  };
}

// Linhas 67-73: Desenha o cartão de fundo com cantos arredondados e borda
function roundedCard(fill = C.card) {
  return {
    type: 'rect', silent: true,
    shape: { x: 1, y: 1, width: Math.max(0, W - 2), height: Math.max(0, H - 2), r: 14 },
    style: { fill, stroke: C.border, lineWidth: 1, shadowBlur: 18, shadowColor: 'rgba(0,0,0,.18)' }
  };
}

// Linhas 75-81: Desenha a barra lateral de destaque de cor (Accent Line)
function accentLine(color) {
  return {
    type: 'rect', silent: true,
    shape: { x: 0, y: 0, width: 4, height: H, r: [14, 0, 0, 14] },
    style: { fill: color }
  };
}

// Linhas 83-90: Sanitização contra XSS para tooltips e HTML injetado
function esc(v) {
  return String(v ?? '')
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}

// Linhas 92-99: Mapeia objetos ou estruturas aninhadas do Zabbix para texto legível
function objName(v) {
  if (v === null || v === undefined) return '';
  if (typeof v === 'string' || typeof v === 'number' || typeof v === 'boolean') return String(v);
  if (Array.isArray(v)) return v.map(objName).filter(Boolean).join(', ');
  if (typeof v === 'object') {
    return String(v.name ?? v.host ?? v.value ?? v.tag ?? v.description ?? '');
  }
  return String(v);
}

// Linhas 101-113: Busca a série de dados "problems" retornada pela consulta Zabbix
function problemsFromFrames() {
  const frames = (context.panel.data && context.panel.data.series) || [];
  const out = [];
  for (const frame of frames) {
    if (!frame || !frame.fields) continue;
    const pf = frame.fields.find(f => String(f.name).toLowerCase() === 'problems') ||
      frame.fields.find(f => f.config && f.config.custom && f.config.custom.type === 'problems');
    if (!pf) continue;
    for (const p of vals(pf)) {
      if (p && typeof p === 'object') out.push(p);
    }
  }
  return out;
}

// Linhas 115-122: Definições de cores e nomenclaturas para cada nível de severidade do Zabbix (0 a 5)
const SEV = [
  { short: 'N/C', label: 'Não classificado', color: C.gray },
  { short: 'INFO', label: 'Informação', color: C.blue },
  { short: 'AVISO', label: 'Warning', color: C.yellow },
  { short: 'MÉDIA', label: 'Average', color: C.orange },
  { short: 'ALTA', label: 'High', color: C.red },
  { short: 'DESASTRE', label: 'Disaster', color: C.deepRed }
];

// Linhas 124-132: Converte e valida o valor de severidade
function sevRank(p) {
  const n = Number(p && p.severity);
  return Number.isFinite(n) && n >= 0 && n <= 5 ? n : null;
}

function sevMeta(p) {
  const n = sevRank(p);
  return n === null ? { short: 'N/C', label: 'Não classificado', color: C.gray } : SEV[n];
}

// Linhas 134-144: Extração de nome de host e grupos
function hostName(p) {
  const hs = p && p.hosts;
  if (Array.isArray(hs) && hs.length) return objName(hs[0]) || '—';
  return objName(hs) || '—';
}

function groupName(p) {
  const gs = p && p.groups;
  if (Array.isArray(gs) && gs.length) return gs.map(objName).filter(Boolean).slice(0, 2).join(' · ') || '';
  return objName(gs);
}

// Linhas 146-159: Verifica se o incidente foi reconhecido no Zabbix e resume quem reconheceu
function acknowledged(p) {
  if (!p) return false;
  if (p.acknowledged === true || p.acknowledged === 1 || String(p.acknowledged) === '1') return true;
  return Array.isArray(p.acknowledges) && p.acknowledges.length > 0;
}

function ackSummary(p) {
  const a = Array.isArray(p && p.acknowledges) ? p.acknowledges : [];
  if (!a.length) return 'Sem reconhecimento';
  const last = a[a.length - 1] || {};
  const who = last.user || [last.name, last.surname].filter(Boolean).join(' ') || 'usuário';
  const when = last.time || fmtDateTime(last.clock);
  return `${who}${when && when !== '—' ? ' · ' + when : ''}`;
}

// Linhas 161-171: Filtros por estado e janelas temporais de eventos recentes
function isRecovered(p) {
  return HISTORY_MODE && String(p && p.value) === '0';
}

function isRecent(p) {
  const ts = normalizeTime(p && p.timestamp);
  if (ts === null) return false;
  return (Date.now() - ts) <= RECENT_MINUTES * 60 * 1000;
}

const HISTORY_MODE = false;
const RECENT_MINUTES = 15;
const MIN_VISIBLE_SEVERITY = 3; // Oculta eventos abaixo de Average/Média

// Linhas 177-190: Ordena os problemas por severidade (decrescente) e data (mais recente primeiro)
let rows = problemsFromFrames();
rows = rows.filter(p => {
  const s = sevRank(p);
  return s === null || s >= MIN_VISIBLE_SEVERITY;
});
rows.sort((a, b) => {
  const sa = sevRank(a) ?? -1;
  const sb = sevRank(b) ?? -1;
  if (sb !== sa) return sb - sa;
  const ta = normalizeTime(a.timestamp) ?? 0;
  const tb = normalizeTime(b.timestamp) ?? 0;
  return tb - ta;
});

// Linhas 192-202: Totalizadores gerais para cabeçalho do painel
const total = rows.length;
const counts = [0, 0, 0, 0, 0, 0];
let acked = 0;
let recent = 0;
for (const p of rows) {
  const s = sevRank(p);
  if (s !== null) counts[s]++;
  if (acknowledged(p)) acked++;
  if (isRecent(p)) recent++;
}

// Linhas 204-216: Tratamento de estado vazio (quando não existem alertas)
if (!total) {
  return {
    backgroundColor: 'transparent',
    graphic: [
      roundedCard(), accentLine(C.cyan),
      txt(22, 27, "INCIDENTES ATIVOS · ALTA PRIORIDADE", 14, C.text, 850),
      txt(22, 53, "Problemas ativos Average, High e Disaster · reconhecimento, idade e dados operacionais.", 10, C.sub, 500),
      txt(22, Math.max(105, H * 0.48), 'Nenhum incidente encontrado', 18, C.text, 800),
      txt(22, Math.max(134, H * 0.48 + 30), 'A consulta Zabbix não retornou eventos visíveis para esta classificação.', 11, C.mute, 500)
    ],
    xAxis: { show: false }, yAxis: { show: false }, series: []
  };
}

// Linhas 218-245: Construção das colunas e cabeçalho dinâmico adaptável à largura da tela (Responsivo)
const visibleRows = Math.max(6, Math.min(13, Math.floor((H - 132) / 38)));
const compactMode = W < 1050;
const showGroup = W >= 1280;
const showOpdata = W >= 1180;
const showAck = W >= 900;

// Linhas 247-360: Função customizada de renderização de linhas (Custom Series renderItem)
// Desenha dinamicamente retângulos, ícones e textos para cada linha da tabela de incidentes.
function rowRender(params, api) {
  // Executa o desenho de cada linha com base no índice atual
  // Exibe severidade, data, host, problema/opdata e status ACK
  ...
}

// Linhas 362-375: Configuração de DataZoom para permitir rolagem de tabela no gráfico
const zoom = total > visibleRows ? [
  {
    type: 'inside', yAxisIndex: 0, startValue: 0, endValue: visibleRows - 1,
    zoomOnMouseWheel: false, moveOnMouseWheel: true, moveOnMouseMove: true
  },
  {
    type: 'slider', yAxisIndex: 0, right: 5, top: 112, bottom: 18, width: 7,
    startValue: 0, endValue: visibleRows - 1, showDetail: false, showDataShadow: false,
    borderColor: 'transparent', backgroundColor: 'rgba(127,141,168,.08)',
    fillerColor: 'rgba(23,199,213,.24)', handleSize: 0,
    textStyle: { color: C.mute }
  }
] : [];

// Linhas 377-425: Retorno final da estrutura do ECharts com dados, tooltips estilizadas e eixos
return {
  backgroundColor: 'transparent',
  animation: false,
  graphic: graphics,
  grid: { left: 16, right: total > visibleRows ? 18 : 12, top: 112, bottom: 16, containLabel: false },
  tooltip: { ... },
  xAxis: { type: 'value', min: 0, max: 100, show: false },
  yAxis: { type: 'category', inverse: true, show: false, data: rows.map((_, i) => String(i)) },
  dataZoom: zoom,
  series: [{ type: 'custom', coordinateSystem: 'cartesian2d', renderItem: rowRender, data: rows.map((_, i) => [0, String(i)]) }]
};
```

## 3. Análise do Painel: Alertas por Hora (panel-alerts-hour)
Este painel agrupa temporalmente os alertas do Zabbix em intervalos (buckets) de 1 hora.

### Agrupamento por Bucket de Hora:

```javascript
function hourBucket(ts) {
  const d = new Date(ts);
  d.setMinutes(0, 0, 0);
  return d.getTime();
}
```
A função zera os minutos e segundos para associar cada evento ao seu respectivo horário exato (ex: 11:00).

### Preenchimento de Lacunas (Zero-Filling):
Para manter a continuidade temporal sem quebras no gráfico de barras, o script calcula o intervalo do menor ao maior timestamp e preenche as horas vazias com 0 alertas:

```javascript
if (spanHours <= 744) {
  const filled = [];
  for (let t = minTs; t <= maxTs; t += 3600000) {
    filled.push([t, buckets.get(t) || 0]);
  }
  ordered = filled;
}
```

### Gradiente e Animação:
Cada barra é renderizada utilizando um gradiente linear ciano/azul com destaque para os valores de pico no topo (Pico X · DD/MM/AAAA HH:mm).

## 4. Análise do Painel: Problemas por Severidade (panel-severity-analytics)
Este painel utiliza o tipo pie (gráfico de rosca/donut) para resumir os totais por nível de criticidade:

### Contagem por Severidade:
Itera sobre a série de problemas do Zabbix, mapeando os ranks 3 (Média/Orange), 4 (Alta/Red) e 5 (Desastre/DeepRed).

### Indicador Central:
Exibe no centro do Donut o valor total de problemas ativos e identifica dinamicamente o nível dominante com maior quantidade de ocorrências (Maior: Alta).

## Fontes e Referências
[Grafana Documentation - Dashboard Schema](https://grafana.com/docs/grafana/latest/dashboards/)

[Business Charts Panel Configuration](https://grafana.com/grafana/plugins/volkovlabs-echarts-panel/)

[Apache ECharts - Custom Series & Rendering](https://echarts.apache.org/en/option.html#title)

[Zabbix API & Trigger Severities](https://www.zabbix.com/documentation/current/en/manual/api/reference)
