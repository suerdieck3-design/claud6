---
title: Mi Biblioteca — área de membros construída e revisada
data: 2026-09-28
branch: miembros-v1 (merge de miembros/db + miembros/app + 3 commits de integração, sem push)
status: código pronto e verificado · migrations NÃO aplicadas · função NÃO publicada · NADA publicado
---

# MIEMBROS-RELATORIO — Mi Biblioteca · 28/09/2026

> **O que muda para a compradora:** quem compra *qualquer* produto Dios y Yo ganha login e vê o que
> comprou (PDF abre, áudio toca com progresso), o que não comprou (com botão de compra) e a
> Coordinadora Pro com cadeado. A assinante continua no painel de sempre, com um bloco a mais.

---

## 0. Onde está o trabalho

| | |
|---|---|
| Branch | `miembros-v1`, saído de `b91b77e` (a spec) |
| Commits | 14 (A: 5 · B: 5 · merges: 2 · integração: 2) + este relatório · **nenhum push** |
| Spec | `docs/MIEMBROS-SPEC.md` — o contrato entre os agentes; seção 4 item 4 revisada aqui |
| Capturas | `docs/capturas/login-390.png`, `biblioteca-390.png`, `escuchar-390.png` (390×844) |

```
4c3c0db  Mi Biblioteca: modelo de dados, RLS, funções de acesso e seed do catálogo
d4fd50f  Webhook: módulo puro de eventos -> accesos com tabela de testes
7f89e55  Webhook: produto avulso concede/revoga acesso na biblioteca
7646bc0  Biblioteca: título, ordem e tipo a partir do nome do arquivo
3a0f467  Scripts de operação: subir-archivos e otorgar-acceso
3b35a2f  Supabase types: tabelas e funções de Mi Biblioteca
48e7937  Mi Biblioteca: camada de dados, acesso e hooks
dbd66ef  Mi Biblioteca: cards, linha de arquivo e player de áudio
a8a0774  Mi Biblioteca: páginas e rotas
511b92c  Sidebar, painel e plano passam a falar de Mi Biblioteca
32d4957  Merge branch 'miembros/db' into miembros-v1
3b6cd01  Merge branch 'miembros/app' into miembros-v1
2593f3f  Webhook: rota por catálogo antes de criar conta; produto desconhecido segue a assinatura
819885d  Seed: reexecutar não apaga o checkout_url preenchido pelo painel
```

---

## 1. O que foi construído

### 1.1 Por rota (todas dentro de `/dashboard`, protegidas por `DashboardLayout`)

| rota | arquivo | o que faz |
|---|---|---|
| `/dashboard` | `src/pages/Dashboard.tsx` | assinante: painel de sempre + bloco "Tu biblioteca" (3 últimos). **Sem assinatura: `Navigate` para `/dashboard/biblioteca`** (nunca volta: a biblioteca não redireciona). |
| `/dashboard/biblioteca` | `src/pages/biblioteca/MiBiblioteca.tsx` | "Tus productos" (o que tem, bundle resolvido no cliente) + card da Coordinadora Pro (`Tuyo` ou cadeado + "Ver el plan") + "También para ti" (o que não tem; botão "Conseguir" só com `checkout_url`) |
| `/dashboard/biblioteca/:slug` | `src/pages/biblioteca/ProductoDetalle.tsx` | arquivos do produto (PDF/imagem abre em aba nova com URL assinada; áudio vai ao player); bundle mostra "Lo que trae esta colección"; sem acesso, cadeado e card de compra; `tipo='app'` mostra o ProCard |
| `/dashboard/escuchar/:archivoId` | `src/pages/biblioteca/Escuchar.tsx` | player em tela cheia: play/pause 72px, ±15 s 56px, barra, velocidade 1/1,25/1,5×, anterior/siguiente dentro do produto; salva `progreso` a cada 10 s, ao pausar, ao sair e ao terminar (e passa ao próximo) |

Sidebar (`src/components/AppSidebar.tsx`): grupo "Mi biblioteca" no topo; "El encuentro" e "Mi grupo"
ganham `Lock` e levam a `/dashboard/planos` para quem não assina (7 itens apontam para o plano nesse
caso, contando "Mi plan"). `Planos.tsx`: o plano gratuito diz "Tu biblioteca: todo lo que compraste".

