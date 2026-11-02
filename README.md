# Cria Mais

Plataforma para organizar campanhas de marketing, conferir artes automaticamente e aprovar com uma pessoa no controle.

O fluxo funciona assim: o usuário envia o briefing, cria o projeto, envia as artes, o sistema aponta pontos para revisar (preço, escrita, margem), a pessoa decide o que corrigir, gera versões para outros canais e um revisor aprova ou devolve. Só depois da aprovação a arte pode ser exportada.

> Princípio do projeto: a máquina aponta, a pessoa decide. Nenhuma alteração é aplicada sem revisão, a arte original nunca é sobrescrita e a aprovação é sempre humana.

Esta documentação cobre o back-end, que é a parte pronta do projeto. O front-end será desenvolvido futuramente.

---

## 1. Arquitetura

```text
Cliente HTTP ──:3000──► back-end (FastAPI :80)
                                      │
                       ┌────────────────────┼────────────────────────┐
                       ▼                    ▼                        ▼
               SQLite + arquivos     PaddleOCR (no back-end)   LanguageTool (container)
               (volume ./data)       lê o texto da arte        confere a escrita

                                     IOPaint (container, OPCIONAL)  estende o fundo ao adaptar formatos
                                     Ollama  (container, OPCIONAL)  fora do fluxo principal
```

Escolhas de tecnologia, seguindo a regra de usar IA generativa só quando ela é realmente necessária:

| Necessidade | Solução | IA generativa? |
|---|---|---|
| Ler o texto das artes | PaddleOCR | Não |
| Comparar preço com o briefing | Regras + extração de dados | Não |
| Conferir ortografia | LanguageTool local | Não |
| Texto perto da borda | Geometria dos elementos | Não |
| Aplicar correções | Substituição de texto em cópia da arte | Não |
| Adaptar para outro formato | Outpainting (IOPaint), opcional | Sim (modelo LaMa), controlado |

---

## 2. Estrutura de pastas

```text
projeto/
├── docker-compose.yaml         # Orquestra todos os serviços (com profiles opcionais)
├── .env.example                # Variáveis do compose (copie para .env)
├── .gitignore
├── README.md
│
├── Back-end/
│   ├── Dockerfile
│   ├── requirements.txt        # Dependências leves (API, banco, imagens, PDF/DOCX)
│   ├── requirements-ocr.txt    # Dependências do PaddlePaddle/PaddleOCR
│   ├── app/
│   │   ├── main.py             # Cria o app, CORS, rotas, tratamento de erros, /api/saude
│   │   ├── config.py           # Todos os parâmetros (limites, URLs, regras)
│   │   ├── db.py               # Conexão SQLAlchemy e criação das tabelas
│   │   ├── models.py           # Tabelas: Projeto, Arte, VersaoArte, Analise, Achado...
│   │   ├── schemas.py          # Modelos do módulo opcional de visão (llava)
│   │   ├── serializers.py      # Objeto do banco -> JSON que o front consome
│   │   ├── deps.py             # Usuário atual (X-Usuario) e checagem de revisor
│   │   │
│   │   ├── routes/             # Camada HTTP: só orquestra, sem regra pesada
│   │   │   ├── projects.py     # Dashboard, projetos, briefing (extrair, anexar, baixar)
│   │   │   ├── arts.py         # Upload, versões, análise, correções, adaptação, comentários
│   │   │   ├── images.py       # Análise e leitura de imagens
│   │   │   └── approvals.py    # Abas, avaliação, aprovar, devolver, exportar
│   │   │
│   │   └── services/           # Regras de negócio e integrações
│   │       ├── briefing.py     # Lê PDF/DOCX/TXT e extrai os campos (com a página)
│   │       ├── ocr.py          # Interface do OCR (PaddleOCR por trás)
│   │       ├── ortografia.py   # Cliente do LanguageTool
│   │       ├── analyses.py     # As conferências automáticas
│   │       ├── corrections.py  # Aplica correções aceitas numa cópia da arte
│   │       ├── outpainting.py  # Estende o fundo via IOPaint (código do outro dev, corrigido)
│   │       ├── adaptations.py  # Leva a arte ao formato do canal (usa o outpainting)
│   │       ├── formatos.py     # Tabela de canais e tamanhos
│   │       ├── armazenamento.py# Arquivos em disco, com proteção de caminho
│   │       ├── erros.py        # ErroNegocio / ErroServicoIA (status HTTP + mensagem)
│   │       └── vision.py       # Opcional: llava via Ollama (fora do fluxo principal)
│
├── languagetool/
│   └── Dockerfile              # Imagem local do corretor ortográfico
├── iopaint/
│   └── Dockerfile              # Imagem do IOPaint (não existe oficial no Docker Hub)
│
│   ── Pastas que existem no repositório mas NÃO fazem parte desta entrega ──
├── Agentes/                    # Volumes de modelos, criados em tempo de execução:
│   ├── ollama-models/          #   (só com --profile ia)
│   ├── iopaint-models/         #   modelo LaMa baixado na 1ª execução
│   └── paddle-cache/           #   modelos do PaddleOCR baixados na 1ª leitura
└── data/                       # Criada em tempo de execução: banco SQLite, briefings, versões das artes
```

> Atenção: se existir uma pasta `IOPaint/` (com maiúsculas) ao lado de `iopaint/`, ela é duplicada e deve ser removida. No Windows e no macOS as duas colidem. A pasta `.devcontainer/`, se existir, aponta para um Dockerfile que não existe e pode ser apagada.

