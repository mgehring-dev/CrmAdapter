# Análise de Integração College - Central de Conversão

## 1. Mapeamento de Endpoints com Integração College

Todos os endpoints abaixo foram marcados com `Tags = new[] { "Integração College" }` no Swagger.

| Controller | Método | Rota | Descrição | Integração College |
|---|---|---|---|---|
| **LeadController** | PATCH | `/api/v1/lead/IntegrarComGVCollege/{idLead}` | Integra lead único com College | `IntegracaoCollegeLead.InscreverAluno()` |
| **LeadController** | PATCH | `/api/v1/lead/IntegrarComGVCollege/multiplosLeads` | Integra múltiplos leads com College | `IntegracaoCollegeLead.InscreverAluno()` (batch) |
| **CampanhaController** | GET | `/api/v1/campanha/Integracao/CamposDeIntegracaoGvCollegeInscreverAluno` | Campos mapeados para inscrição | Mapeamento de campos do College |
| **ConfiguracaoTaxaController** | GET | `/api/v1/ConfiguracaoTaxa/LocaisPagamento/{idOferta}` | Locais de pagamento da oferta | `IntegracaoCollegeLocalPagamento` |
| **ConfiguracaoTaxaController** | GET | `/api/v1/ConfiguracaoTaxa/ContasBancarias/{idOferta}/{codigoBanco}` | Contas bancárias por banco | `IntegracaoCollegeContaBancaria` |
| **ConfiguracaoTaxaController** | GET | `/api/v1/ConfiguracaoTaxa/EventosFinanceiros/{idOferta}` | Eventos financeiros da oferta | `IntegracaoCollegeConfiguracaoParcela` |
| **OrigemMatriculaController** | GET | `/api/v1/origemmatricula/GVCollege` | Lista origens de matrícula | `IntegracaoCollegeOrigensMatricula` |
| **OrigemMatriculaController** | GET | `/api/v1/origemmatricula/IntegracaoGVCollege` | Config de integração College | Consulta entidade `IntegracaoGVCollege` |
| **TurmaController** | GET | `/api/v1/turma/GVCollegeParaEquivalenciaOfertas` | Turmas para equivalência | `IntegracaoCollegeTurmasPrincipais` |
| **EquivalenciaController** | POST | `/api/v1/equivalencia/GVCollege` | Cria equivalência curso↔oferta | Persiste mapeamento College→Central |
| **RetencaoController** | GET | `/api/v1/retencao/FonteDados` | Fontes OData do College | `IntegracaoCollegeIndicador.GetFonteDados()` |
| **RetencaoController** | GET | `/api/v1/retencao/CamposFonteDados` | Metadados de fonte OData | `IntegracaoCollegeIndicador.GetCamposFonteDados()` |

---

## 2. Arquitetura Atual de Integração

```
┌─────────────────────────────────────────────────────────┐
│                    Controllers (API)                      │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│            Domain Services (Orquestração)                 │
│  IntegracaoService, TurmaService, RetencaoService        │
└──────────────────────┬──────────────────────────────────┘
                       │ Injeta interfaces (IIntegracao*)
┌──────────────────────▼──────────────────────────────────┐
│       Integracao.Base/Interfaces (Contratos)              │
│  IIntegracaoAPI, IIntegracaoLead, IIntegracaoIndicador   │
│  IIntegracaoEstabelecimentos, IIntegracaoOrigensMatricula│
│  IIntegracaoLocalPagamento, IIntegracaoContaBancaria...  │
└──────────────────────┬──────────────────────────────────┘
                       │ Implementação direta
┌──────────────────────▼──────────────────────────────────┐
│           Integracao.GVCollege/Classes                    │
│  IntegracaoCollegeAPI, IntegracaoCollegeLead...          │
│  (Acoplamento direto com endpoints do College)           │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP REST
┌──────────────────────▼──────────────────────────────────┐
│               GVCollege API (ERP)                         │
│  /inscreverAluno, /turmasprincipais, /odata, etc.        │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Proposta: Adapter Pattern para Multi-ERP

### 3.1 Ponto de Inserção do Adapter

O ponto ideal para inserir o Adapter Pattern é **entre as interfaces base (`Integracao.Base/Interfaces`) e os Domain Services**. A arquitetura já possui interfaces, mas o DI registra diretamente as implementações do College. Precisamos de:

1. **Uma camada de abstração ERP-agnostic** (Adapter)
2. **Um Factory/Strategy** para resolver qual implementação usar por tenant
3. **Normalização dos DTOs** de entrada/saída

### 3.2 Arquitetura Proposta

```
┌─────────────────────────────────────────────────────────┐
│                    Controllers (API)                      │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│            Domain Services (Orquestração)                 │
│  IntegracaoService, TurmaService, RetencaoService        │
└──────────────────────┬──────────────────────────────────┘
                       │ Injeta IErpAdapter
