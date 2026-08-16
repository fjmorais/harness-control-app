---
domain: supabase
topic: quick-reference
---

# Supabase — Quick Reference

### Invariantes

| # | Invariante |
|---|---|
| SB-01 | RLS **habilitado em toda tabela** com dados de usuário — sem exceção |
| SB-02 | Nunca usar `service_role` key no frontend — apenas no servidor |
| SB-03 | JWT verificado antes de qualquer operação em Edge Function |
| SB-04 | pgvector com pre-filter de `user_id` antes da busca semântica |
| SB-05 | `SECURITY DEFINER` em funções SQL apenas quando documentado — risco de bypass RLS |

### Clientes: `anon` vs `service_role`

| Cliente | Onde usar | Respeita RLS |
|---|---|---|
| `anon` key | Frontend / mobile | ✅ Sim |
| `service_role` key | Backend servidor / Edge Functions com cuidado | ❌ Não (bypass total) |

### Setup mínimo no cliente

```typescript
import { createClient } from "@supabase/supabase-js"

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,  // anon key no frontend
)
```
