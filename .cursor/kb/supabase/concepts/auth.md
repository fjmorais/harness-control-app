# Supabase Auth

## Métodos de autenticação

| Método | Quando usar |
|---|---|
| Magic Link (email) | Usuários internos, baixo volume, sem senha para gerenciar |
| OAuth (Google, GitHub) | Apps públicos, experiência social login |
| Password | Legado ou quando magic link não é viável |
| Phone OTP | Mobile, alta confiabilidade em regiões sem email |

## Magic Link

```typescript
// Enviar link
const { error } = await supabase.auth.signInWithOtp({
  email: "usuario@empresa.com",
  options: {
    emailRedirectTo: `${window.location.origin}/auth/callback`,
  },
})

// Callback — Supabase processa automaticamente o token na URL
// Em Next.js App Router: app/auth/callback/route.ts
import { createRouteHandlerClient } from "@supabase/auth-helpers-nextjs"
import { cookies } from "next/headers"

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const code = searchParams.get("code")

  if (code) {
    const supabase = createRouteHandlerClient({ cookies })
    await supabase.auth.exchangeCodeForSession(code)
  }

  return NextResponse.redirect(new URL("/dashboard", request.url))
}
```

## OAuth (Google)

```typescript
const { error } = await supabase.auth.signInWithOAuth({
  provider: "google",
  options: {
    redirectTo: `${window.location.origin}/auth/callback`,
    scopes: "email profile",
  },
})
```

## Sessão e usuário atual

```typescript
// Obter sessão atual
const { data: { session } } = await supabase.auth.getSession()

// Obter usuário atual (verificado no servidor)
const { data: { user } } = await supabase.auth.getUser()

// Escutar mudanças de auth
supabase.auth.onAuthStateChange((event, session) => {
  if (event === "SIGNED_IN")  { /* atualizar estado */ }
  if (event === "SIGNED_OUT") { /* limpar estado */ }
  if (event === "TOKEN_REFRESHED") { /* sessão renovada */ }
})
```

## Signout

```typescript
await supabase.auth.signOut()
// Remove cookies/localStorage + invalida token no servidor
```

## JWT e claims customizados

O JWT emitido pelo Supabase contém `sub` (user_id), `email`, `role`.
Para claims customizados (ex: `org_id`), usar Auth Hook ou `app_metadata`:

```sql
-- Adicionar claim customizado via função (executar como service_role)
UPDATE auth.users
SET app_metadata = app_metadata || '{"org_id": "org-abc"}'::jsonb
WHERE id = 'user-uuid';
```

```typescript
// Ler claim no frontend
const { data: { session } } = await supabase.auth.getSession()
const orgId = session?.user.app_metadata?.org_id
```

## Servidor: verificar JWT (FastAPI / Edge Function)

```python
# FastAPI — verificar JWT do Supabase
import jwt
from fastapi import HTTPException, Depends
from fastapi.security import HTTPBearer

security = HTTPBearer()

def verify_supabase_jwt(token: str = Depends(security)) -> dict:
    try:
        payload = jwt.decode(
            token.credentials,
            SUPABASE_JWT_SECRET,
            algorithms=["HS256"],
            audience="authenticated",
        )
        return payload
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Token inválido")
```

## Anti-padrões

```typescript
// ERRADO: service_role no frontend
const supabase = createClient(url, process.env.SERVICE_ROLE_KEY!)  // bypass total de RLS!

// ERRADO: confiar em dados do JWT sem verificação no servidor
const userId = localStorage.getItem("user_id")  // manipulável pelo usuário

// CERTO: sempre usar auth.uid() nas políticas RLS
// (automático quando se usa o cliente anon com sessão ativa)
```
