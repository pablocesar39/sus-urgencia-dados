# sus-urgencia-dados
Análise de internações de urgência e emergência no SUS utilizando Google BigQuery, SQL e Looker Studio.

# Análise de Otimização e Gargalos na Rede de Urgência e Emergência do SUS

## 1. Dissertação sobre o Problema

### Descrição do Problema
A sobrecarga nos serviços públicos de saúde e o tempo excessivo de espera para atendimentos de urgência e
 emergência no Brasil representam um gargalo crítico no Sistema Único de Saúde (SUS). A alocação ineficiente
 de recursos humanos e suprimentos, somada à imprevisibilidade da demanda sazonal, resulta em atrasos que
 impactam diretamente a resolutividade e a satisfação do usuário.

### Importância e Relevância Social
A saúde pública é um direito fundamental. Filas de espera prolongadas em unidades hospitalares levam a
 complicações evitáveis, superlotação e custos operacionais elevados por conta de internações prolongadas.
 Otimizar a gestão do fluxo de internações e gargalos de leitos garante uma distribuição mais equitativa de
 recursos e preservação de vidas.

### Como a Análise de Dados Ajuda na Solução
A análise de dados abertos possibilita identificar padrões temporais, espaciais e demográficos na internação
 de urgência. Através da modelagem de dados no Google BigQuery e visualização no Looker Studio, é possível:
1. Mapear períodos de alta demanda hospitalar e sazonalidade.
2. Mensurar a pressão de fluxo regional e a evasão de pacientes entre municípios.
3. Dimensionar de forma eficiente a ocupação e o custo por internação nas unidades federativas.

---

## 2. Fontes de Dados Públicas (BigQuery)

Utilizamos datasets consolidados de internação hospitalar hospedados no 
**Google BigQuery** (via projeto *Base dos Dados*).

| Fonte de Dados | Descrição | Tipo de Dados | Método de Acesso |
| :--- | :--- | :--- | :--- |
| **DataSUS (SIHSUS)** | Dados de internações hospitalares e Autorizações de Internação Hospitalar (AIH).
 | Estruturado | SQL via BigQuery (`basedosdados.br_ms_sih.aihs_reduzidas`) |
| **IBGE (Municípios)** | Diretório de municípios e diretrizes territoriais. | 
Estruturado | SQL via BigQuery (`basedosdados.br_bd_diretorios_brasil.municipio`) |

---

## 3. Scripts SQL de Modelagem e Tratamento (Dataset: `projetosemantix-512020.dados_sus`)

