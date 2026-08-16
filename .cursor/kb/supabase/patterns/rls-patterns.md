# Políticas RLS Prontas

## Catálogo de políticas por cenário

### 1. Tabela pessoal (usuário vê só seus dados)

```sql
-- Aplicar em: notas, preferências, histórico pessoal
CREATE POLICY "pessoal_select" ON notas FOR SELECT USING (user_id = auth.uid());
CREATE POLICY "pessoal_insert" ON notas FOR INSERT WITH CHECK (user_id = auth.uid());
CREATE POLICY "pessoal_update" ON notas FOR UPDATE USING (user_id = auth.uid()) WITH CHECK (user_id = auth.uid());
CREATE POLICY "pessoal_delete" ON notas FOR DELETE USING (user_id = auth.uid());
```

### 2. Multi-tenant — membro lê, admin escreve

```sql
-- Criar tabela de membership primeiro
CREATE TABLE memberships (
  user_id UUID REFERENCES auth.users(id),
  org_id  UUID NOT NULL,
  role    TEXT NOT NULL CHECK (role IN ('admin', 'member', 'viewer')),
  PRIMARY KEY (user_id, org_id)
);
ALTER TABLE memberships ENABLE ROW LEVEL SECURITY;
CREATE POLICY "proprio_membership" ON memberships FOR SELECT USING (user_id = auth.uid());

-- Helper: função para checar role (reuso nas políticas)
CREATE OR REPLACE FUNCTION auth.user_org_role(p_org_id UUID)
RETURNS TEXT LANGUAGE sql STABLE SECURITY DEFINER AS $$
  SELECT role FROM memberships WHERE user_id = auth.uid() AND org_id = p_org_id;
$$;

-- Políticas da tabela principal
CREATE POLICY "org_member_select"
ON casos FOR SELECT
USING (auth.user_org_role(org_id) IN ('admin', 'member', 'viewer'));

CREATE POLICY "org_member_insert"
ON casos FOR INSERT
WITH CHECK (auth.user_org_role(org_id) IN ('admin', 'member'));

CREATE POLICY "org_admin_update"
ON casos FOR UPDATE
USING (auth.user_org_role(org_id) = 'admin')
WITH CHECK (auth.user_org_role(org_id) = 'admin');

CREATE POLICY "org_admin_delete"
ON casos FOR DELETE
USING (auth.user_org_role(org_id) = 'admin');
```

### 3. Tabela de referência pública (somente leitura para todos)

```sql
-- Categorias, status, tipos — qualquer autenticado lê, ninguém escreve via app
CREATE POLICY "publico_select" ON categorias FOR SELECT USING (true);
-- Sem INSERT/UPDATE/DELETE via app — apenas via migration ou service_role
```

### 4. Compartilhamento (owner + convidados)

```sql
CREATE TABLE shares (
  resource_id UUID NOT NULL,
  shared_with UUID REFERENCES auth.users(id),
  permission  TEXT NOT NULL CHECK (permission IN ('read', 'write')),
  PRIMARY KEY (resource_id, shared_with)
);

CREATE POLICY "share_select"
ON documentos FOR SELECT
USING (
  user_id = auth.uid()  -- owner
  OR EXISTS (
    SELECT 1 FROM shares
    WHERE resource_id = documentos.id AND shared_with = auth.uid()
  )
);

CREATE POLICY "share_update"
ON documentos FOR UPDATE
USING (
  user_id = auth.uid()  -- só owner pode editar
  OR EXISTS (
    SELECT 1 FROM shares
    WHERE resource_id = documentos.id
      AND shared_with = auth.uid()
      AND permission = 'write'
  )
)
WITH CHECK (user_id = auth.uid() OR EXISTS (...));
```

### 5. Soft delete (deleted_at pattern)

```sql
-- Usuário não vê registros deletados
CREATE POLICY "excluindo_deletados"
ON registros FOR SELECT
USING (user_id = auth.uid() AND deleted_at IS NULL);

-- Ao "deletar", apenas setar deleted_at — nunca DELETE físico via app
CREATE POLICY "soft_delete"
ON registros FOR UPDATE
USING (user_id = auth.uid() AND deleted_at IS NULL)
WITH CHECK (user_id = auth.uid());
```

## Verificação de políticas

```sql
-- Listar políticas ativas de uma tabela
SELECT policyname, cmd, roles, qual, with_check
FROM pg_policies WHERE tablename = 'casos' ORDER BY cmd;

-- Simular query como usuário (usar em testes de migração)
BEGIN;
  SET LOCAL ROLE authenticated;
  SET LOCAL "request.jwt.claims" TO '{"sub": "00000000-0000-0000-0000-000000000001"}';
  SELECT count(*) FROM casos;  -- deve retornar apenas os casos do user acima
ROLLBACK;
```

## Checklist de RLS

- [ ] `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` em toda tabela com dados de usuário
- [ ] Políticas para todos os comandos usados (SELECT, INSERT, UPDATE, DELETE)
- [ ] UPDATE tem tanto `USING` quanto `WITH CHECK`
- [ ] Testada com simulação de usuário antes de ir a produção
- [ ] `service_role` key nunca exposta no frontend