Camada de dados: `src/lib/biblioteca/{tipos,api,acceso,formato,archivos}.ts`, hooks
`src/hooks/useBiblioteca.tsx` (TanStack Query) e `src/hooks/useAudioPlayer.ts` (`<audio>` nativo,
sem lib). Componentes: `src/components/biblioteca/{ProductoCard,ProCard,ArchivoRow,AudioPlayer}.tsx`.
`src/integrations/supabase/types.ts` ganhou as 5 tabelas e 4 funções à mão, no formato gerado.

### 1.2 Por tabela (`supabase/migrations/20260928120000_mi_biblioteca.sql`)

| tabela | leitura (`authenticated`) | escrita |
|---|---|---|
| `productos` | `activo` | service role |
| `producto_incluye` | tudo | service role |
| `producto_archivos` | `biblioteca_tiene_acceso(producto_id)` | service role |
| `accesos` | dona | service role (`biblioteca_otorgar` / `biblioteca_revocar`) |
| `progreso` | dona | dona — insert/update/delete com `with check`; `updated_at` vem de trigger |

Além das policies, `revoke all ... from anon, authenticated` + `grant select` por tabela: mesmo uma
policy permissiva por engano não libera escrita. Funções `security definer` com `search_path`
fixo; `biblioteca_otorgar`/`revocar` levantam exceção se `current_user` é `authenticated`/`anon`.
Bucket `biblioteca` privado; policy `select` em `storage.objects` por
`biblioteca_tiene_acceso_slug((storage.foldername(name))[1])` — `createSignedUrl` passa por ela.

Seed (`20260928120100_seed_productos.sql`): 12 produtos + `proverbios`, 3 linhas do bundle.
`tipo` bate com os literais de `tipos.ts`; slugs batem com o que o frontend usa (`coordinadora-pro`
só é resolvido por `tipo='app'`, nunca por slug fixo); `checkout_url` só para `coordinadora-pro`
(o mesmo `CHECKOUT_URL` de `config/precios.ts`) e `dinamicas-ninos`.

### 1.3 Webhook e scripts

`supabase/functions/hotmart-webhook/index.ts` decide a rota por produto (`rutaWebhook`, puro, em
`_shared/accesos.ts`): assinatura → `profiles` (lógica original intacta); produto do catálogo →
`accesos` via rpc. Para produto avulso só três eventos importam: `PURCHASE_APPROVED`/`COMPLETE`
concedem (e criam a conta), `PURCHASE_REFUNDED`/`CHARGEBACK` revogam, o resto ignora.
Scripts (Node 22, sem build): `scripts/subir-archivos.ts --dir <pasta> --check|--push` e
`scripts/otorgar-acceso.ts --email x --producto <slug>`; ambos testados aqui em modo seco.

---

## 2. Revisão adversarial: o que foi encontrado

### 2.1 Corrigido (2 bugs reais, 1 commit cada)

**B1 · Webhook criava a conta antes de saber se o produto existia, e roteava só por id fixo**
`supabase/functions/hotmart-webhook/index.ts:171` e `:227` (antes da correção).
Uma `PURCHASE_APPROVED` de produto fora do catálogo chamava `ensureUser` (conta + e-mail de senha)
e só depois o rpc devolvia `0`. Pior: `PRODUCTOS_SUSCRIPCION` só tem a Coordinadora Pro
(`7635451`), mas a spec diz que o grupo de webhook da Hotmart tem **dois** produtos de assinatura —
o segundo, que não está no seed, cairia no caminho da biblioteca e a assinante **nunca seria
ativada em `profiles`**. Correção (`2593f3f`): `rutaWebhook(productId, enCatalogo)` decide antes de
qualquer `ensureUser` — sem id ou id de assinatura → `profiles`; no catálogo com `tipo='app'` →
`profiles` (como a spec pedia); outro tipo → `accesos`; **fora do catálogo → `profiles`, com
`warn`** — que é exatamente o que sempre aconteceu com qualquer produto do grupo. Uma leitura
indexada em `productos` por evento; erro de leitura vira 500 (a Hotmart reenvia). 6 testes novos
em `src/test/biblioteca-accesos.test.ts`; spec seção 4 item 4 atualizada.

**B2 · Reexecutar o seed apagava o `checkout_url` preenchido pelo painel**
`supabase/migrations/20260928120100_seed_productos.sql:80`. O `on conflict do update` gravava o
`NULL` do seed por cima do que o dono digitou (checklist, passo 5). Correção (`819885d`):
`coalesce(excluded.checkout_url, productos.checkout_url)`.

