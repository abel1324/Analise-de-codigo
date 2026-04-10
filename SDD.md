# SDD - Software Design Document

## 1. Visão Geral do Sistema

O sistema é uma aplicação web do tipo SPA (Single Page Application) desenvolvida com React + TypeScript e empacotada com Vite. Ele suporta gestão de procedimentos de contratação pública, fornecedores, instituições e geração de documentos relacionados ao workflow.

A aplicação atual não possui backend conectado nem banco de dados persistente. Os dados são fornecidos por arquivos de mock local, carregados de forma estática em `src/data/*.ts` e manipulados em memória pelos stores do frontend.

## 2. Escopo do Documento

Este documento descreve:

- arquitetura de software
- infraestrutura tecnológica usada
- organização do código
- modelo lógico de dados equivalente a um banco de dados
- tabelas e relacionamentos necessários para uma implementação persistente

## 3. Arquitetura Técnica

### 3.1. Camadas do Sistema

- Apresentação (UI): React, Tailwind CSS, componentes Radix UI, layouts e páginas.
- Roteamento: `react-router-dom` gerencia as rotas da aplicação.
- Estado e dados: stores locais em arquivos TypeScript (`procedure-store.ts`, `document-store.ts`) com arrays de objetos em memória.
- Dados mock: `mock-data.ts`, `mock-suppliers.ts`, `mock-institutions.ts`, `document-store.ts`.
- Build: Vite via `vite.config.ts`.

### 3.2. Tecnologias Principais

- Node.js / npm
- Vite
- React 18
- TypeScript
- Tailwind CSS
- @shadcn/ui + Radix UI
- React Query (`@tanstack/react-query`)
- Sonner (toast notifications)
- jszip / jspdf / html2canvas para exportação
- Vitest para testes
- Playwright para testes E2E

### 3.3. Arquivo de Configuração Importante

- `package.json`: dependências e scripts
- `vite.config.ts`: configura plugin React SWC, alias `@` para `src`
- `tsconfig.json`: configurações de TypeScript
- `tailwind.config.ts`: configuração Tailwind

## 4. Infraestrutura de Execução

### 4.1. Ambiente de Desenvolvimento

- Node.js 18+ (o ambiente usado reportou Node 22.17.0)
- npm como gerenciador de pacotes
- comando de desenvolvimento: `npm run dev`
- comando de build: `npm run build`

### 4.2. Deploy Possível

A aplicação produz arquivos estáticos no diretório `dist/` e pode ser hospedada em:

- servidores estáticos (Netlify, Vercel, S3 + CloudFront)
- CDN
- qualquer servidor HTTP capaz de servir assets estáticos

### 4.3. Observação sobre Persistência

Atualmente não há persistência em banco de dados. Todos os dados são instanciados em memória na inicialização do frontend.

Uma implementação futura de backend poderia usar um banco relacional ou document store para armazenar os mesmos modelos.

## 5. Modelo Lógico de Dados

A seguir estão as entidades principais extraídas dos arquivos de mock e stores.

### 5.1. Tabela `users`

Representa o usuário atual e possíveis usuários do sistema.

```sql
CREATE TABLE users (
  id varchar(50) PRIMARY KEY,
  name varchar(255) NOT NULL,
  email varchar(255) NOT NULL,
  role varchar(50) NOT NULL,
  institution varchar(255) NOT NULL
);
```

Campos principais:

- `id`
- `name`
- `email`
- `role` (`ADMIN`, `GESTOR_CONTRATACAO`, `JURISTA`, `MEMBRO_CA`, `PRESIDENTE_CA`, `AUDITOR`, `VISUALIZADOR`)
- `institution`

### 5.2. Tabela `procedures`

Armazena os procedimentos de contratação pública.

```sql
CREATE TABLE procedures (
  id varchar(50) PRIMARY KEY,
  procedure_number varchar(100) NOT NULL,
  procedure_type varchar(100) NOT NULL,
  status varchar(100) NOT NULL,
  object_summary text NOT NULL,
  object_description text NOT NULL,
  institution varchar(255) NOT NULL,
  estimated_value numeric NOT NULL,
  created_by varchar(255) NOT NULL,
  created_at timestamptz NOT NULL,
  updated_at timestamptz NOT NULL,
  decision_date date,
  submission_deadline date,
  opening_session_date date
);
```

### 5.3. Tabela `procedure_audit_log`

Registra o histórico de ações sobre um procedimento.

```sql
CREATE TABLE procedure_audit_log (
  id serial PRIMARY KEY,
  procedure_id varchar(50) NOT NULL,
  action varchar(100) NOT NULL,
  user_name varchar(255) NOT NULL,
  timestamp timestamptz NOT NULL,
  details text,
  FOREIGN KEY (procedure_id) REFERENCES procedures(id)
);
```

### 5.4. Tabela `suppliers`

Armazena fornecedores cadastrados.

