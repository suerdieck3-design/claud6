# Mi Biblioteca — área de membros própria do Dios y Yo

**Data:** 28/09/2026 · **Repo:** `suerdieck3-design/ldercristopro` · **Supabase:** `lpgjazzkwcpdytvyqpgj`
**Branch de integração:** `miembros-v1`

---

## 0. A virada

Hoje o app só existe para quem assina a Coordinadora Pro (US$ 27,90/mês). Quem compra o kit de
230 dinámicas (US$ ~12), os 31 Salmos en Audio, os Pasatiempos ou a Biblioteca Devocional recebe
**um e-mail da Hotmart com link de download** e nunca mais fala com a marca.

**Mi Biblioteca** muda isso: **toda compradora de qualquer produto Dios y Yo ganha login**. Dentro
ela vê o que comprou (PDFs para baixar/ler, áudios para ouvir com progresso), o que **não** comprou
(com link de checkout), e a Coordinadora Pro aparece como o andar de cima — bloqueado com cadeado,
não escondido.

Resultado: um lugar só, base de upsell entre produtos, dependência menor da Hotmart, e a assinatura
vendida para quem já está logada (a conversão mais barata que existe).

## 1. Regras que não caem

1. **A conta continua nascendo só por compra** (webhook). Não existe cadastro público.
2. **Autorização no servidor.** RLS decide o que ela vê; o cliente é conveniência.
3. **Arquivos em bucket privado.** Nunca URL pública. Storage RLS por `accesos`.
4. **Nenhuma migration já aplicada é editada.** Só arquivos novos.
5. **Espanhol neutro LatAm com tuteo** em toda interface nova. Sem "usted", sem "vos".
6. **Design system existente.** Nada de componente de UI novo — só shadcn que já está em `src/components/ui`.
   Alvos de toque ≥ 48px, corpo 18px, sem cruz/templo (interdenominacional).
7. **Tudo que já funciona continua funcionando**: Coordinadora Pro (catálogo, grupo, recomendador,
   conducción, adaptador) não muda de comportamento para assinante.

## 2. Produtos Dios y Yo (fonte: e-mails "produto aprovado" da Hotmart)

| hotmart_product_id | slug | nome | tipo | conteúdo |
|---|---|---|---|---|
| `7294813` | `kit-encuentro` | Kit Encuentro con Dios — 230 Dinámicas | `pdf` | 12 cuadernos PDF |
| `7635451` | `coordinadora-pro` | Coordinadora Pro | `app` | assinatura (já existe via `profiles`) |
| `7329805` | `dinamicas-ninos` | 10 Dinámicas Bíblicas para Niños | `pdf` | 1 PDF |
| `7353157` | `salmos-explicados` | 150 Salmos Explicados Versículo por Versículo | `pdf` | PDF |
| `8554501` | `salmos-audio` | 31 Salmos en Audio | `audio` | 31 mp3 |
| `8574800` | `pasatiempos` | Pasatiempos Bíblicos — 120 sopas de letras y crucigramas | `pdf` | PDF |
| `8574965` | `biblioteca-devocional` | Biblioteca Devocional: Salmos, Proverbios y Pasatiempos | `bundle` | inclui `salmos-explicados` + `pasatiempos` + `proverbios` |
| `7419129` | `dinamicas-catequistas` | +100 Dinámicas Bíblicas para Catequistas | `pdf` | PDF (vendido; fora do funil) |
| `7330131` | `dinamicas-juveniles` | +100 Dinámicas para Ministerios Juveniles | `pdf` | PDF |
| `7331224` | `rompehielos` | 60 Dinámicas y Rompehielos Bíblicos | `pdf` | PDF |
| `7495770` | `kit-encontro-pt` | Kit Encontro com Deus (PT) | `pdf` | 12 cadernos PT |

`proverbios` (sem hotmart id próprio, só dentro do bundle): slug `proverbios`, tipo `pdf`, `hotmart_product_id = null`.

Produtos fora da marca (insulina, postres, huevo de pascua) **não entram**.

**IDs de produto são `text`** (a Hotmart manda número no payload; normalizamos com `String()`).

## 3. Modelo de dados (migration nova `20260928120000_mi_biblioteca.sql`)

