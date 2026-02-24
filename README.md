# ERP Brechó — Concept Document

---

## Visão Geral

Sistema completo de gestão para brechós com dois domínios distintos: um **ERP interno** para operação do negócio (estoque, vendas, financeiro, funcionários) e uma **loja virtual pública** voltada para clientes finais. Ambos compartilham a mesma base de dados e infraestrutura, com separação clara de contextos e responsabilidades.

---

## Arquitetura Geral

```
┌─────────────────────────────────────────────────────────────┐
│                        INFRAESTRUTURA                       │
│                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌──────────────┐   │
│   │  Next.js    │    │  Next.js    │    │  Laravel     │   │
│   │  (ERP UI)   │    │  (Loja)     │    │  (API)       │   │
│   └──────┬──────┘    └──────┬──────┘    └──────┬───────┘   │
│          │                  │                  │           │
│          └──────────────────┴──────────────────┘           │
│                             │                              │
│                   ┌─────────┴──────────┐                   │
│                   │   PostgreSQL        │                   │
│                   │   Redis             │                   │
│                   └────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Stack Tecnológica

### Back-end

```yaml
framework: Laravel 11
linguagem: PHP 8.3
autenticação: Laravel Sanctum (SPA tokens) + Laravel Passport (OAuth para loja)
filas: Laravel Horizon + Redis
cache: Redis (tags, sessões, rate limiting)
busca: Laravel Scout + PostgreSQL Full-Text Search
storage: Laravel Filesystem (S3-compatible para imagens de produtos)
eventos: Laravel Events + Queued Listeners
websockets: Laravel Reverb (notificações em tempo real no ERP)
testes: Pest PHP
```

### Front-end

```yaml
framework: Next.js 14 (App Router)
linguagem: TypeScript
estado_global: Zustand
data_fetching: TanStack Query (React Query)
forms: React Hook Form + Zod
ui_erp: Shadcn/ui + Tailwind CSS
ui_loja: Componentes customizados + Tailwind CSS
animacoes: Framer Motion
testes: Vitest + Testing Library + Playwright (e2e)
```

### Banco de Dados

```yaml
principal: PostgreSQL 16
  - schemas separados por domínio (erp, store, auth)
  - full-text search nativo
  - JSONB para atributos dinâmicos de produtos
  - Row Level Security para isolamento de dados

cache_sessoes: Redis 7
  - cache de listagens e produtos (TTL configurável)
  - sessões de usuário (loja)
  - filas de jobs (Horizon)
  - pub/sub para eventos em tempo real
```

---

## Módulos do ERP Interno

### 1. Autenticação e Controle de Acesso

```php
// Roles disponíveis no ERP
enum Role: string {
    case ADMIN       = 'admin';
    case GERENTE     = 'gerente';
    case ESTOQUISTA  = 'estoquista';
    case CAIXA       = 'caixa';
    case VENDEDOR    = 'vendedor';
}
```

Cada role possui permissões granulares via Spatie Laravel Permission. O painel de controle de acesso permite criar perfis customizados.

---

### 2. Módulo de Estoque

Responsável pelo ciclo completo de vida das peças: entrada, triagem, etiquetagem, movimentação e baixa.

```
Fluxo de uma peça:
  RECEBIMENTO → TRIAGEM → PRECIFICAÇÃO → ETIQUETAGEM → DISPONÍVEL → VENDIDO/DESCARTADO
```

**Entidades principais:**

```sql
-- Produto (peça de brechó)
produtos (
  id              uuid primary key,
  codigo          varchar unique,       -- gerado automaticamente (BCH-2024-00001)
  nome            varchar,
  descricao       text,
  categoria_id    uuid references categorias,
  condicao        enum('otimo','bom','regular','para_reforma'),
  tamanho         varchar,
  cor             varchar[],
  genero          enum('masculino','feminino','unissex','infantil'),
  marca           varchar,
  origem_id       uuid references origens,  -- doação, compra, consignado
  preco_custo     numeric(10,2),
  preco_venda     numeric(10,2),
  preco_promocional numeric(10,2),
  status          enum('triagem','disponivel','reservado','vendido','descartado'),
  publicado_loja  boolean default false,
  atributos       jsonb,                -- campos extras dinâmicos
  fotos           text[],               -- URLs S3
  criado_em       timestamptz,
  atualizado_em   timestamptz
)
```

**Funcionalidades:**

- Cadastro em lote com geração automática de código de barras/QR Code
- Etiquetas para impressão (PDF gerado no Laravel)
- Histórico completo de movimentações por peça
- Relatório de giro de estoque por categoria/período
- Alertas de peças paradas (configurable aging)
- Importação via planilha CSV

---

### 3. Módulo de Vendas (PDV)

Interface de caixa otimizada para velocidade, com suporte a leitura de código de barras.

```
Venda:
  ┌────────────────────────────────────────────┐
  │  Scanner / busca manual de produtos        │
  │  Carrinho com cálculo automático           │
  │  Aplicação de descontos (% ou R$)          │
  │  Seleção de forma de pagamento             │
  │  Emissão de comprovante (PDF/impressão)    │
  └────────────────────────────────────────────┘
