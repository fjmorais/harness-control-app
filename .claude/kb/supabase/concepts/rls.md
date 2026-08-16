# Row Level Security (RLS)

## Por que RLS

Sem RLS, qualquer usuário autenticado pode `SELECT * FROM tabela` e ver todos os dados.
RLS garante que cada usuário só enxerga e modifica suas próprias linhas — no banco, não na aplicação.

## Ativar RLS

```sql
-- Habilitar em toda tabela com dados de usuário
ALTER TABLE casos ENABLE ROW LEVEL SECURITY;
ALTER TABLE casos FORCE ROW LEVEL SECURITY;  -- bloqueia até o owner da tabela
```

## Políticas fundamentais

### Owner pattern (usuário vê apenas seus dados)

```sql
-- SELECT: usuário vê apenas seus próprios casos
CREATE POLICY "usuario_ve_proprios_casos"
ON casos FOR SELECT
USING (user_id = auth.uid());

-- INSERT: usuário só insere com seu próprio user_id
CREATE POLICY "usuario_insere_proprios_casos"
ON casos FOR INSERT
WITH CHECK (user_id = auth.uid());

-- UPDATE: usuário só atualiza seus próprios casos
CREATE POLICY "usuario_atualiza_proprios_casos"
ON casos FOR UPDATE
USING (user_id = auth.uid())
WITH CHECK (user_id = auth.uid());

-- DELETE: usuário só deleta seus próprios casos
CREATE POLICY "usuario_deleta_proprios_casos"
ON casos FOR DELETE
USING (user_id = auth.uid());
```

### Multi-tenant (usuário vê dados da sua organização)

```sql
-- Tabela de membros da organização
CREATE TABLE org_members (
  user_id  UUID REFERENCES auth.users(id),
  org_id   UUID NOT NULL,
  role     TEXT NOT NULL DEFAULT 'member',  -- 'admin' | 'member' | 'viewer'
  PRIMARY KEY (user_id, org_id)
);

-- Política: usuário vê dados da sua org
CREATE POLICY "org_member_ve_dados"
ON casos FOR SELECT
USING (
  org_id IN (
    SELECT org_id FROM org_members WHERE user_id = auth.uid()
  )
);
```

### Admin bypass (admin vê tudo da org)

```sql
CREATE POLICY "admin_ve_todos_casos_da_org"
ON casos FOR ALL
USING (
  EXISTS (
    SELECT 1 FROM org_members
    WHERE user_id = auth.uid()
      AND org_id = casos.org_id
      AND role = 'admin'
  )
);
```

### Leitura pública (tabelas de referência)

```sql
CREATE POLICY "publico_le_categorias"
ON categorias FOR SELECT
USING (true);  -- qualquer um pode ler
```

## `auth.uid()` vs `auth.jwt()`

```sql
auth.uid()   -- UUID do usuário autenticado (shortcut)
auth.jwt()   -- payload JWT completo

-- Acessar claims customizados do JWT
(auth.jwt() ->> 'org_id')::uuid

-- Exemplo: política usando claim do JWT
CREATE POLICY "jwt_org_claim"
ON casos FOR SELECT
USING (org_id = (auth.jwt() ->> 'org_id')::uuid);
```

## SECURITY DEFINER — cuidado

```sql
-- Função com SECURITY DEFINER executa como o owner (bypassando RLS)
-- Usar APENAS quando necessário e documentado
CREATE OR REPLACE FUNCTION public.get_all_org_cases(p_org_id UUID)
RETURNS SETOF casos
LANGUAGE sql
SECURITY DEFINER  -- bypassa RLS — DOCUMENTE O MOTIVO
SET search_path = public
AS $$
  SELECT * FROM casos WHERE org_id = p_org_id;
$$;
-- GRANT apenas para roles específicas
GRANT EXECUTE ON FUNCTION public.get_all_org_cases TO authenticated;
```

## Verificar políticas ativas

```sql
-- Ver todas as políticas de uma tabela
SELECT policyname, cmd, qual, with_check
FROM pg_policies
WHERE tablename = 'casos';

-- Testar como um usuário específico
SET LOCAL ROLE authenticated;
SET LOCAL "request.jwt.claims" TO '{"sub": "user-uuid-aqui"}';
SELECT * FROM casos;  -- deve retornar apenas os dados do usuário
RESET ROLE;
```

## Anti-padrões

```sql
-- ERRADO: tabela sem RLS exposta via anon key
-- Qualquer usuário não autenticado pode fazer SELECT

-- ERRADO: política que filtra no SELECT mas não no INSERT/UPDATE
-- Usuário pode inserir dados com org_id de outra organização

-- ERRADO: confiar apenas em filtros da aplicação
-- Filtros de app podem ser bypassados via Supabase JS direto
-- RLS deve ser a garantia, não o filtro da app

-- CERTO: sempre USING + WITH CHECK no UPDATE
CREATE POLICY "update_safe"
ON casos FOR UPDATE
USING (user_id = auth.uid())        -- leitura: só os seus
WITH CHECK (user_id = auth.uid());  -- escrita: não pode mudar para outro usuário
```