```sql
-- catálogo de produtos vendáveis (leitura pública para logadas: alimenta o "también para ti")
productos(
  id uuid pk, slug text unique, hotmart_product_id text unique null,
  nombre text, descripcion text null, tipo text check in ('pdf','audio','app','bundle'),
  portada_url text null, checkout_url text null, orden int default 100, activo bool default true,
  created_at timestamptz
)

-- o que um bundle inclui
producto_incluye(bundle_id uuid fk productos, producto_id uuid fk productos, pk(bundle_id, producto_id))

-- arquivos de cada produto, no bucket privado `biblioteca`, path `<slug>/<archivo>`
producto_archivos(
  id uuid pk, producto_id uuid fk productos on delete cascade,
  titulo text, tipo text check in ('pdf','audio','imagen'),
  storage_path text unique,            -- ex: 'salmos-audio/01-salmo-23.mp3'
  orden int default 100, duracion_seg int null, tamano_bytes bigint null,
  created_at timestamptz
)

-- direito de acesso (a linha que o webhook escreve)
accesos(
  id uuid pk, user_id uuid fk auth.users on delete cascade, producto_id uuid fk productos,
  origen text check in ('hotmart','admin') default 'hotmart',
  hotmart_transaction text null,
  estado text check in ('activo','reembolsado','chargeback') default 'activo',
  otorgado_en timestamptz default now(), revocado_en timestamptz null,
  unique(user_id, producto_id)
)

-- progresso de consumo (áudio ouvido, pdf aberto)
progreso(
  user_id uuid fk auth.users, archivo_id uuid fk producto_archivos on delete cascade,
  posicion_seg int default 0, completado bool default false, updated_at timestamptz,
  pk(user_id, archivo_id)
)
```

### Funções (security definer, `set search_path = public`)

- `biblioteca_tiene_acceso(p_producto_id uuid) returns boolean` — true se existe `accesos` ativo
  do `auth.uid()` para o produto **ou** para um bundle que o inclui.
- `biblioteca_tiene_acceso_slug(p_slug text) returns boolean` — idem por slug (é o que o storage usa).
- `biblioteca_otorgar(p_user_id uuid, p_hotmart_product_id text, p_transaction text) returns int` —
  só service role; upsert em `accesos` (reativa se estava revogado); devolve 1 se conhecido, 0 se o
  produto não existe. **Não** expande bundle em linhas: o acesso ao filho é derivado na leitura.
- `biblioteca_revocar(p_user_id uuid, p_hotmart_product_id text, p_estado text) returns int` — só service role.

### RLS

| tabela | select | insert/update/delete |
|---|---|---|
| `productos` | `authenticated` onde `activo` | só service role |
| `producto_incluye` | `authenticated` | só service role |
| `producto_archivos` | `authenticated` e `biblioteca_tiene_acceso(producto_id)` | só service role |
| `accesos` | dona (`auth.uid() = user_id`) | só service role |
| `progreso` | dona | dona (insert/update/delete com `with check`) |

### Storage

- Bucket **`biblioteca`**, `public = false`, criado na migration via `insert into storage.buckets`.
- Policy em `storage.objects` (select) para `authenticated`:
  `bucket_id = 'biblioteca' and public.biblioteca_tiene_acceso_slug((storage.foldername(name))[1])`.
- Sem policy de insert/update/delete: só service role escreve (script de upload).
- O cliente obtém o arquivo com `supabase.storage.from('biblioteca').createSignedUrl(path, 3600)` —
  a assinatura passa pela RLS acima, então sem acesso a chamada falha.

### Seed (`20260928120100_seed_productos.sql`)

Insere os 12 produtos da seção 2 + `proverbios`, e as 3 linhas de `producto_incluye` do bundle.
`on conflict (slug) do update` (reexecutável). `checkout_url` fica **NULL** para tudo, exceto:
- `coordinadora-pro`: `https://pay.hotmart.com/I105576629Y?off=ncyr7jn7&checkoutMode=6`
- `dinamicas-ninos`: `https://pay.hotmart.com/R104765961X?off=1gvg6qki`
(os outros links o Antônio preenche pelo painel; a UI esconde o botão de compra quando é NULL).

## 4. Webhook (`supabase/functions/hotmart-webhook`)

Hoje: só produtos de assinatura, escreve em `profiles`. Passa a:

1. Ler `data.product.id` → `String()` → `productId`. `data.product.name` só para log.
2. **Assinatura** (productId `7635451` ou qualquer `productos.tipo = 'app'`): mantém **exatamente** a
   lógica atual (`mapHotmartEvent` → `profiles`). Nada muda para ela.