### Script 1: Mapeamento de Pressão de Fluxo Regional (Origem do Paciente)
```sql
CREATE OR REPLACE TABLE `projetosemantix-512020.dados_sus.tb_fluxo_regional` AS
SELECT 
    CASE SUBSTR(id_municipio_estabelecimento, 1, 2)
        WHEN '11' THEN 'RO' WHEN '12' THEN 'AC' WHEN '13' THEN 'AM' WHEN '14' THEN 'RR'
        WHEN '15' THEN 'PA' WHEN '16' THEN 'AP' WHEN '17' THEN 'TO' WHEN '21' THEN 'MA'
        WHEN '22' THEN 'PI' WHEN '23' THEN 'CE' WHEN '24' THEN 'RN' WHEN '25' THEN 'PB'
        WHEN '26' THEN 'PE' WHEN '27' THEN 'AL' WHEN '28' THEN 'SE' WHEN '29' THEN 'BA'
        WHEN '31' THEN 'MG' WHEN '32' THEN 'ES' WHEN '33' THEN 'RJ' WHEN '35' THEN 'SP'
        WHEN '41' THEN 'PR' WHEN '42' THEN 'SC' WHEN '43' THEN 'RS' WHEN '50' THEN 'MS'
        WHEN '51' THEN 'MT' WHEN '52' THEN 'GO' WHEN '53' THEN 'DF'
        ELSE 'Outro'
    END AS uf_estabelecimento,
    CASE 
        WHEN id_municipio_paciente = id_municipio_estabelecimento THEN 'Local'
        ELSE 'Outro Município (Migrante)'
    END AS origem_paciente,
    COUNT(id_aih) AS total_internacoes
FROM 
    `basedosdados.br_ms_sih.aihs_reduzidas`
WHERE 
    ano = 2023 
    AND carater_internacao = '2'
GROUP BY 
    uf_estabelecimento, 
    origem_paciente;

Script 2: Identificação de Padrões Sazonais de Atendimento


CREATE OR REPLACE TABLE `projetosemantix-512020.dados_sus.tb_sazonalidade_atendimento` AS
WITH dados_agregados AS (
    SELECT 
        CASE SUBSTR(id_municipio_estabelecimento, 1, 2)
            WHEN '11' THEN 'RO' WHEN '12' THEN 'AC' WHEN '13' THEN 'AM' WHEN '14' THEN 'RR'
            WHEN '15' THEN 'PA' WHEN '16' THEN 'AP' WHEN '17' THEN 'TO' WHEN '21' THEN 'MA'
            WHEN '22' THEN 'PI' WHEN '23' THEN 'CE' WHEN '24' THEN 'RN' WHEN '25' THEN 'PB'
            WHEN '26' THEN 'PE' WHEN '27' THEN 'AL' WHEN '28' THEN 'SE' WHEN '29' THEN 'BA'
            WHEN '31' THEN 'MG' WHEN '32' THEN 'ES' WHEN '33' THEN 'RJ' WHEN '35' THEN 'SP'
            WHEN '41' THEN 'PR' WHEN '42' THEN 'SC' WHEN '43' THEN 'RS' WHEN '50' THEN 'MS'
            WHEN '51' THEN 'MT' WHEN '52' THEN 'GO' WHEN '53' THEN 'DF'
            ELSE 'Outro'
        END AS sigla_uf,
        mes,
        COUNT(id_aih) AS volume_atendimentos
    FROM 
        `basedosdados.br_ms_sih.aihs_reduzidas`
    WHERE 
        ano = 2023
    GROUP BY 
        sigla_uf, 
        mes
)
SELECT 
    sigla_uf,
    mes,
    volume_atendimentos,
    AVG(volume_atendimentos) OVER(PARTITION BY sigla_uf) AS media_mensal_uf
FROM 
    dados_agregados
ORDER BY 
    sigla_uf, 
    mes;


Script 3: Agregação e Custo Médio por Município


CREATE OR REPLACE TABLE `projetosemantix-512020.dados_sus.tb_urgencia_municipio` AS
WITH urgencia_tratada AS (
  SELECT 
    id_municipio_estabelecimento AS municipio_id,
    DATE(CAST(ano AS INT64), CAST(mes AS INT64), 1) AS data_atendimento,
    COUNT(id_aih) AS total_procedimentos,
    SUM(valor_aih) AS custo_total
  FROM 
    `basedosdados.br_ms_sih.aihs_reduzidas`
  WHERE 
    ano >= 2022
    AND carater_internacao = '2'
  GROUP BY 
    municipio_id, data_atendimento
)
SELECT 
  u.municipio_id,
  m.nome AS nome_municipio,
  m.sigla_uf,
  u.data_atendimento,
  u.total_procedimentos,
  u.custo_total,
  SAFE_DIVIDE(u.custo_total, u.total_procedimentos) AS custo_medio_por_atendimento
FROM 
  urgencia_tratada u
LEFT JOIN 
  `basedosdados.br_bd_diretorios_brasil.municipio` m 
ON 
  u.municipio_id = m.id_municipio
ORDER BY 
  u.total_procedimentos DESC;


## 4. Relatório de Insights e Recomendações
Pressão de Fluxo Regional: Identificou-se que cerca de 32,5% das internações de urgência envolveram o
 deslocamento de pacientes para fora de seu município de residência, evidenciando sobrecarga em polos 
regionais de saúde.

Concentração por Estado: Estados como São Paulo e Minas Gerais absorvem volumes massivos tanto de 
demanda local quanto migrante.

## 5. Visualização no Looker Studio
O projeto está estruturado com tabelas agregadas e otimizadas no BigQuery (projetosemantix-512020.dados_sus),
garantindo alta performance, carregamento instantâneo e ausência de travamentos ao alimentar os 
gráficos e mapas no Looker Studio.

[Acessar o Dashboard Interativo no Looker Studio]
https://datastudio.google.com/reporting/4f23da42-5bea-4039-8d5f-81b7173464c3

### Prévia do Painel
![Preview do Dashboard] (assets/Preview.png)

## 6. Recomendações e Plano de Ação

1. Ações Focadas em Gestão de Recursos e Custos
Auditoria e Ajuste Orçamentário por Custo Médio:

Contexto do Dado: A análise de custos por atendimento (custo_medio_por_atendimento) revelou discrepâncias em 
determinados municípios.

Ação Prática: Direcionar auditorias e repasses financeiros diferenciados para os municípios com custos 
desproporcionalmente altos, avaliando se a ineficiência decorre de contratos de insumos, complexidade dos
 casos ou ociosidade estrutural.

Otimização de Leitos de Retaguarda:

Contexto do Dado: O alto volume de procedimentos de urgência nas unidades centrais.

Ação Prática: Investir na ampliação de leitos de retaguarda em hospitais de menor complexidade (médica) nas
redes municipais vizinhas, evitando que o paciente precise obrigatoriamente se descolar ou sobrecarregar 
os grandes hospitais de referência.

2. Ações Focadas no Fluxo Regional e Migração de Pacientes
Consórcios Intermunicipais de Saúde:

Contexto do Dado: O indicador de pacientes migrantes (origem_paciente = 'Outro Município') evidenciou que
 polos urbanos absorvem uma carga elevada de pacientes de cidades vizinhas desprovidas de infraestrutura 
adequada.

Ação Prática: Estabelecer acordos de cooperação e consórcios intermunicipais de financiamento partilhado.
Os municípios de origem que não possuem estrutura de alta urgência devem comparticipar financeiramente 
na manutenção dos hospitais de referência que absorvem a sua população.

Fortalecimento da Atenção Básica (UBS e UPA locais):

Contexto do Dado: Grande parte da sobrecarga de urgência decorre de casos de baixa e média complexidade que
 poderiam ser resolvidos na rede básica.

Ação Prática: Expandir o horário de atendimento de unidades básicas de saúde nos municípios emissores de 
migrantes, reduzindo a necessidade de deslocamento para os pronto-socorros centrais.

3. Ações Focadas em Planejamento Sazonal e Operacional
Escalas Dinâmicas e Alertas Preditivos (Sazonalidade):

Contexto do Dado: Os gráficos temporais mensais demonstraram picos consistentes de internações em 
determinados períodos do ano (ex: surtos sazonais respiratórios ou epidemias).

Ação Prática: Implementar um modelo preditivo baseado nos históricos mensais para que os gestores estaduais
 e municipais realizem contratações temporárias de profissionais de saúde e comprasantecipadas de insumos 30
 dias antes dos picos históricos identificados no dashboard.

Campanhas de Prevenção Direcionadas:

Contexto do Dado: A concentração de atendimentos em meses específicos por região.

Ação Prática: Lancar campanhas de vacinação e conscientização alinhadas com a janela temporal exata em que
 os dados mostram a subida de internações, mitigando a pressão sobre as redes de urgência.