┌──────────────────────▼──────────────────────────────────┐
│        IErpAdapter (Novo - Contrato Unificado)           │
│  ├─ InscreverAluno(InscricaoUnificadaDTO)                │
│  ├─ GetTurmas(filtros)                                   │
│  ├─ GetLocaisPagamento(idOferta)                         │
│  ├─ GetContasBancarias(idOferta, banco)                  │
│  ├─ GetOrigensMatricula()                                │
│  ├─ GetFonteDadosOData()                                 │
│  └─ GetEstabelecimentos()                                │
└──────────────────────┬──────────────────────────────────┘
                       │ Strategy/Factory por Tenant
         ┌─────────────┼─────────────┐
         │             │             │
┌────────▼───┐  ┌─────▼──────┐  ┌──▼──────────┐
│ CollegeErp │  │ TocerErp   │  │ ThirdParty  │
│  Adapter   │  │  Adapter   │  │   Adapter   │
└────────────┘  └────────────┘  └─────────────┘
         │             │             │
    College API    Tocer API    Outro ERP API
```

### 3.3 Estrutura de Pastas Proposta

```
src/GVCentralConversao.Domain/
└── Integracao/
    ├── Integracao.Base/
    │   ├── Interfaces/
    │   │   ├── IErpAdapter.cs              ← NOVO (contrato unificado)
    │   │   ├── IErpAdapterFactory.cs       ← NOVO (factory por tenant)
    │   │   └── ... (manter existentes)
    │   └── DTOs/
    │       ├── Unificados/                 ← NOVO (DTOs normalizados)
    │       │   ├── InscricaoUnificadaDTO.cs
    │       │   ├── TurmaUnificadaDTO.cs
    │       │   └── ...
    │       └── ... (manter existentes)
    ├── Integracao.GVCollege/
    │   ├── CollegeErpAdapter.cs            ← NOVO (implementa IErpAdapter)
    │   └── Classes/ (manter existentes, usados internamente pelo adapter)
    ├── Integracao.TerceirosERP/            ← NOVO
    │   ├── TerceiroErpAdapter.cs
    │   └── Classes/
    └── Integracao.OData/                   ← NOVO (reutilizável)
        ├── ODataErpAdapter.cs
        └── ODataQueryBuilder.cs
```

### 3.4 Exemplo de Implementação

```csharp
// Contrato Unificado
public interface IErpAdapter
{
    Task<ResultadoInscricao> InscreverAluno(InscricaoUnificadaDTO inscricao);
    Task<IEnumerable<TurmaUnificadaDTO>> GetTurmas(FiltroTurma filtro);
    Task<IEnumerable<LocalPagamentoUnificadoDTO>> GetLocaisPagamento(Guid idOferta);
    Task<IEnumerable<ContaBancariaUnificadaDTO>> GetContasBancarias(Guid idOferta, string banco);
    Task<IEnumerable<OrigemMatriculaUnificadaDTO>> GetOrigensMatricula();
    Task<IEnumerable<FonteDadosUnificadaDTO>> GetFonteDados();
    Task<IEnumerable<EstabelecimentoUnificadoDTO>> GetEstabelecimentos();
    Task<StatusIntegracao> ObterStatusProcessamento(string idIntegracao);
}

// Factory
public interface IErpAdapterFactory
{
    IErpAdapter Create(string tipoErp); // "college", "tocer", "odata-generico"
}

