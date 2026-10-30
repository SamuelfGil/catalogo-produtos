# Catálogo de Produtos

SPA em React + Vite que lista os produtos da API REST de produtos (Node.js + Express) desenvolvida na atividade anterior, permite pesquisar por nome e consultar os detalhes de um produto, sem recarregar a página.

## Como executar

Pré-requisito: Node.js 20.19+ (testado com Node 24).

### 1. Subir a API de produtos

Na pasta da API:

```bash
npm install
npm start
```

A API sobe em `http://localhost:3000`.

### 2. Subir o catálogo

```bash
git clone https://github.com/SamuelfGil/catalogo-produtos.git
cd catalogo-produtos
npm install
npm run dev
```

Abra o endereço indicado no terminal (normalmente `http://localhost:5173`).

## Configuração da API

O endereço fica no arquivo `.env`, na raiz do projeto:

```env
VITE_API_URL=/api
API_PROXY_TARGET=http://localhost:3000
```

- `VITE_API_URL` é o prefixo usado pelo front em `src/services/productsApi.js`.
- `API_PROXY_TARGET` é o endereço real da API. Para usar outro servidor, troque esse valor.

A API de produtos não envia cabeçalhos CORS, então o navegador bloquearia uma chamada direta de `localhost:5173` para `localhost:3000`. Para não alterar o back-end, o `vite.config.js` configura um proxy: as chamadas para `/api/...` são repassadas ao `API_PROXY_TARGET` sem o prefixo `/api`.

Reinicie o `npm run dev` depois de alterar o `.env`.

### Adaptação ao contrato da API

| Rota usada | Retorno |
|---|---|
| `GET /produtos` | array de produtos |
| `GET /produtos/:id` | um produto (ou 404) |

Cada produto tem `id`, `descricao`, `preco`, `categoria` e `estoque`. A API não tem o campo `nome`: o nome do produto vem em `descricao`. Por isso, o cartão, a pesquisa e o painel de detalhes usam `descricao` como nome, e o painel mostra o `estoque`.

## Árvore de componentes

```
App                      (estados: products, search, loading, error,
│                         selectedId, selectedProduct, detailsLoading, detailsError)
├── Header
├── ProductSearch        (props: value, onSearchChange)
├── ProductList          (props: products, onViewDetails)
│   └── ProductCard      (props: product, onViewDetails) — um para cada produto
└── ProductDetails       (props: product, loading, error, onClose) — só quando há produto selecionado
```

## Fluxo de dados

Todos os estados ficam em `App`, que é o pai comum dos componentes que precisam deles. Ao montar, um `useEffect` busca a lista em `GET /produtos` e guarda em `products`, controlando `loading` e `error`. O `ProductSearch` é um campo controlado: recebe `search` pela prop `value` e, a cada digitação, chama `onSearchChange` (o `setSearch` de `App`). Com o novo `search`, `App` renderiza de novo e calcula `filteredProducts`, sem precisar de um segundo estado. A lista filtrada desce por props para `ProductList`, que usa `map` para criar um `ProductCard` por produto, com `key={product.id}`. No cartão, o botão "Ver detalhes" chama `onViewDetails(product.id)`, que é o `setSelectedId` de `App`. A mudança de `selectedId` dispara outro `useEffect`, que busca `GET /produtos/:id` e passa o resultado para `ProductDetails` por props. O botão "Fechar detalhes" chama `onClose`, que limpa a seleção e esconde o painel. Assim, uma ação iniciada no `ProductCard` sobe por callback até `App` e desce como prop até o `ProductDetails`.

## Testes manuais

Executados em 29/10/2026 com a API local e o Microsoft Edge. Os prints estão em [`docs/evidencias/`](docs/evidencias/).

| # | Teste | Resultado | Evidência |
|---|---|---|---|
| 1 | A aplicação abre sem erros no console | OK, nenhum erro com a API no ar | — |
| 2 | Os produtos são carregados da API | OK, 5 cartões vindos de `GET /produtos` | [01](docs/evidencias/01-lista-da-api.png) |
| 3 | A busca por nome funciona | OK, "mouse" deixa 1 cartão e o contador mostra "1 produto(s) encontrado(s)." | [02](docs/evidencias/02-pesquisa-mouse.png) |
| 4 | A mensagem de lista vazia aparece | OK, "geladeira" mostra "Nenhum produto encontrado." | [03](docs/evidencias/03-pesquisa-sem-resultado.png) |
| 5 | Uma falha na API exibe mensagem de erro | OK, com a API parada aparece "Erro HTTP 502" | [06](docs/evidencias/06-erro-api-fora-do-ar.png) |
| 6 | "Ver detalhes" consulta o produto por ID | OK, requisição `GET /produtos/3` e painel do Monitor 24 polegadas | [04](docs/evidencias/04-detalhes-do-produto.png) |
| 7 | Selecionar outro produto | OK, `GET /produtos/1` troca o painel para o Teclado Mecânico | — |
| 8 | "Fechar detalhes" oculta o painel | OK | [05](docs/evidencias/05-detalhes-fechados.png) |
| 9 | Tudo ocorre sem recarregar a página | OK, uma marca gravada em `window` no início continuou lá até o fim | — |

Com a API parada, o proxy do Vite responde 502 (Bad Gateway), e é esse código que aparece na mensagem de erro. Em desenvolvimento, `GET /produtos` aparece duas vezes na aba de rede: é o Strict Mode montando o efeito duas vezes, e a primeira requisição é cancelada pelo `AbortController`.

## Discussão

- **Onde estão os estados?** Todos em `App`. Os outros componentes só recebem props.
- **Por que a pesquisa é controlada por `App`?** Porque o texto pesquisado é usado por dois irmãos: o `ProductSearch` mostra o valor, e a lista filtrada vai para o `ProductList`. O estado fica no pai comum.
- **Como uma ação em `ProductCard` atualiza `ProductDetails`?** O cartão chama o callback `onViewDetails`, que muda `selectedId` em `App`. O efeito que depende de `selectedId` busca o produto, e o resultado desce como prop para o `ProductDetails`.
- **Props, `useState` e `useEffect`:** props são dados que o pai passa ao filho, somente leitura. `useState` guarda dados do próprio componente que mudam com o uso e provocam nova renderização. `useEffect` sincroniza o componente com algo externo, como a API, depois da renderização.
