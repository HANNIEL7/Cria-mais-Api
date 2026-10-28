# Hackton Avanade

Aplicação para validar imagens de campanhas, identificar problemas técnicos e semânticos e orientar a revisão humana antes da aprovação final.

## Visão geral

O projeto recebe uma imagem, verifica se ela atende aos critérios mínimos de qualidade, e, em seguida, encaminha o arquivo para um modelo de visão para análise contextual. O sistema devolve:

- se a imagem foi aprovada ou rejeitada;
- em qual etapa ocorreu a decisão;
- a categoria detectada;
- os motivos que levaram à reprovação.

A regra de negócio central é simples: a máquina aponta, a pessoa decide. Nenhuma alteração é aplicada automaticamente sem revisão humana.

## Stack tecnológica

- Frontend: React + TypeScript + Vite
- Backend: Python + FastAPI
- Processamento visual: OpenCV
- IA/Visão: Ollama + modelo de visão
- OCR: suporte para extração de texto em imagem
- Infraestrutura: Docker + Docker Compose + Nginx

## Arquitetura

```text
Navegador
   |
   v
Nginx (porta 80)
   |------------------------------|
   v                              v
Frontend React/Vite           Backend FastAPI
   |                              |
   |                              v
   |                         OpenCV / regras
   |                              |
   |                              v
   |                         Ollama (modelo de visão)
   |
   +------------------------------+

```

Fluxo principal:

1. o usuário envia uma imagem pelo frontend;
2. o backend valida tipo, tamanho e integridade do arquivo;
3. o OpenCV valida critérios técnicos como resolução e proporção;
4. se passar na etapa técnica, a imagem é enviada ao modelo de visão;
5. o resultado é devolvido ao frontend em JSON estruturado.

## Funcionalidades

- upload e pré-visualização da imagem no frontend;
- validação de tipo MIME, arquivo vazio e imagem inválida;
- checagem técnica local com OpenCV;
- integração com Ollama para análise semântica;
- endpoint de saúde para verificar disponibilidade dos serviços;
- suporte para OCR de texto manuscrito/imagem.

## Endpoints principais

### `POST /api/analisar`

Recebe uma imagem no multipart `file` e retorna o resultado da análise.

Exemplos de resposta:

- `200`: análise concluída;
- `400`: arquivo vazio ou inválido;
- `413`: arquivo maior que o limite;
- `415`: tipo de arquivo não aceito;
- `502`, `503` ou `504`: falha de comunicação com o Ollama.

### `GET /api/saude`

Retorna o estado dos serviços internos, como:

- API;
- LanguageTool;
- IOPaint (quando ativado);
- motor OCR configurado.

### `POST /api/uploadHandwriting`

Recebe uma imagem e solicita a extração de texto para OCR.

## Estrutura do repositório

```text
.
├── Back-end/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── requirements-ocr.txt
│   ├── app/
│   │   ├── config.py
│   │   ├── db.py
│   │   ├── deps.py
│   │   ├── main.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── serializers.py
│   │   └── routes/
│   │       ├── approvals.py
│   │       ├── arts.py
│   │       ├── images.py
│   │       └── projects.py
│   └── services/
│       ├── adaptations.py
│       ├── analyses.py
│       ├── briefing.py
│       ├── corrections.py
│       ├── erros.py
│       ├── formatos.py
│       ├── mistakes.py
│       ├── ocr.py
│       ├── ortografia.py
│       ├── outpainting.py
│       ├── technician.py
│       └── vision.py
├── Front-end/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   └── app/
│       ├── index.html
│       ├── package.json
│       └── src/
├── Agentes/
├── data/
├── iopaint/
├── docker-compose.yaml
├── nginx.conf
├── .env.example
├── .gitignore
├── README.md
└── LICENSE (se existir no futuro)
```

## Como executar

### Pré-requisitos

- Docker Engine ou Docker Desktop
- Docker Compose
- espaço em disco suficiente para as imagens e para os modelos de IA/OCR

### Subir a aplicação

Na raiz do projeto:

```bash
docker compose up -d --build
```

Para visualizar os logs em tempo real:

```bash
docker compose up
```

Depois, abra:

```text
http://localhost/
```

O backend também pode ser acessado diretamente em:

```text
http://localhost:3000
```

### Serviços opcionais

O projeto conta com perfis opcionais para recursos extras:

```bash
docker compose --profile outpainting up -d --build
```

```bash
docker compose --profile ia up -d
```

## Desenvolvimento local

Para rodar o frontend localmente fora do container:

```bash
cd Front-end/app
npm install
npm run dev
```

Comandos úteis:

```bash
npm run build
npm run lint
npm run preview
```

## Validação e observações

- O primeiro uso pode demorar enquanto o Ollama baixa o modelo de visão.
- A etapa técnica evita chamar a IA quando a imagem já falha em critérios básicos.
- O backend trata falhas de comunicação com serviços externos de forma explícita.
- Os volumes do Docker mantêm modelos e dados persistidos entre reinicializações.

## Contribuição

As alterações devem ser feitas de forma incremental, mantendo a separação entre:

- regras de negócio;
- integração com serviços externos;
- camada HTTP;
- interface do usuário.

Este repositório foi organizado para facilitar manutenção, testes e evolução do fluxo de análise visual.