```sql
CREATE TABLE suppliers (
  id varchar(50) PRIMARY KEY,
  nif varchar(50) NOT NULL,
  corporate_name varchar(255) NOT NULL,
  trade_name varchar(255),
  legal_form varchar(255) NOT NULL,
  legal_representative varchar(255) NOT NULL,
  representative_role varchar(255) NOT NULL,
  representative_email varchar(255) NOT NULL,
  representative_phone varchar(50) NOT NULL,
  contact_email varchar(255) NOT NULL,
  contact_phone varchar(50) NOT NULL,
  address_street text NOT NULL,
  address_city varchar(255) NOT NULL,
  address_province varchar(255) NOT NULL,
  is_national boolean NOT NULL,
  is_mpmme boolean NOT NULL,
  certification_state varchar(50) NOT NULL,
  bank_name varchar(255),
  iban varchar(100),
  created_at timestamptz NOT NULL,
  updated_at timestamptz NOT NULL
);
```

### 5.5. Tabela `institutions`

Armazena instituições públicas e seus dados de contacto.

```sql
CREATE TABLE institutions (
  id varchar(50) PRIMARY KEY,
  name varchar(255) NOT NULL,
  acronym varchar(100) NOT NULL,
  nif varchar(50) NOT NULL,
  type varchar(100) NOT NULL,
  province varchar(100) NOT NULL,
  address text NOT NULL,
  phone varchar(50) NOT NULL,
  email varchar(255) NOT NULL,
  website varchar(255),
  responsible_name varchar(255) NOT NULL,
  responsible_role varchar(255) NOT NULL,
  is_active boolean NOT NULL,
  created_at timestamptz NOT NULL,
  updated_at timestamptz NOT NULL,
  street_name varchar(255),
  street_number varchar(50),
  municipality varchar(255),
  country varchar(255)
);
```

### 5.6. Tabela `documents`

Registra documentos emitidos para procedimentos.

```sql
CREATE TABLE documents (
  id varchar(50) PRIMARY KEY,
  template_code varchar(100) NOT NULL,
  template_name varchar(255) NOT NULL,
  procedure_id varchar(50) NOT NULL,
  procedure_number varchar(100) NOT NULL,
  institution varchar(255) NOT NULL,
  supplier_name varchar(255),
  emitted_by varchar(255) NOT NULL,
  emitted_at timestamptz NOT NULL,
  legal_basis varchar(255),
  FOREIGN KEY (procedure_id) REFERENCES procedures(id)
);
```

### 5.7. Tabela `workflow_definitions`

Contém as regras de workflow por tipo de procedimento.

```sql
CREATE TABLE workflow_definitions (
  procedure_type varchar(100) PRIMARY KEY,
  label varchar(255) NOT NULL,
  max_duration_days integer,
  legal_basis text NOT NULL,
  value_limit_max numeric,
  value_limit_description text
);

CREATE TABLE workflow_steps (
  id serial PRIMARY KEY,
  procedure_type varchar(100) NOT NULL,
  step_code varchar(100) NOT NULL,
  label varchar(255) NOT NULL,
  description text NOT NULL,
  sla_hours integer,
  required_documents text[],
  is_mandatory boolean NOT NULL,
  legal_basis text,
  FOREIGN KEY (procedure_type) REFERENCES workflow_definitions(procedure_type)
);
```

### 5.8. Tabela `document_templates` (Opcional)

Esta tabela pode documentar os tipos de documento usados no sistema.

```sql
CREATE TABLE document_templates (
  template_code varchar(100) PRIMARY KEY,
  template_name varchar(255) NOT NULL,
  description text
);
```

## 6. Relacionamentos

- `procedures` 1:N `procedure_audit_log`
- `procedures` 1:N `documents`
- `workflow_definitions` 1:N `workflow_steps`
- `procedure_type` em `procedures` refere-se a `workflow_definitions.procedure_type`

## 7. Pontos Importantes da Implementação Atual

- Dados de procedimentos são mantidos em memória em `src/data/procedure-store.ts`.
- Documentos emitidos são mantidos em `src/data/document-store.ts`.
- Fornecedores e instituições são representados por `src/data/mock-suppliers.ts` e `src/data/mock-institutions.ts`.
- Workflow, regras e documentos obrigatórios são definidos em `src/data/workflow-definitions.ts`.
- Não há API REST, GraphQL ou serviço de backend implementado neste repositório.

## 8. Recomendações para Evolução

Para tornar o sistema persistente em produção, as seguintes camadas são necessárias:

- Backend/API: Node.js + Express/NestJS ou similar.
- Banco de dados relacional: PostgreSQL, MySQL, SQL Server.
- Mapeamento ORM: Prisma, TypeORM ou Sequelize.
- Autenticação e autorização: JWT ou sessão com controle por papel.
- Migração de dados: transformação dos dados mock para tabelas reais.

## 9. Observação Final

Este SDD foi produzido com base no código disponível em `procure-hub-main`. Como não há conexão com banco de dados real ou infraestrutura de servidor neste repositório, o modelo de dados apresentado é um esquema lógico recomendado para implementação futura.