3. **Qualquer outro produto**: novo módulo puro `_shared/accesos.ts`:
   ```ts
   export type AccesoAction =
     | { kind: "otorgar"; createUserIfMissing: true }
     | { kind: "revocar"; estado: "reembolsado" | "chargeback" }
     | { kind: "ignore" };
   export function mapAccesoEvent(event: unknown): AccesoAction
   ```
   - `PURCHASE_APPROVED`, `PURCHASE_COMPLETE` → `otorgar` (cria usuário se não existir, mesmo
     fluxo de senha aleatória + e-mail de definir senha que já existe).
   - `PURCHASE_REFUNDED` → `revocar('reembolsado')`; `PURCHASE_CHARGEBACK` → `revocar('chargeback')`.
   - Todo o resto (`PURCHASE_CANCELED`, `PURCHASE_DELAYED`, `PURCHASE_BILLET_PRINTED`,
     `SWITCH_PLAN`, `PURCHASE_OUT_OF_SHOPPING_CART`…) → `ignore`. Produto avulso não tem
     "atraso": ou pagou ou não pagou.
4. Chama `biblioteca_otorgar` / `biblioteca_revocar` via `supabase.rpc(...)` com service role.
   **Produto fora do catálogo** (`productos` não tem esse `hotmart_product_id`): a rota se decide
   *antes* de criar conta (`rutaWebhook`, puro) e o evento segue o **caminho de assinatura**, com
   log `warn` — é o que sempre aconteceu para qualquer produto do grupo de webhook, e protege a
   assinante do segundo produto de assinatura da Hotmart, que não está no seed. (Revisado na
   integração, 28/09: a versão anterior deste item respondia `unknown_product`, mas só depois de já
   ter criado a conta.) O `0` do rpc continua tratado como `unknown_product`, por defesa.
5. A resposta para a Hotmart continua `200` em todos os casos tratados; `401` hottok, `503` sem segredo.

Testes (`src/test/biblioteca-accesos.test.ts`): tabela de eventos → ação, produto de assinatura
não passa por `mapAccesoEvent`, `String(7294813)` casa com o seed.

> **Passo manual do dono (não é código):** no painel da Hotmart, adicionar os produtos da seção 2
> ao grupo de webhook "App Coordinadora Pro" (hoje só tem os 2 de assinatura). Sem isso, nenhuma
> compradora de kit ganha conta.

## 5. Frontend

### Rotas novas (dentro de `/dashboard`, protegido por `DashboardLayout`)

| rota | página | o quê |
|---|---|---|
| `/dashboard/biblioteca` | `pages/biblioteca/MiBiblioteca.tsx` | grid do que ela tem + "También para ti" (o que não tem, com botão de compra se `checkout_url`) + card da Coordinadora Pro (bloqueado se não assina) |
| `/dashboard/biblioteca/:slug` | `pages/biblioteca/ProductoDetalle.tsx` | lista de arquivos do produto; PDF → abre em nova aba / baixa; áudio → player |
| `/dashboard/escuchar/:archivoId` | `pages/biblioteca/Escuchar.tsx` | player de áudio em tela cheia (mobile-first): título, play/pause 72px, ±15s, barra, velocidade, próximo/anterior dentro do produto; salva `progreso` a cada 10s e ao pausar |

### Onde ela cai ao entrar

`/dashboard` (`Dashboard.tsx`): **se `isSubscriptionActive`** → painel atual, intacto, **mais** um
bloco compacto "Tu biblioteca" com os 3 últimos produtos. **Se não assina** → redireciona para
`/dashboard/biblioteca` (a compradora do kit não pode cair num painel que só fala do Pro e quebra).

### Sidebar (`AppSidebar.tsx`)

Grupo novo **"Mi biblioteca"** no topo: `Mis productos` → `/dashboard/biblioteca`.
Os grupos "El encuentro" e "Mi grupo" continuam, mas para quem **não** assina cada item mostra um
`Lock` à direita e leva para `/dashboard/planos` (não para a tela bloqueada). Usa `useCredits()`.

### Camada de dados

`src/lib/biblioteca/tipos.ts` (Producto, ProductoArchivo, Acceso, Progreso — espelhando a seção 3),
`src/lib/biblioteca/api.ts` (listarProductos, misAccesos, listarArchivos(slug), urlFirmada(path),
guardarProgreso, listarProgreso), `src/hooks/useBiblioteca.tsx` (TanStack Query, mesmo padrão de
`useKit.tsx`). **Adicionar as 5 tabelas + 4 funções em `src/integrations/supabase/types.ts`** à mão,
no mesmo formato gerado (o CLI não roda aqui). Zero `as any`.

