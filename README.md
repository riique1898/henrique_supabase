# Pesquisa prática — Ionic, Vue, Capacitor e Supabase

Aplicação simples com tela de login, recuperação de senha, boas-vindas e perfil do usuário.

## Como executar

```bash
npm install
npm run dev
```

Copie `.env.example` para `.env` e preencha a URL e a chave anônima do seu projeto Supabase.

## Estrutura esperada no Supabase

Crie uma tabela `profiles` com as colunas `id` (uuid, chave primária), `name` (text), `birth_date` (date), `nickname` (text) e `avatar_url` (text). O `id` deve referenciar `auth.users.id`.

Crie também um bucket público chamado `avatars`. Em um projeto real, configure as políticas RLS para permitir que cada usuário leia e altere apenas o próprio perfil e seus arquivos.

Sem as variáveis do Supabase, o projeto abre em modo demonstração para facilitar a apresentação da interface.