// DI Registration
builder.Services.AddScoped<IErpAdapterFactory, ErpAdapterFactory>();
builder.Services.AddScoped<IErpAdapter>(sp =>
{
    var factory = sp.GetRequiredService<IErpAdapterFactory>();
    var tenant = sp.GetRequiredService<ITenantProvider>();
    return factory.Create(tenant.TipoErp);
});
```

---

## 4. Estimativa de Tempo para Desacoplamento

| Fase | Atividade | Estimativa |
|------|-----------|-----------|
| **1. Fundação** | Criar `IErpAdapter`, DTOs unificados, Factory | 3-5 dias |
| **2. College Adapter** | Implementar `CollegeErpAdapter` usando classes existentes | 5-8 dias |
| **3. Refatorar Services** | Trocar injeção direta por `IErpAdapter` nos Services | 5-7 dias |
| **4. OData Genérico** | Extrair lógica OData para adapter reutilizável | 3-5 dias |
| **5. Endpoints de Especificação** | Criar endpoints OData + especificação para terceiros | 8-12 dias |
| **6. Testes** | Unit + Integration tests para adapters | 5-7 dias |
| **7. Terceiro ERP** | Implementar primeiro adapter de terceiro | 5-8 dias |
| **TOTAL** | | **~34-52 dias** (7-10 sprints) |

### Riscos e Complexidades
- A `RetencaoService` faz chamadas OData diretas com `HttpClient` (não passa pelo `IIntegracaoAPI`)
- O `IntegracaoService` possui lógica de negócio acoplada a DTOs do College (ex: `InscricaoAluno`)
- Cache por tenant pode conflitar com multi-ERP no mesmo tenant

---

## 5. Funcionalidades que Precisam Ser Criadas

### 5.1 Para Atender Multi-ERP

| # | Funcionalidade | Descrição |
|---|---|---|
| 1 | **Configuração de ERP por Tenant** | Tela admin para selecionar qual ERP o tenant usa (College, Tocer, etc.) |
| 2 | **Mapeamento de Campos Dinâmico** | Configuração de equivalência de campos entre Central de Conversão e cada ERP |
| 3 | **Adapter Registry** | Registro dinâmico de adapters disponíveis no sistema |
| 4 | **Health Check por ERP** | Verificação de saúde/disponibilidade de cada ERP conectado |
| 5 | **Fallback e Retry** | Política de retry com Polly para cada adapter ERP |
| 6 | **Log de Integração Unificado** | Dashboard único de logs de integração independente do ERP |
| 7 | **Webhook Receiver** | Endpoint genérico para receber callbacks de ERPs terceiros |

### 5.2 Para Enviar Candidatos para ERPs Educacionais

| # | Funcionalidade | Descrição |
|---|---|---|
| 1 | **API de Especificação (Outbound)** | Endpoints padronizados que terceiros possam consultar (pull) |
| 2 | **Push Webhook** | Envio ativo de candidatos via webhook para ERPs que suportem |
| 3 | **Fila de Integração** | Service Bus queue para envio assíncrono com retry |
| 4 | **Transformador de Payload** | Mapeia o lead para o formato esperado por cada ERP |
| 5 | **Endpoint de Status** | API para consultar status da integração independente do ERP |
| 6 | **Batch Export** | Exportação em lote de leads para ERPs que não suportem realtime |

---

## 6. Ferramentas para Enviar Candidatos para ERPs Educacionais

### 6.1 Estratégia OData + Endpoints de Especificação

```
┌─────────────────────────────────────────────────────────────┐
│               Central de Conversão (Provedor)                │
│                                                              │
│  /odata/candidatos         → OData endpoint (query)          │
│  /api/v1/spec/candidatos   → REST endpoint (padrão)          │
│  /api/v1/webhook/push      → Envio ativo para ERP            │
│  /api/v1/integracao/status → Status consolidado              │
└──────────────────────────┬──────────────────────────────────┘
                           │
         ┌─────────────────┼────────────────────┐
         │ (OData Pull)    │ (REST Pull)        │ (Webhook Push)
         ▼                 ▼                    ▼
   ┌──────────┐     ┌──────────┐        ┌──────────┐
   │ ERP que  │     │ ERP com  │        │ ERP com  │
   │ suporta  │     │ API REST │        │ webhook  │
   │  OData   │     │  padrão  │        │ receiver │
   └──────────┘     └──────────┘        └──────────┘
```

### 6.2 Adapter no Backend: ERPs Internos vs Terceiros

```csharp
// Adapter para ERPs internos (GVdasa)
public class CollegeErpAdapter : IErpAdapter
{
    // Usa IntegracaoCollegeAPI existente (token interno, endpoints conhecidos)
}

// Adapter para ERPs terceiros (genérico)
public class ThirdPartyErpAdapter : IErpAdapter
{
    // Configuração dinâmica: URL base, auth type, field mapping
    // Lê configuração do tenant: qual ERP, quais endpoints, qual auth
}