Lógica pura testável em `src/lib/biblioteca/acceso.ts`: `tengoAcceso(producto, accesos, incluye)` —
resolve bundle no cliente para desenhar o cadeado sem ida ao servidor. Teste em
`src/lib/biblioteca/acceso.test.ts`.

### Componentes

`components/biblioteca/ProductoCard.tsx` (capa, nome, "X archivos", selo `Tuyo` ou botão `Conseguir`),
`components/biblioteca/ArchivoRow.tsx` (linha 56px: ícone por tipo, título, duração/tamanho, progresso, ação),
`components/biblioteca/AudioPlayer.tsx` (hook `useAudioPlayer` + UI; `<audio>` nativo, sem lib nova).

### Copy (espanhol, tuteo)

- Título: **Mi biblioteca** · sub: "Todo lo que compraste, en un solo lugar."
- Seção do que não tem: **También para ti**.
- Card Pro bloqueado: "Coordinadora Pro · Tus 230 dinámicas buscables, el recomendador y el modo
  conducción." botão "Ver el plan".
- Vazio (conta sem acesso nenhum): "Todavía no vemos compras en esta cuenta. Si compraste con otro
  correo, escríbenos." (sem link de suporte inventado).

## 6. Scripts de operação (`scripts/`)

- `scripts/subir-archivos.ts` — Node 22 puro, service role. Lê uma pasta local
  `--dir <ruta>/<slug>/*.{pdf,mp3,png}`, sobe para `biblioteca/<slug>/<archivo>` e faz upsert em
  `producto_archivos` (título a partir do nome do arquivo: `01-salmo-23.mp3` → "Salmo 23", ordem 1;
  duração do mp3 não se calcula aqui — fica null e o player descobre no `loadedmetadata`).
  Flags `--check` (só lista) e `--push`. Lógica pura de nome→título em `src/lib/biblioteca/archivos.ts`
  com teste.
- `scripts/otorgar-acceso.ts` — `--email x --producto <slug>`: chama `biblioteca_otorgar` para
  compradoras antigas (quem comprou antes do webhook existir). Cria a conta se não existir, usando o
  mesmo fluxo do webhook (senha aleatória + e-mail).

## 7. Divisão do trabalho

| agente | worktree | branch | entrega |
|---|---|---|---|
| **A · banco + webhook** | `/home/user/lc-db` | `miembros/db` | migration, seed, `_shared/accesos.ts`, webhook, testes, `scripts/*.ts`, `src/lib/biblioteca/archivos.ts` |
| **B · frontend** | `/home/user/lc-app` | `miembros/app` | types.ts, tipos/api/hooks, `acceso.ts`+teste, páginas, componentes, sidebar, dashboard, rotas |
| **C · integração/QA** (depois) | `/home/user/ldercristopro` | `miembros-v1` | merge A+B, build, tsc, vitest, revisão adversarial, `docs/MIEMBROS-RELATORIO.md` |

Contrato entre A e B é **esta spec**. Se um dos dois precisar mudar a seção 3, muda a spec primeiro.

## 8. Fora do escopo desta entrega

- Devocional do dia (cuaderno 07 em JSON) — v1.5, depende do JSON estruturado que está na máquina do Antônio.
- Visualizador de PDF embutido — v1 abre em nova aba (iOS/Android já renderizam).
- Domínio próprio / deploy na Vercel — decisão separada (hoje publica pela Lovable).
- E-mail de boas-vindas próprio — hoje é o e-mail de definir senha do Supabase.

## 9. Checklist de publicação (dono)

1. Aplicar as 2 migrations no `lpgjazzkwcpdytvyqpgj`.
2. `supabase functions deploy hotmart-webhook`.
3. Hotmart → Ferramentas → Webhook → grupo "App Coordinadora Pro" → adicionar os produtos da seção 2.
4. `node scripts/subir-archivos.ts --dir <pasta> --push` com os PDFs e mp3.
5. Preencher `productos.checkout_url` no painel para os produtos que devem aparecer em "También para ti".
6. Testar: compra de teste do kit → e-mail de senha → login → arquivo abre.
7. Publicar.
