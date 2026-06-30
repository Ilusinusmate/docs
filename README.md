# Contrato de API — CDE Mobile (30/06/2026)

> Documento gerado a partir dos datasources do app Flutter.
> Descreve os endpoints que o backend precisa implementar/manter para
> que o app funcione corretamente.
>
> **Base URL:** configurada via variável de ambiente `API_URL` (ex: `https://api.seudominio.com`)
> **Autenticação:** todos os endpoints (exceto login) exigem `Bearer token` no header `Authorization`.

---

## Índice

1. [Autenticação](#1-autenticação)
2. [Avisos (Notices)](#2-avisos-notices)
3. [Mensagens / Envio com Anexos](#3-mensagens--envio-com-anexos)
4. [Upload de Arquivos (Anexos)](#4-upload-de-arquivos-anexos)
5. [Galeria de Fotos](#5-galeria-de-fotos)
6. [Perfil / Destinatários](#6-perfil--destinatários)
7. [Matrícula Online](#7-matrícula-online)
8. [Materiais do Aluno](#8-materiais-do-aluno)

---

## Convenções

| Símbolo | Significado |
|---------|-------------|
| `*` | Campo obrigatório na resposta |
| `?` | Campo opcional / nullable |
| `[]` | Array |
| `bool` | `true` / `false` (ou `1` / `0` — o app aceita ambos) |
| ISO 8601 | Formato de datas: `"2024-05-10T14:30:00.000Z"` |

**Cache / Sincronização incremental (If-Modified-Since):**
Alguns endpoints suportam sincronização incremental. O app envia o header
`If-Modified-Since` com o valor recebido anteriormente no header `Last-Modified`.
Quando nada mudou, o backend deve responder `304 Not Modified` (sem body) ou
`204 No Content`. Quando há dados novos, responde `200` com o body completo e
inclui o header `Last-Modified` na resposta.

---

## 1. Autenticação

> Endpoints de login já existentes — não alterados nesta rodada. Listados apenas
> para referência de onde o `accessToken` é obtido.

O token JWT retornado no login é armazenado localmente e enviado como:

```
Authorization: Bearer <accessToken>
```

---

## 2. Avisos (Notices)

### 2.1 Listar Avisos

```
GET /avisos/
```

**Descrição:** Retorna todos os avisos do usuário autenticado. Suporta
sincronização incremental via `If-Modified-Since`.

**Headers de requisição:**

```
Authorization: Bearer <token>
If-Modified-Since: <valor retornado anteriormente em Last-Modified>   (opcional)
```

**Respostas:**

| Status | Significado |
|--------|-------------|
| `200`  | Lista de avisos (body abaixo) |
| `204`  | Nenhum aviso disponível (body vazio) |
| `304`  | Nenhuma alteração desde `If-Modified-Since` (body vazio) |

**Response `200` — body (array):**

```json
[
  {
    "id_aviso": 42,
    "titulo": "Reunião de pais",
    "mensagem": "Informamos que haverá reunião na sexta-feira às 19h.",
    "dt_cadastro": "2024-05-10T14:30:00.000Z",
    "dt_alteracao": "2024-05-10T15:00:00.000Z",
    "id_escola": 1,
    "ids_dests": [101, 102],
    "lido": false,
    "deletado": false,
    "tipo": "comunicado",
    "respondido": false,
    "resposta": null,
    "urgente": false,
    "ce_aviso_categoria": 1,
    "imagens": [
      "https://cdn.exemplo.com/avisos/img1.jpg",
      "https://cdn.exemplo.com/avisos/img2.png"
    ],
    "arquivos": [
      "https://cdn.exemplo.com/avisos/documento.pdf"
    ]
  }
]
```

**Tipos dos campos:**

| Campo | Tipo | Notas |
|-------|------|-------|
| `id_aviso` * | `int` | Identificador único do aviso |
| `titulo` ? | `string \| null` | Título do aviso |
| `mensagem` * | `string` | Corpo do aviso |
| `dt_cadastro` * | `string` (ISO 8601) | Data de criação |
| `dt_alteracao` * | `string` (ISO 8601) | Data da última alteração |
| `id_escola` * | `int` | ID da escola |
| `ids_dests` * | `int[]` | IDs dos alunos destinatários |
| `lido` * | `bool` | Se o aviso foi lido |
| `deletado` * | `bool` | Se o aviso foi deletado |
| `tipo` ? | `string \| null` | Tipo/categoria textual |
| `respondido` * | `bool` | Se o aviso foi respondido |
| `resposta` ? | `string \| null` | Texto da resposta |
| `urgente` * | `bool` | Se é urgente |
| `ce_aviso_categoria` ? | `int \| null` | Categoria numérica (default: `1`) |
| `imagens` ? | `string[]` | **NOVO** — URLs de imagens anexadas |
| `arquivos` ? | `string[]` | **NOVO** — URLs de outros arquivos anexados |

> **Sobre `imagens` e `arquivos`:**
> O app também aceita os aliases em inglês `images` / `files` e aceita que
> cada item da lista seja um objeto `{ "url": "...", "caption": "..." }` ou
> `{ "link": "..." }` além de string simples.

**Header de resposta esperado:**

```
Last-Modified: Fri, 10 May 2024 15:00:00 GMT
```

---

### 2.2 Marcar Aviso como Lido

```
PATCH /notices/{noticeId}
```

> **Atenção (TODO no código):** Esta rota usa `/notices` (inglês). A intenção é
> migrar para `/avisos/{noticeId}` futuramente. Implementar de acordo com o que
> o backend já tiver — apenas garantir que ambos os caminhos funcionem, ou
> atualizar o app quando a migração ocorrer.

**Parâmetros de rota:**

| Param | Tipo | Descrição |
|-------|------|-----------|
| `noticeId` | `int` | ID do aviso (`id_aviso`) |

**Headers:**

```
Authorization: Bearer <token>
Content-Type: application/json
```

**Body:**

```json
{
  "read": true
}
```

**Resposta esperada:** qualquer `2xx`.

---

### 2.3 Deletar Aviso

```
DELETE /notices/{noticeId}
```

> Mesma observação de migração de rota do item 2.2.

**Parâmetros de rota:**

| Param | Tipo | Descrição |
|-------|------|-----------|
| `noticeId` | `int` | ID do aviso (`id_aviso`) |

**Headers:**

```
Authorization: Bearer <token>
```

**Resposta esperada:** `2xx`. O app trata qualquer erro não-2xx como falha.

---

## 3. Mensagens / Envio com Anexos

### 3.1 Enviar Mensagem para Secretaria

```
POST /notices
```

**Descrição:** Envia uma mensagem de um responsável para a secretaria/setor,
opcionalmente com anexos (URLs já hospedadas via endpoint `/uploads`).

**Headers:**

```
Authorization: Bearer <token>
Content-Type: application/json
```

**Body:**

```json
{
  "alunos": [101],
  "msg": "Bom dia, gostaria de solicitar a segunda via do histórico.",
  "dest": 3,
  "dests": [7, 12],
  "anexos": [
    "https://cdn.exemplo.com/uploads/doc_20240510_a1b2c3.pdf",
    "https://cdn.exemplo.com/uploads/img_20240510_d4e5f6.jpg"
  ]
}
```

**Campos do body:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `alunos` | `int[]` | Sim | Array com o ID do aluno (sempre um item neste caso) |
| `msg` | `string` | Sim | Texto da mensagem |
| `dest` | `int` | Sim | Tipo de destino — `3` = mensagem para setor |
| `dests` | `int[]` | Não | IDs dos destinatários (administradores/setores) |
| `anexos` | `string[]` | Não | **NOVO** — URLs dos arquivos já enviados via `/uploads` |

**Resposta `200`:**

```json
{
  "mensagem": "A mensagem foi enviada com sucesso."
}
```

**Resposta de erro (não-200):**

```json
{
  "mensagem": "Erro ao enviar mensagem."
}
```

---

### 3.2 Enviar Alerta

```
POST /notices
```

**Descrição:** Envio de alerta (diferente de mensagem comum — sem texto livre,
sem destinatários específicos).

**Body:**

```json
{
  "alunos": [101, 102],
  "dest": 4
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `alunos` | `int[]` | IDs dos alunos |
| `dest` | `int` | Tipo de destino — `4` = alerta |

**Resposta `200`:**

```json
{
  "mensagem": "O alerta foi enviado."
}
```

---

## 4. Upload de Arquivos (Anexos)

> **Novo endpoint** — necessário para suportar envio de mensagens com anexos.
> Implementa o padrão de upload em 2 etapas:
> **Etapa 1:** enviar o arquivo → receber a URL.
> **Etapa 2:** enviar a mensagem com as URLs dos arquivos (endpoint `/notices`).

### 4.1 Upload de Arquivo

```
POST /uploads
```

**Descrição:** Recebe um arquivo via `multipart/form-data`, armazena, e retorna
a URL pública permanente do arquivo.

**Headers:**

```
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

**Body (multipart):**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `file` | `binary` | O arquivo a ser enviado |

**Exemplo cURL:**

```bash
curl -X POST https://api.exemplo.com/uploads \
  -H "Authorization: Bearer <token>" \
  -F "file=@/caminho/para/documento.pdf"
```

**Resposta `200` ou `201`:**

O app aceita as seguintes estruturas (da mais à menos preferida):

```json
{ "url": "https://cdn.exemplo.com/uploads/documento_abc123.pdf" }
```

```json
{ "file_url": "https://cdn.exemplo.com/uploads/documento_abc123.pdf" }
```

```json
{ "fileUrl": "https://cdn.exemplo.com/uploads/documento_abc123.pdf" }
```

```json
{ "location": "https://cdn.exemplo.com/uploads/documento_abc123.pdf" }
```

```json
{ "path": "https://cdn.exemplo.com/uploads/documento_abc123.pdf" }
```

```json
{
  "data": {
    "url": "https://cdn.exemplo.com/uploads/documento_abc123.pdf"
  }
}
```

Ou simplesmente uma string pura no body:

```
"https://cdn.exemplo.com/uploads/documento_abc123.pdf"
```

> Recomendação: usar `{ "url": "..." }` por ser o formato mais simples.

**Restrições de arquivo (validadas no client antes do upload):**

| Tipo | Extensões aceitas |
|------|------------------|
| Imagem | `jpg`, `jpeg`, `png`, `gif`, `webp`, `heic`, `bmp` |
| Vídeo | `mp4`, `mov`, `avi`, `mkv`, `3gp`, `webm` |
| Áudio | `mp3`, `wav`, `m4a`, `aac`, `ogg`, `opus`, `amr` |
| Documento | `pdf`, `doc`, `docx`, `xls`, `xlsx`, `ppt`, `pptx`, `txt`, `rtf`, `csv`, `odt`, `ods`, `odp` |

Tamanho máximo por arquivo: **2 MB** (após compressão client-side para imagens/vídeos).

---

## 5. Galeria de Fotos

> **Refatorado** — a estrutura anterior era plana (lista de imagens por galeria).
> Agora é hierárquica: **Galeria → Posts → Imagens**.

### 5.1 Listar Galerias (com Posts e Imagens)

```
GET /gallery
```

**Descrição:** Retorna todas as galerias do usuário, com seus posts e imagens.
Suporta sincronização incremental via `If-Modified-Since`.

**Headers:**

```
Authorization: Bearer <token>
If-Modified-Since: <valor retornado anteriormente em Last-Modified>   (opcional)
```

**Respostas:**

| Status | Significado |
|--------|-------------|
| `200`  | Lista de galerias |
| `304`  | Nenhuma alteração |
| `204`  | Sem dados |

**Response `200` — body:**

O backend pode retornar um **array** de galerias (preferencial) ou um **objeto**
único (o app trata ambos).

```json
[
  {
    "id": "10",
    "title": "Festa Junina 2024",
    "description": "Fotos da festa junina da turma do 3º ano.",
    "deleted": false,
    "posts": [
      {
        "id": "55",
        "galleryId": "10",
        "title": "Chegada dos alunos",
        "description": "Momento da chegada das crianças fantasiadas.",
        "createdAt": "2024-06-15T09:00:00.000Z",
        "updatedAt": "2024-06-15T10:00:00.000Z",
        "isFavorite": false,
        "studentId": 101,
        "deleted": false,
        "images": [
          {
            "url": "https://cdn.exemplo.com/gallery/img_001.jpg",
            "caption": "Entrada da quadra"
          },
          {
            "url": "https://cdn.exemplo.com/gallery/img_002.jpg",
            "caption": null
          }
        ]
      }
    ]
  }
]
```

**Campos — Galeria:**

| Campo | Tipo | Notas |
|-------|------|-------|
| `id` * | `string \| int` | ID da galeria (o app converte para string) |
| `title` * | `string` | Título da galeria |
| `description` * | `string` | Descrição |
| `deleted` ? | `bool` | Marcação de remoção para sync incremental |
| `posts` * | `Post[]` | Lista de posts da galeria |

> O app também aceita o alias legado `pictures` no lugar de `posts`.

**Campos — Post:**

| Campo | Tipo | Notas |
|-------|------|-------|
| `id` * | `string \| int` | ID do post |
| `galleryId` ? | `string \| int` | ID da galeria pai (alias: `gallery_id`) |
| `title` * | `string` | Título do post |
| `description` * | `string` | Descrição do post |
| `createdAt` * | `string` (ISO 8601) | Data de criação |
| `updatedAt` ? | `string` (ISO 8601) \| `null` | Data de atualização |
| `isFavorite` * | `bool` | Se o post é favorito do aluno |
| `studentId` * | `int` | ID do aluno dono do post (alias: `student_id`) |
| `deleted` ? | `bool` | Marcação de remoção para sync incremental |
| `images` * | `Image[]` | Lista de imagens (alias legado: `picturesUrl`) |

**Campos — Image:**

| Campo | Tipo | Notas |
|-------|------|-------|
| `url` * | `string` | URL da imagem (aliases: `link`, `src`) |
| `caption` ? | `string \| null` | Legenda da imagem |

> O app aceita que cada item de `images` seja uma **string pura** (apenas a URL)
> ou um **objeto** `{ url, caption }`.

**Header de resposta esperado:**

```
Last-Modified: Sat, 15 Jun 2024 10:00:00 GMT
```

---

### 5.2 Deletar Post

```
DELETE /gallery/{userId}/posts/{postId}
```

**Descrição:** Remove um post específico da galeria do usuário.

**Parâmetros de rota:**

| Param | Tipo | Descrição |
|-------|------|-----------|
| `userId` | `int` | ID do usuário/aluno |
| `postId` | `string` | ID do post |

**Headers:**

```
Authorization: Bearer <token>
Content-Type: application/json
```

**Resposta esperada:** `2xx` (sem body ou body ignorado).

---

### 5.3 Marcar/Desmarcar Post como Favorito

```
PUT /gallery/{userId}/posts/{postId}/favorite
```

**Parâmetros de rota:**

| Param | Tipo | Descrição |
|-------|------|-----------|
| `userId` | `int` | ID do usuário/aluno |
| `postId` | `string` | ID do post |

**Headers:**

```
Authorization: Bearer <token>
Content-Type: application/json
```

**Body:**

```json
{
  "is_favorite": true
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `is_favorite` | `bool` | `true` para favoritar, `false` para desfavoritar |

**Resposta esperada:** `2xx`.

---

## 6. Perfil / Destinatários

### 6.1 Listar Destinatários (Setores/Secretaria)

```
GET /notices/destinatarios/
```

**Descrição:** Retorna os setores e seus respectivos destinatários disponíveis
para envio de mensagens.

**Headers:**

```
Authorization: Bearer <token>
```

**Response `200` — body (array):**

```json
[
  {
    "departamento": "Secretaria",
    "destinatarios": [
      {
        "id_administrador": 7,
        "nome": "Maria Silva",
        "alunos": [101, 102, 103]
      },
      {
        "id_administrador": 12,
        "nome": "João Santos",
        "alunos": [101]
      }
    ]
  },
  {
    "departamento": "Coordenação",
    "destinatarios": [
      {
        "id_administrador": 15,
        "nome": "Ana Costa",
        "alunos": [101, 102]
      }
    ]
  }
]
```

**Campos — Setor:**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `departamento` * | `string` | Nome do setor/departamento |
| `destinatarios` * | `Destinatario[]` | Lista de pessoas do setor |

**Campos — Destinatário:**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id_administrador` * | `int` | ID do destinatário |
| `nome` * | `string` | Nome do destinatário |
| `alunos` * | `int[]` | IDs dos alunos vinculados a este destinatário |

---

## 7. Matrícula Online

### 7.1 Obter Dados de Matrícula

```
GET /matriculaonline
```

**Headers:**

```
Authorization: Bearer <token>
```

**Response `200` — body:**

```json
{
  "matricula_aluno_novato": { },
  "matriculas": [ ]
}
```

> O app verifica a presença de `matricula_aluno_novato` **ou** `matriculas` no
> body. Se nenhum estiver presente, trata como erro.

---

## 8. Materiais do Aluno

### 8.1 Obter Link de Materiais

```
GET /students/materiais/{studentId}
```

**Parâmetros de rota:**

| Param | Tipo | Descrição |
|-------|------|-----------|
| `studentId` | `int` | ID do aluno |

**Headers:**

```
Authorization: Bearer <token>
```

**Response `200`:**

```json
{
  "url_materiais": "https://drive.google.com/drive/folders/abc123"
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `url_materiais` * | `string` | URL para a pasta/página de materiais do aluno |

---

## Resumo de Endpoints

| Método | Rota | Feature | Status |
|--------|------|---------|--------|
| `GET` | `/avisos/` | Listar avisos | Existente — campos `imagens`/`arquivos` adicionados |
| `PATCH` | `/notices/{id}` | Marcar aviso como lido | Existente |
| `DELETE` | `/notices/{id}` | Deletar aviso | Existente |
| `POST` | `/notices` | Enviar mensagem/alerta | Existente — campo `anexos` adicionado |
| `POST` | `/uploads` | Upload de arquivo | **NOVO** |
| `GET` | `/gallery` | Listar galerias | Existente — estrutura Posts/Images refatorada |
| `DELETE` | `/gallery/{userId}/posts/{postId}` | Deletar post | **NOVO** |
| `PUT` | `/gallery/{userId}/posts/{postId}/favorite` | Favoritar post | **NOVO** |
| `GET` | `/notices/destinatarios/` | Listar destinatários | Existente |
| `POST` | `/notices` | Enviar alerta | Existente |
| `GET` | `/matriculaonline` | Dados de matrícula | Existente |
| `GET` | `/students/materiais/{studentId}` | URL de materiais | Existente |

---

## Notas de Implementação

### Sincronização Incremental (Offline-First)

Os endpoints `GET /avisos/` e `GET /gallery` implementam sincronização
incremental. O fluxo esperado:

1. **Primeira requisição:** sem `If-Modified-Since` → resposta `200` com todos os
   dados + header `Last-Modified`.
2. **Requisições subsequentes:** app envia `If-Modified-Since: <valor salvo>`.
   - Se houve alteração → `200` com os dados atualizados + novo `Last-Modified`.
   - Se não houve alteração → `304` (sem body) ou `204`.

Para a galeria, o campo `deleted: true` em uma galeria ou post indica que aquele
item deve ser removido do cache local do app (soft-delete via sync).

### Upload em 2 Etapas

O envio de mensagens com anexos segue este fluxo:

```
[App]                           [Backend]
  |                                 |
  |-- POST /uploads (arquivo 1) --->|
  |<------- { url: "https://..." }--|
  |                                 |
  |-- POST /uploads (arquivo 2) --->|
  |<------- { url: "https://..." }--|
  |                                 |
  |-- POST /notices (msg + urls) -->|
  |<------- { mensagem: "OK" } -----|
```

### Tolerância de Formato

O app é tolerante a variações de nomenclatura de campos para facilitar
compatibilidade com versões anteriores da API:

| Campo preferido | Aliases aceitos |
|-----------------|-----------------|
| `posts` (galeria) | `pictures` |
| `images` (post) | `picturesUrl` |
| `galleryId` | `gallery_id` |
| `studentId` | `student_id` |
| `deleted` | `deletado` |
| `imagens` (aviso) | `images` |
| `arquivos` (aviso) | `files` |
| `url` (imagem) | `link`, `src` |