### 2.2 Verificado e correto (sem mudança)

- **Contrato A↔B**: toda query de `src/lib/biblioteca/api.ts` bate com coluna e tipo da migration;
  `guardarProgreso` (upsert) tem policy insert+update com `with check`; `contarArchivos` e
  `listarArchivos` passam pela RLS de `producto_archivos`; `urlFirmada` passa pela policy do storage;
  nomes, args e retornos das 4 rpc em `types.ts` batem com o SQL.
- **Webhook**: o caminho de assinatura é o original (`mapHotmartEvent` → `profiles`,
  `createUserIfMissing && isPurchaseEvent`); `SUBSCRIPTION_CANCELLATION` com id de kit → `ignore`,
  não revoga; `productId` vazio → caminho antigo; revogação usa `ensureUser(..., false)` — nunca cria
  conta; nenhum `JSON.stringify(body)`, e-mail sempre por `redactEmail`; 401 hottok, 503 sem segredo.
- **Frontend**: hooks todos antes de qualquer `return`; `useCredits` só é usado dentro de
  `DashboardLayout` (as 3 rotas são filhas dele); `AudioPlayer` sem autoplay ao montar (só `play()`
  ao vir de "terminou o anterior", e engole a rejeição); `window.open` síncrono no gesto e só depois
  o `await` da URL assinada (iOS); espanhol com tuteo, zero "usted"; nenhum preço fora de
  `config/precios.ts`; alvos: Conseguir 48px, linhas 56px, play 72px, ±15 s 56px, "Ver todo" 48px.

### 2.3 Observações que ficaram como estão

- `findUser` do webhook agora percorre todas as páginas de `listUsers` (o código antigo olhava só
  a primeira, de 50). É melhoria, não mudança de regra — mas é uma diferença de comportamento.
- `types.ts` tipa `p_transaction: string` (o SQL aceita `null`); os scripts contornam com `as never`,
  a mesma convenção do gerador. Os scripts não entram no `tsc` do app.
- `guardarProgreso` manda `updated_at`; o trigger sobrescreve. Inofensivo.
- `Dashboard` mostra o painel Pro por um instante enquanto `useCredits` carrega, antes do redirect.
- Pré-existentes, fora do escopo: avisos do React Router v7 (future flags) no console; a `Sheet`
  do sidebar shadcn sem `DialogTitle` (aviso de acessibilidade ao abrir o menu no celular); o
  logotipo do login tem uma cruz (regra 6 da spec vale para UI *nova*).
- `data.product.name` do payload vai ao log sem truncar (não é dado pessoal; hottok já validou).

---

## 3. Resultado real dos comandos (depois das correções)

```
$ npm test
 Test Files  13 passed | 1 skipped (14)
      Tests  326 passed | 6 skipped (332)          # baseline do branch: 320 · +6 de rutaWebhook

$ npx tsc --noEmit -p tsconfig.app.json
(sem saída) · exit 0

$ npm run build
(!) Some chunks are larger than 500 kB after minification.   # aviso pré-existente
✓ built in 5.44s
```

Sem `deno` na máquina: os dois arquivos da Edge Function foram checados só sintaticamente
(`typescript.transpileModule`: ok). O type-check real acontece no `supabase functions deploy`.

---

## 4. Smoke no navegador (Playwright · Chromium · 390×844)

Dev server com `.env` de chave falsa (não commitado).

**Sem sessão:** `/` e `/login` renderizam; `/dashboard`, `/dashboard/biblioteca`,
`/dashboard/biblioteca/salmos-audio` e `/dashboard/escuchar/x` **redirecionam para `/login`**.
Console: só erros de rede para o Supabase falso e os avisos do React Router. Nenhum erro de runtime.

**Com sessão falsa e Supabase mockado na rede** (perfil, catálogo, acessos, arquivos, URL assinada
devolvendo um WAV real de 1 s) — 26 verificações, todas ok:

- compradora sem assinatura: `/dashboard` → `/dashboard/biblioteca`; "Tuyo" no que tem; bundle
  listado e os filhos não aparecem como faltantes; ProCard com cadeado; "También para ti" com
  "Conseguir"; 7 itens do sidebar apontam para o plano; detalhe de áudio com "Empezar a escuchar" e
  2 arquivos; bundle mostra o que inclui sem o vazio falso; produto sem acesso mostra cadeado e não
  mostra "Conseguir" sem `checkout_url`; slug e arquivo inexistentes têm tela própria;
