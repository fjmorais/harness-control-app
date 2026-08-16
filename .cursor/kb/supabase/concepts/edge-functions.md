# Edge Functions (Deno Serverless)

## Quando usar Edge Functions

| Use caso | Edge Function | Backend próprio |
|---|---|---|
| Webhook de pagamento (Stripe, etc.) | ✅ | ✅ |
| Lógica simples pós-auth | ✅ | — |
| Envio de email/SMS disparado por evento | ✅ | — |
| Processamento pesado / ML | — | ✅ |
| Grafo LangGraph complexo | — | ✅ |
| Acesso a DB com lógica de negócio complexa | — | ✅ |

## Estrutura básica

```typescript
// supabase/functions/minha-funcao/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
}

serve(async (req: Request) => {
  // CORS preflight
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders })
  }

  try {
    // Verificar JWT do usuário
    const authHeader = req.headers.get("Authorization")
    if (!authHeader) {
      return new Response(JSON.stringify({ error: "Unauthorized" }), {
        status: 401, headers: { ...corsHeaders, "Content-Type": "application/json" },
      })
    }

    // Cliente com contexto do usuário (respeita RLS)
    const supabase = createClient(
      Deno.env.get("SUPABASE_URL")!,
      Deno.env.get("SUPABASE_ANON_KEY")!,
      { global: { headers: { Authorization: authHeader } } },
    )

    const { data: { user } } = await supabase.auth.getUser()
    if (!user) return new Response(JSON.stringify({ error: "Unauthorized" }), { status: 401 })

    const body = await req.json()

    // Lógica da função
    const result = await processRequest(supabase, user, body)

    return new Response(JSON.stringify(result), {
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    })

  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" },
    })
  }
})
```

## Secrets — nunca hardcoded

```bash
# Definir segredo via CLI
supabase secrets set OPENAI_API_KEY=sk-...
supabase secrets set STRIPE_WEBHOOK_SECRET=whsec-...

# Listar segredos
supabase secrets list
```

```typescript
// Usar na função
const openAIKey = Deno.env.get("OPENAI_API_KEY")!
```

## Cliente service_role na Edge Function (com cuidado)

```typescript
// Usar service_role apenas para operações que precisam bypass de RLS
// Sempre documentar o motivo
const adminClient = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!,  // bypass RLS — documente por quê
)
```

## Deploy e invocação

```bash
# Deploy
supabase functions deploy minha-funcao

# Invocar localmente (dev)
supabase functions serve minha-funcao --env-file .env.local

# Invocar via CLI
curl -L -X POST \
  "https://<project>.supabase.co/functions/v1/minha-funcao" \
  -H "Authorization: Bearer <ANON_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"campo": "valor"}'
```

```typescript
// Invocar do frontend
const { data, error } = await supabase.functions.invoke("minha-funcao", {
  body: { campo: "valor" },
})
```

## Webhook pattern (ex: Stripe)

```typescript
serve(async (req) => {
  const signature = req.headers.get("stripe-signature")!
  const body = await req.text()

  // Verificar assinatura antes de qualquer processamento
  let event
  try {
    event = stripe.webhooks.constructEvent(body, signature, Deno.env.get("STRIPE_WEBHOOK_SECRET")!)
  } catch {
    return new Response("Invalid signature", { status: 400 })
  }

  // Processar evento
  if (event.type === "payment_intent.succeeded") {
    const adminClient = createClient(url, serviceRoleKey)
    await adminClient.from("payments").insert({ ... })
  }

  return new Response(JSON.stringify({ received: true }), { status: 200 })
})
```

## Anti-padrões

```typescript
// ERRADO: sem verificar JWT
serve(async (req) => {
  const body = await req.json()
  await db.insert(body)  // qualquer um pode inserir dados!
})

// ERRADO: secrets hardcoded
const apiKey = "sk-abc123..."  // exposto no código

// ERRADO: processar payload de webhook sem verificar assinatura
const event = await req.json()  // pode ser forjado
```