```

**Formas de pagamento suportadas:**
- Dinheiro (com cálculo de troco)
- PIX (integração via API do banco, QR Code dinâmico)
- Cartão de débito/crédito (registro manual ou integração com maquininha via SDK)
- Pagamento misto (combinação de formas)
- Vale/crédito de cliente

**Entidades:**

```sql
vendas (
  id              uuid primary key,
  numero          serial,
  cliente_id      uuid references clientes,
  operador_id     uuid references usuarios,
  subtotal        numeric(10,2),
  desconto        numeric(10,2),
  total           numeric(10,2),
  status          enum('aberta','finalizada','cancelada','estornada'),
  canal           enum('pdv','loja_online'),
  criado_em       timestamptz
)

itens_venda (
  id              uuid primary key,
  venda_id        uuid references vendas,
  produto_id      uuid references produtos,
  quantidade      integer default 1,
  preco_unitario  numeric(10,2),
  desconto_item   numeric(10,2)
)

pagamentos_venda (
  id              uuid primary key,
  venda_id        uuid references vendas,
  forma           enum('dinheiro','pix','debito','credito','vale'),
  valor           numeric(10,2),
  referencia      varchar,   -- NSU, txid PIX, etc.
  confirmado      boolean
)
```

---

### 4. Módulo Financeiro

Controle de caixa, contas a pagar e receber, e relatórios financeiros.

**Funcionalidades:**
- Abertura e fechamento de caixa com conferência
- Sangrias e suprimentos
- Contas a pagar (fornecedores, aluguel, despesas)
- Contas a receber (consignados, vendas parceladas)
- Fluxo de caixa diário/semanal/mensal
- DRE simplificado
- Conciliação bancária manual

```sql
lancamentos_financeiros (
  id              uuid primary key,
  tipo            enum('entrada','saida'),
  categoria       varchar,
  descricao       text,
  valor           numeric(10,2),
  data_vencimento date,
  data_pagamento  date,
  status          enum('pendente','pago','cancelado'),
  referencia_id   uuid,       -- venda, compra, despesa
  referencia_tipo varchar,
  criado_por      uuid references usuarios
)
```

---

### 5. Módulo de Clientes (CRM Básico)

```sql
clientes (
  id              uuid primary key,
  nome            varchar,
  cpf             varchar unique,
  email           varchar unique,
  telefone        varchar,
  data_nascimento date,
  endereco        jsonb,
  saldo_credito   numeric(10,2) default 0,
  total_compras   numeric(10,2) default 0,
  qtd_compras     integer default 0,
  tags            text[],
  observacoes     text,
  criado_em       timestamptz
)
```

**Funcionalidades:**
- Ficha completa do cliente com histórico de compras
- Programa de fidelidade (pontos por compra)
- Saldo de crédito (troca, vale)
- Segmentação por tags para campanhas
- Exportação de base para email marketing

---

### 6. Módulo de Consignação

Brechós frequentemente trabalham com consignados. Este módulo gerencia o ciclo completo.

```
Consignado:
  CADASTRO FORNECEDOR → ENTRADA PEÇAS → PRAZO → VENDA/DEVOLUÇÃO → ACERTO FINANCEIRO
```

```sql
consignacoes (
  id              uuid primary key,
  fornecedor_id   uuid references fornecedores,
  data_entrada    date,
  data_prazo      date,
  percentual_loja numeric(5,2),   -- ex: 40% para o brechó
  status          enum('ativo','encerrado','aguardando_acerto'),
  total_recebido  numeric(10,2),
  total_a_pagar   numeric(10,2)
)