### Como as camadas se relacionam

```text
routes/  ──chama──►  services/  ──usa──►  models/db
   │                     │
   └─ HTTP e validação   └─ regras, OCR, IA, arquivos
```

Regra de ouro: rotas não contêm regra de negócio. Para mudar uma conferência, mexa em `app/services/analyses.py`. Para trocar de OCR, em `app/services/ocr.py`.

---

## 3. Como rodar

Requisito: Docker.

```bash
# 1) Subida padrão (back-end e LanguageTool)
docker compose up -d --build

# 2) Com outpainting real (IOPaint)
echo "OUTPAINTING_ATIVO=true" > .env
docker compose --profile outpainting up -d --build

# 3) Com Ollama (experimentos com llava; fora do fluxo principal)
docker compose --profile ia up -d
```

| O quê | Endereço |
|---|---|
| API | http://localhost:3000 |
| Documentação interativa da API (Swagger) | http://localhost:3000/docs |
| Estado dos serviços de apoio | http://localhost:3000/api/saude |

O front-end será conectado futuramente à API exposta na porta 3000.

Hot reload: use `docker compose watch` (ou `up --watch`). Com `up -d` puro, o código não é sincronizado.

O primeiro build é demorado (PaddlePaddle é grande) e a primeira leitura de OCR baixa os modelos para `Agentes/paddle-cache`. A primeira subida do IOPaint também baixa o modelo LaMa.

### Validação local

O backend deve ser validado em três etapas: sintaxe, configuração do Compose e endpoints HTTP. Execute os comandos a partir da raiz do repositório:

```bash
# 1) Sintaxe Python
python3 -m compileall -q Back-end/app

# 2) Configuração resolvida do Compose
docker compose config

# 3) Subida do backend e dos serviços
docker compose up -d --build

# 4) Smoke tests da API
curl -fsS http://localhost:3000/api
curl -fsS http://localhost:3000/api/saude
curl -fsS -H "X-Usuario: ana" http://localhost:3000/api/me
curl -fsS -H "X-Usuario: ana" http://localhost:3000/api/formatos

# 5) Estado dos containers
docker compose ps
```

O resultado esperado de `/api` é `{"status":"ok"}`. Em `/api/saude`, `api` e `languagetool` devem estar como `true`. Um usuário inválido deve retornar HTTP 401.

---

## 4. Endpoints e regras do sistema

### API principal

- `GET /api` — confirma que a API está no ar.
- `GET /api/saude` — verifica estado dos serviços de apoio.
- `GET /api/me` — retorna o identificador do usuário do header `X-Usuario`.
- `GET /api/formatos` — lista os formatos e dimensões suportados.
- `POST /api/projetos` — cria um projeto e inicia o fluxo de briefing.
- `GET /api/projetos` — lista os projetos.
- `POST /api/arts/upload` — faz upload da arte para revisão.
- `POST /api/arts/{id}/analise` — dispara a análise automatizada da arte.
- `POST /api/arts/{id}/corrigir` — aplica correções aprovadas pela pessoa.
- `POST /api/arts/{id}/adaptar` — adapta a arte para outro canal.
- `POST /api/aprovacoes` — registra aprovação ou devolução da arte.
- `POST /api/exportar` — exporta o arquivo final quando aprovado.

### Regras de negócio centrais

- a revisão humana é obrigatória antes da aprovação final;
- o sistema não sobrescreve a arte original;
- todas as correções são feitas em cópia da imagem ou versão derivada;
- apenas um revisor pode aprovar, devolver ou exportar a arte;
- o fluxo de adaptação para outros formatos só acontece com a revisão humana.

---

## 5. Tecnologias e dependências

- Backend: Python, FastAPI, SQLAlchemy, Pydantic
- Banco de dados: SQLite
- OCR: PaddleOCR
- Validação ortográfica: LanguageTool
- Processamento de imagem: OpenCV
- Outpainting: IOPaint (opcional)
- IA opcional: Ollama / modelos visionários
- Containers: Docker + Docker Compose

---

## 6. Observações importantes

- O primeiro build é mais lento por causa do PaddleOCR e do download de modelos.
- O volume `Agentes/paddle-cache` armazena os modelos do OCR para evitar downloads repetidos.
- A opção de outpainting e a integração com Ollama são extras e não fazem parte do fluxo principal.
- O front-end ainda não faz parte da execução atual e será conectado futuramente à API.
- O arquivo `nginx.conf` está reservado para quando o front-end for incorporado ao Compose.

---

## 7. Desenvolvimento e manutenção

Para manter o código organizado, a regra é clara:

- rotas orquestram chamadas HTTP;
- services implementam regras de negócio;
- modelos e banco persistem estado;
- arquivos e integrações ficam isolados em serviços específicos.

Se você quiser trocar a regra de revisão automática, ajuste `Back-end/app/services/analyses.py`; para trocar o motor de OCR, altere `Back-end/app/services/ocr.py`.

---

## 8. Conclusão

Este projeto foi desenhado para reduzir retrabalho e facilitar a revisão de campanhas visuais, com um fluxo que combina regras automatizadas, OCR, validação textual e revisão humana. A combinação torna o processo mais rápido, mais confiável e mais controlado, sem perder o critério humano na decisão final.