// Adapter OData genérico
public class ODataErpAdapter : IErpAdapter
{
    // Para ERPs que expõem OData, reutiliza lógica do RetencaoService
    // mas de forma genérica e configurável
}
```

---

## 7. Retenção: O Que Precisa Funcionar

### 7.1 Requisitos para Retenção Multi-ERP

| Requisito | Detalhe |
|-----------|---------|
| **Fonte de dados OData** | Cada ERP deve expor dados via OData OU a Central deve ter adapter para extrair dados |
| **Dados mínimos obrigatórios** | Notas, Frequência, Parcelas Pendentes, Ocorrências Pedagógicas |
| **Views padronizadas** | `crm_vwnotas`, `crm_vwparcelaspendentes`, `crm_vwfrequencia`, `crm_vwocorrenciapedagogica` |
| **Autenticação** | Token Bearer por tenant com acesso ao OData do ERP |
| **Cache de metadados** | `$metadata` do OData para descobrir campos disponíveis |
| **Filtro por estabelecimento** | Campo `codigoEmpresa` obrigatório em todas as views |
| **Paginação** | Suporte a `$top`, `$skip`, `@odata.nextLink` |

### 7.2 Para ERPs Terceiros que NÃO Possuem OData

```
Opção A: ERP terceiro expõe OData → Usar ODataErpAdapter (plug & play)

Opção B: ERP terceiro expõe REST → Criar adapter que:
  1. Consulta REST do ERP
  2. Normaliza dados para formato OData interno
  3. Armazena em cache/banco intermediário
  4. RetencaoService consome dados normalizados

Opção C: ERP terceiro faz push → Criar webhook receiver que:
  1. Recebe dados via webhook
  2. Persiste em tabela intermediária
  3. RetencaoService consulta tabela local (não ERP)
```

### 7.3 Contrato Mínimo de Dados para Retenção

```csharp
// Cada ERP deve fornecer (via adapter):
public interface IRetencaoDataProvider
{
    Task<ODataResponse<NotaDTO>> GetNotas(List<string> estabelecimentos, List<Regra> filtros);
    Task<ODataResponse<ParcelaPendenteDTO>> GetParcelasPendentes(List<string> estabelecimentos, List<Regra> filtros);
    Task<ODataResponse<FrequenciaDTO>> GetFrequencia(List<string> estabelecimentos, List<Regra> filtros);
    Task<ODataResponse<OcorrenciaDTO>> GetOcorrencias(List<string> estabelecimentos, List<Regra> filtros);
    Task<IEnumerable<FonteDadosUnificadaDTO>> GetFontesDisponiveis();
    Task<IEnumerable<CampoMetadataDTO>> GetMetadados(string fonteDados);
}
```

---

## 8. Mudança de Arquitetura Necessária

### 8.1 De → Para

| Aspecto | Atual | Proposto |
|---------|-------|----------|
| **DI** | `IIntegracaoLead → IntegracaoCollegeLead` (fixo) | `IErpAdapter → Factory.Create(tenant.TipoErp)` |
| **DTOs** | DTOs específicos do College (`InscricaoAluno`, `PessoaCollegeDTO`) | DTOs unificados + mapping por adapter |
| **OData** | HttpClient direto no `RetencaoService` | `IRetencaoDataProvider` com adapters |
| **Config** | URL do College no `ITenantProvider` | Config multi-ERP: tipo, URL, auth, mappings |
| **Endpoints** | Apenas consumidor (chama College) | Consumidor + Provedor (expõe OData/REST para terceiros) |
| **Auth** | Token College via CAC | Auth configurável por ERP (OAuth, API Key, Basic) |

### 8.2 Impacto na Base de Dados

Novas tabelas necessárias:
- `ConfiguracaoErp` - tipo de ERP, URL, credenciais por tenant
- `MapeamentoCamposErp` - de/para de campos entre Central e ERP
- `LogIntegracaoUnificado` - logs independentes do ERP
- `DadosRetencaoCache` - cache local para ERPs sem OData

### 8.3 Impacto no Frontend

- Configuração de ERP (hoje assume College)
- Mapeamento de campos dinâmico
- Dashboard de status multi-ERP
- Seleção de ERP destino ao integrar leads

---

## 9. Resumo Executivo

### Cenário Atual
- 12 endpoints com integração direta ao College
- Interfaces existentes (`IIntegracaoLead`, etc.) facilitam desacoplamento
- RetencaoService tem acoplamento forte (HttpClient direto)
- Arquitetura 80% preparada para Adapter Pattern

### O Que Já Existe a Favor
- Interfaces base em `Integracao.Base/Interfaces/`
- Separação clara entre contratos e implementações
- DI já configurado via interfaces (fácil trocar implementação)
- Padrão de DTOs separados por integração

### Próximos Passos Recomendados
1. Criar `IErpAdapter` como contrato unificado
2. Implementar `CollegeErpAdapter` usando classes existentes (sem breaking changes)
3. Criar `IErpAdapterFactory` com resolução por tenant
4. Migrar `IntegracaoService` para usar `IErpAdapter`
5. Extrair OData do `RetencaoService` para `IRetencaoDataProvider`
6. Criar endpoints de especificação (OData provider)
7. Implementar primeiro adapter terceiro como POC