itens_consignacao (
  id              uuid primary key,
  consignacao_id  uuid references consignacoes,
  produto_id      uuid references produtos,
  preco_combinado numeric(10,2),
  status          enum('disponivel','vendido','devolvido')
)
```

---

### 7. Módulo de Relatórios e Dashboard

Dashboard em tempo real com métricas via WebSocket (Laravel Reverb).

**KPIs exibidos:**
- Faturamento do dia/semana/mês
- Ticket médio
- Produtos mais vendidos
- Categorias em destaque
- Estoque atual (total e por categoria)
- Peças paradas há mais de X dias
- Vendas por operador
- Comparativo com período anterior

**Relatórios exportáveis (PDF/Excel):**
- Inventário completo
- Movimentação de estoque
- Vendas por período
- Desempenho por categoria
- Acerto de consignados
- Fluxo de caixa

---

## Loja Virtual Pública

### Visão Geral

Next.js com SSR/ISR para SEO otimizado. Catálogo público de produtos disponíveis, com carrinho, checkout e acompanhamento de pedidos.

```
Loja:
  HOME → CATÁLOGO → PRODUTO → CARRINHO → CHECKOUT → CONFIRMAÇÃO → ACOMPANHAMENTO
```

---

### Páginas e Rotas

```
/                          → Home (destaques, banners, categorias)
/colecao                   → Catálogo com filtros
/colecao/[categoria]       → Catálogo filtrado por categoria
/produto/[slug]            → Página do produto
/carrinho                  → Carrinho de compras
/checkout                  → Fluxo de checkout
/checkout/confirmacao      → Confirmação do pedido
/conta                     → Área do cliente (autenticado)
/conta/pedidos             → Lista de pedidos
/conta/pedidos/[id]        → Detalhe do pedido
/conta/favoritos           → Lista de desejos
/conta/credito             → Saldo e histórico de crédito
```

---

### Funcionalidades da Loja

**Catálogo:**
- Filtros por categoria, tamanho, cor, condição, faixa de preço, gênero
- Ordenação por novidade, preço, relevância
- Busca full-text com sugestões em tempo real (Redis cache)
- Paginação com infinite scroll ou paginação clássica
- Produto marcado como "última unidade" automaticamente
- Fotos com zoom e galeria

**Checkout:**
- Cálculo de frete via API dos Correios / Melhor Envio
- Endereço salvo para clientes cadastrados
- Cupons de desconto
- PIX (QR Code + cópia e cola) via integração bancária
- Cartão de crédito via Stripe ou Mercado Pago
- Uso de saldo de crédito da conta

**Pedidos:**
- Status em tempo real
- Tracking de envio
- Solicitação de troca/devolução
- Nota fiscal (link para PDF)

---

### Rendering Strategy (Next.js)

```typescript
// Página de produto — ISR com revalidação a cada 5 minutos
export const revalidate = 300

// Catálogo — SSR para filtros dinâmicos e SEO
// (getServerSideProps equivalente no App Router)

// Home — SSG + ISR para banners e destaques
export const revalidate = 3600

// Área do cliente — Client-side com React Query
// (dados sensíveis e personalizados)
```

---

## API Laravel — Estrutura

### Organização de Rotas

```php
// routes/api.php

// ERP — autenticação via Sanctum, middleware 'erp'
Route::prefix('erp')->middleware(['auth:sanctum', 'role:erp'])->group(function () {

    Route::apiResource('produtos',    ERP\ProdutoController::class);
    Route::apiResource('vendas',      ERP\VendaController::class);
    Route::apiResource('clientes',    ERP\ClienteController::class);
    Route::apiResource('consignacoes',ERP\ConsignacaoController::class);
    Route::apiResource('usuarios',    ERP\UsuarioController::class);

    Route::prefix('financeiro')->group(function () {
        Route::get('caixa/atual',           FinanceiroController::class . '@caixaAtual');
        Route::post('caixa/abrir',          FinanceiroController::class . '@abrirCaixa');
        Route::post('caixa/fechar',         FinanceiroController::class . '@fecharCaixa');
        Route::get('fluxo',                 FinanceiroController::class . '@fluxo');
        Route::apiResource('lancamentos',   LancamentoController::class);
    });

    Route::prefix('relatorios')->group(function () {
        Route::get('dashboard',     RelatorioController::class . '@dashboard');
        Route::get('vendas',        RelatorioController::class . '@vendas');
        Route::get('estoque',       RelatorioController::class . '@estoque');
        Route::get('consignados',   RelatorioController::class . '@consignados');
    });
});