- player: metadata carrega, título/“1 de 2”/Siguiente, play de 72px; ao terminar grava
  `progreso` (1 upsert, `completado=true`, `user_id` correto) e passa para o próximo;
- assinante: fica em `/dashboard`, vê "Tu biblioteca"; na biblioteca o ProCard é "Tuyo".

---

## 5. O que ficou de fora / não deu para verificar

- Nada foi aplicado no projeto `lpgjazzkwcpdytvyqpgj`: migration, RLS e policy de storage não
  rodaram contra Postgres real. A policy em `storage.objects` é o padrão Supabase, mas só o
  `db push` confirma.
- O id do **segundo produto de assinatura** da Hotmart não está no repositório. Até ele entrar em
  `PRODUCTOS_SUSCRIPCION` (ou em `productos` com `tipo='app'`), ele funciona pelo fallback com `warn`.
- Fora do escopo por spec: devocional do dia, visualizador de PDF embutido, domínio/Vercel,
  e-mail de boas-vindas próprio.
- iOS Safari real (o smoke foi Chromium com emulação de toque).

---

## 6. Checklist de publicação (dono)

Pré-requisito: Supabase CLI logada e projeto linkado uma vez —
`supabase link --project-ref lpgjazzkwcpdytvyqpgj`.

1. **Aplicar as 2 migrations**
   `supabase db push`
   Conferir: no SQL Editor, `select slug, hotmart_product_id, tipo, checkout_url from productos order by orden;`
   (13 linhas) e `select * from storage.buckets where id = 'biblioteca';` (`public = false`).
2. **Publicar o webhook**
   `supabase functions deploy hotmart-webhook`
   Os segredos já existentes continuam valendo (`HOTMART_WEBHOOK_TOKEN`, `PASSWORD_RESET_REDIRECT_URL`).
3. **Hotmart (manual):** Ferramentas → Webhook → grupo "App Coordinadora Pro" → adicionar os 11
   produtos da seção 2 da spec (todos menos `proverbios`). Eventos mínimos: Compra aprovada, Compra
   completa, Reembolso, Chargeback. Se houver um segundo produto de assinatura no grupo, anotar o
   id dele e acrescentar em `supabase/functions/_shared/accesos.ts` → `PRODUCTOS_SUSCRIPCION`
   (ou inserir em `productos` com `tipo = 'app'`), e publicar de novo (passo 2).
4. **Subir os arquivos** (pasta local `<pasta>/<slug>/<archivo>`; ex.: `<pasta>/salmos-audio/01-salmo-23.mp3`)
   `node scripts/subir-archivos.ts --dir <pasta> --check` (só lista, sem credenciais)
   `SUPABASE_URL=https://lpgjazzkwcpdytvyqpgj.supabase.co SUPABASE_SERVICE_ROLE_KEY=<chave> node scripts/subir-archivos.ts --dir <pasta> --push`
   A chave só na sessão do terminal, nunca gravada. Reexecutar não duplica nada.
5. **`checkout_url`** dos produtos que devem aparecer em "También para ti": Table Editor →
   `productos` → coluna `checkout_url` (ou `update productos set checkout_url = '...' where slug = '...'`).
   Vazio = sem botão de compra.
6. **Compradoras antigas** (quem comprou antes do webhook):
   `SUPABASE_URL=... SUPABASE_SERVICE_ROLE_KEY=... SUPABASE_ANON_KEY=<publishable> node scripts/otorgar-acceso.ts --email ana@example.com --producto kit-encuentro`
   Cria a conta se não existir e manda o e-mail de senha (precisa da `SUPABASE_ANON_KEY`).
7. **Testar com uma compra de teste** (Hotmart → produto do kit → "compra de teste"/sandbox, com um
   e-mail novo): em Supabase → Edge Functions → `hotmart-webhook` → Logs deve aparecer
   `Acesso concedido: user=... product=7294813`; chega o e-mail de definir senha; ao entrar, cai em
   `/dashboard/biblioteca` com o kit em "Tus productos"; um PDF abre em aba nova. Depois, reembolsar a
   compra de teste: o log mostra `Acesso revogado (reembolsado)` e o arquivo deixa de abrir.
   Se o log disser `fora do catalogo`, o id do produto na Hotmart não bate com o seed.
8. **Publicar o app:** merge de `miembros-v1` em `main` e push (a Lovable publica a partir daí).
