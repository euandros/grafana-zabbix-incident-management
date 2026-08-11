# Gerenciamento de Incidentes — Grafana Dashboard (Business Charts + Zabbix)

Este repositório contém o modelo completo (export em JSON / As-Code format) do dashboard **Gerenciamento de Incidentes** para o **Grafana (v12+)**, integrado diretamente ao **Zabbix** e renderizado utilizando o plugin **Business Charts (Volkov Labs - Apache ECharts)**.

## Demonstração do Dashboard
| Visão Geral do Dashboard |
| :---: |
| <img width="1672" height="941" alt="visao_geral_gerenciamento_de_incidentes" src="https://github.com/user-attachments/assets/13afb4c0-32ae-4018-8a91-3a74638de31b" /> |

| Distribuição de Problemas por Severidade | Histórico de Alertas por Hora |
| :---: | :---: |
| <img width="1448" height="1086" alt="grafico_problemas_por_severidade" src="https://github.com/user-attachments/assets/2db3ef08-57d9-4b9b-accb-07d1f07c50c7" /> | <img width="1448" height="1086" alt="grafico_alertas_por_hora" src="https://github.com/user-attachments/assets/6b2ae556-2a61-403a-be72-c286f8a985c6" /> |

## Visão Geral
O dashboard foi projetado para centros de operações de rede (NOC/SOC) que necessitam de uma visualização moderna, limpa e de alta performance dos problemas e alertas monitorados pelo **Zabbix**. 

Em vez de utilizar as tabelas padrão, o dashboard aproveita a capacidade do plugin **BusinessCharts** (antes Volkov Labs ECharts), para processar arrays complexos de dados diretamente via JavaScript, proporcionando:
- **Design moderno e escuro (Dark Theme)** otimizado para vídeo walls de monitoramento.
- **Normalização de datas** para o padrão brasileiro (`DD/MM/AAAA HH:mm`).
- **Tratamento dinâmico de severidades, reconhecimentos (acknowledges) e dados operacionais**.
- **Scrolagem fluída** e formatação gráfica de alta precisão.

## Estrutura e Painéis do Dashboard
O dashboard é composto por 5 estruturas principais organizadas em layout de grid:

1. **Header Personalizado (`panel-premium-header`)**:
   - Título dinâmico com indicador de atualização (`Refresh: 1 min`) e metadados da integração.
2. **Problemas Ativos por Severidade (`panel-severity-analytics`)**:
   - Gráfico de Rosca (Donut Chart) em Apache ECharts que agrupa e contabiliza os alertas por nível de severidade (*Média*, *Alta*, *Desastre*).
3. **Alertas por Hora · Histórico (`panel-alerts-hour`)**:
   - Gráfico de barras temporais em *buckets* de 1 hora, destacando os picos operacionais de eventos ao longo do dia.
4. **Incidentes Ativos · Prioridade (`panel-1`)**:
   - Fila operacional dos incidentes ativos (severidades *Average*, *High* e *Disaster*), exibindo:
     - Severidade codificada por cores.
     - Timestamp normalizado e tempo relativo desde a ocorrência.
     - Host de origem, grupo e dados operacionais (*opdata*).
     - Status de reconhecimento (Acknowledged / Pendente).
5. **Histórico de Incidentes · Prioridade (`panel-2`)**:
   - Registro histórico do comportamento das triggers e problemas, permitindo analisar eventos passados e verificar recuperações (*Recovered*).

## Pré-requisitos

Para importar e executar este dashboard, você precisará dos seguintes componentes instalados e configurados no seu ambiente Grafana:

* **Grafana**: Versão `12.x` ou superior.
* **Datasource Zabbix**: Plugin [`alexanderzobnin-zabbix-datasource`](https://grafana.com/grafana/plugins/alexanderzobnin-zabbix-app/).
  * *Nome configurado no datasource:* `zabbix-monitoramento` (pode ser alterado nas configurações de consulta do JSON).
* **Plugin de Visualização**: [`volkovlabs-echarts-panel`](https://grafana.com/grafana/plugins/volkovlabs-echarts-panel/) (Business Charts v7.2.5 ou superior).

## Como Utilizar

### 1. Clonar o Repositório
```bash
git clone [https://github.com/euandros/grafana-zabbix-incident-management.git](https://github.com/euandros/grafana-zabbix-incident-management.git)
cd grafana-zabbix-incident-management
```

### 2. Importar o Dashboard no Grafana
* Acesse sua instância do Grafana.
* Vá no menu lateral e selecione Dashboards -> Import.
* Faça o upload do arquivo dashboard.json localizado na raiz deste repositório (ou copie e cole o conteúdo do JSON).
* Selecione a sua fonte de dados (Datasource) do Zabbix no seletor, caso solicitado.
* Clique em Import.

## Explicação Detalhada do Código JSON / ECharts
Para entender o funcionamento interno das rotinas em JavaScript (Apache ECharts) que fazem o parse dos dados do Zabbix, veja o arquivo de explicação linha a linha:

* [DOCUMENTACAO_TECNICA.md](https://github.com/euandros/grafana-zabbix-incident-management/blob/main/DOCUMENTACAO_TECNICA.md)