// Loja — pública e autenticada via Passport
Route::prefix('loja')->group(function () {

    // Públicas
    Route::get('produtos',              Loja\CatalogoController::class . '@index');
    Route::get('produtos/{slug}',       Loja\CatalogoController::class . '@show');
    Route::get('categorias',            Loja\CategoriaController::class . '@index');

    // Autenticadas
    Route::middleware('auth:api')->group(function () {
        Route::apiResource('pedidos',       Loja\PedidoController::class);
        Route::apiResource('enderecos',     Loja\EnderecoController::class);
        Route::get('favoritos',             Loja\FavoritoController::class . '@index');
        Route::post('favoritos/{produto}',  Loja\FavoritoController::class . '@toggle');
        Route::get('conta/credito',         Loja\CreditoController::class . '@saldo');
    });

    // Checkout
    Route::post('checkout/calcular-frete', Loja\CheckoutController::class . '@calcularFrete');
    Route::post('checkout/finalizar',      Loja\CheckoutController::class . '@finalizar');
    Route::post('checkout/pix/gerar',      Loja\PagamentoController::class . '@gerarPix');
    Route::post('checkout/webhook',        Loja\PagamentoController::class . '@webhook');
});
```

---

### Cache Strategy com Redis

```php
// Exemplo: listagem de produtos da loja com cache em tags

class CatalogoController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        $filtros = $request->only(['categoria', 'tamanho', 'cor', 'preco_min', 'preco_max', 'genero', 'q']);
        $cacheKey = 'loja:produtos:' . md5(serialize($filtros)) . ':page:' . $request->page;

        $produtos = Cache::tags(['loja', 'produtos'])
            ->remember($cacheKey, now()->addMinutes(10), function () use ($filtros, $request) {
                return Produto::query()
                    ->publicado()
                    ->withFiltros($filtros)
                    ->ordenado($request->sort)
                    ->paginate(24);
            });

        return ProdutoResource::collection($produtos)->response();
    }
}

// Invalidação automática via Observer ao atualizar produto
class ProdutoObserver
{
    public function updated(Produto $produto): void
    {
        Cache::tags(['produtos'])->flush();

        if ($produto->wasChanged('publicado_loja')) {
            Cache::tags(['loja'])->flush();
        }
    }
}
```

---

### Jobs e Filas

```php
// Jobs processados em background via Horizon

// Ao finalizar venda na loja
ProcessarPagamentoJob::class          // confirmar pagamento, baixar estoque
EnviarEmailConfirmacaoPedidoJob::class
GerarEtiquetaEnvioJob::class
AtualizarEstatiticasClienteJob::class

// Rotinas agendadas (Scheduler)
// Todos os dias às 03:00
AtualizarStatusPecasParadasJob::class      // peças sem movimento há X dias
GerarRelatorioFechamentoDiarioJob::class
SincronizarStatusEnviosJob::class          // consulta Correios/Melhor Envio

// Toda semana
GerarRelatorioAcertoConsignadosJob::class
EnviarCampanhaClientesAniversarianteJob::class
```

---

## Estrutura de Pastas

### Laravel (Back-end)

```
app/
├── Console/
│   └── Commands/
├── Domain/                         # Domain-Driven Design por contexto
│   ├── ERP/
│   │   ├── Estoque/
│   │   │   ├── Models/Produto.php
│   │   │   ├── Actions/CadastrarProdutoAction.php
│   │   │   ├── DTOs/ProdutoDTO.php
│   │   │   └── Observers/ProdutoObserver.php
│   │   ├── Vendas/
│   │   ├── Financeiro/
│   │   ├── Clientes/
│   │   └── Consignacao/
│   └── Loja/
│       ├── Catalogo/
│       ├── Pedidos/
│       ├── Checkout/
│       └── Pagamentos/
├── Http/
│   ├── Controllers/
│   │   ├── ERP/
│   │   └── Loja/
│   ├── Middleware/
│   ├── Requests/
│   └── Resources/
├── Jobs/
├── Events/
├── Listeners/
├── Notifications/
└── Services/
    ├── Frete/
    ├── Pagamento/
    └── Etiqueta/
```

### Next.js ERP (Front-end interno)

```
apps/erp/
├── app/
│   ├── (auth)/
│   │   └── login/
│   ├── (erp)/
│   │   ├── layout.tsx          # Shell com sidebar
│   │   ├── dashboard/
│   │   ├── estoque/
│   │   │   ├── page.tsx        # Listagem
│   │   │   ├── novo/
│   │   │   └── [id]/
│   │   ├── vendas/
│   │   │   ├── pdv/            # Interface de caixa
│   │   │   └── historico/
│   │   ├── clientes/
│   │   ├── financeiro/
│   │   ├── consignacao/
│   │   └── relatorios/
├── components/
│   ├── ui/                     # Shadcn components
│   ├── erp/                    # Componentes específicos do ERP
│   │   ├── pdv/
│   │   ├── estoque/
│   │   └── relatorios/
├── hooks/
├── lib/
│   ├── api.ts                  # Cliente HTTP configurado
│   └── auth.ts
└── stores/                     # Zustand stores
    ├── pdv.store.ts
    └── auth.store.ts
```

### Next.js Loja (Front-end público)

```
apps/loja/
├── app/
│   ├── layout.tsx              # Header, footer globais
│   ├── page.tsx                # Home
│   ├── colecao/
│   │   ├── page.tsx
│   │   └── [categoria]/
│   ├── produto/
│   │   └── [slug]/
│   ├── carrinho/
│   ├── checkout/
│   └── conta/
│       ├── pedidos/
│       ├── favoritos/
│       └── credito/
├── components/
│   ├── catalogo/
│   ├── produto/
│   ├── carrinho/
│   ├── checkout/
│   └── layout/
├── hooks/
│   ├── use-carrinho.ts         # Estado do carrinho (Zustand + persistência)
│   ├── use-catalogo.ts
│   └── use-auth.ts
└── lib/
    ├── api.ts
    └── frete.ts
```

---

## Segurança

```
ERP:
  - Autenticação via Sanctum (tokens por dispositivo)
  - 2FA opcional para admin e gerente
  - Rate limiting por endpoint
  - Auditoria de ações (log de quem fez o quê e quando)
  - Sessões com expiração configurável

Loja:
  - Autenticação via Passport (OAuth2)
  - Tokens de refresh com rotação
  - CSRF protection
  - Headers de segurança via middleware
  - Dados de cartão nunca armazenados (tokenização via gateway)

Banco:
  - Row Level Security no PostgreSQL
  - Schemas separados por domínio
  - Backups automáticos diários com retenção de 30 dias
  - Conexões via SSL obrigatório
```

---

## Deploy e Infraestrutura

```yaml
# Exemplo de compose para desenvolvimento local

services:
  api:
    build: ./backend
    environment:
      DB_CONNECTION: pgsql
      DB_HOST: postgres
      REDIS_HOST: redis
      QUEUE_CONNECTION: redis
      CACHE_DRIVER: redis
      SESSION_DRIVER: redis

  erp:
    build: ./frontend/erp
    environment:
      NEXT_PUBLIC_API_URL: http://api/api

  loja:
    build: ./frontend/loja
    environment:
      NEXT_PUBLIC_API_URL: http://api/api

  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: brecho
      POSTGRES_USER: brecho
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

  horizon:
    build: ./backend
    command: php artisan horizon

volumes:
  pgdata:
  redisdata:
```

**Produção sugerida:**
- VPS ou cloud (DigitalOcean, AWS, Hetzner)
- Nginx como reverse proxy
- PM2 ou Docker para Next.js
- Laravel Octane (Swoole) para performance da API
- Backups automatizados no S3
- CI/CD via GitHub Actions

---

## Roadmap Sugerido

```
Fase 1 — Core ERP (2 meses)
  Autenticação e permissões
  Módulo de estoque
  PDV básico (dinheiro e PIX)
  Dashboard inicial

Fase 2 — ERP Completo (2 meses)
  Módulo financeiro
  Módulo de clientes
  Consignação
  Relatórios e exportações
  Etiquetas e código de barras

Fase 3 — Loja Virtual (2 meses)
  Catálogo público
  Carrinho e checkout
  Integração pagamentos (Stripe/MercadoPago)
  Integração frete
  Área do cliente

Fase 4 — Otimizações (1 mês)
  Performance e cache
  SEO da loja
  Notificações push
  App mobile (Progressive Web App)
```

---

> Este documento é um concept técnico e serve como ponto de partida para o planejamento detalhado do sistema. Cada módulo deve ser destrinchado em histórias de usuário, critérios de aceite e especificações de interface antes do desenvolvimento.