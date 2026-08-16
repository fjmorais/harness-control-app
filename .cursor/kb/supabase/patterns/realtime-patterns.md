# Realtime Patterns

## Quando usar Realtime

- Notificações em tempo real (novo caso, nova mensagem)
- Dashboard que atualiza sem polling
- Colaboração simultânea (múltiplos usuários no mesmo recurso)
- Presença (quem está online)

## Subscribe a mudanças de tabela

```typescript
// Escutar INSERT em casos da org do usuário
const channel = supabase
  .channel("casos-org")
  .on(
    "postgres_changes",
    {
      event: "INSERT",
      schema: "public",
      table: "casos",
      filter: `org_id=eq.${orgId}`,  // filtro server-side
    },
    (payload) => {
      console.log("Novo caso:", payload.new)
      setCasos(prev => [...prev, payload.new as Caso])
    }
  )
  .subscribe()

// Limpar ao desmontar componente
return () => { supabase.removeChannel(channel) }
```

## Múltiplos eventos no mesmo canal

```typescript
const channel = supabase
  .channel("casos-realtime")
  .on("postgres_changes", { event: "INSERT", schema: "public", table: "casos" },
    (payload) => onInsert(payload.new))
  .on("postgres_changes", { event: "UPDATE", schema: "public", table: "casos" },
    (payload) => onUpdate(payload.new))
  .on("postgres_changes", { event: "DELETE", schema: "public", table: "casos" },
    (payload) => onDelete(payload.old))
  .subscribe()
```

## Presença — quem está online

```typescript
const channel = supabase.channel("sala-de-investigacao")

// Entrar com presença
channel
  .on("presence", { event: "sync" }, () => {
    const state = channel.presenceState()
    setOnlineUsers(Object.values(state).flat())
  })
  .on("presence", { event: "join" }, ({ newPresences }) => {
    console.log("Entrou:", newPresences)
  })
  .on("presence", { event: "leave" }, ({ leftPresences }) => {
    console.log("Saiu:", leftPresences)
  })
  .subscribe(async (status) => {
    if (status === "SUBSCRIBED") {
      await channel.track({ user_id: userId, email: userEmail, online_at: new Date().toISOString() })
    }
  })

// Sair
await channel.untrack()
```

## Broadcast — mensagem point-to-point

```typescript
// Sender
const channel = supabase.channel("notificacoes")
await channel.send({
  type: "broadcast",
  event: "novo-comentario",
  payload: { caso_id: "123", comentario: "..." },
})

// Receiver
channel.on("broadcast", { event: "novo-comentario" }, ({ payload }) => {
  showNotification(payload)
})
channel.subscribe()
```

## Hook React reutilizável

```typescript
function useCasosRealtime(orgId: string, onUpdate: (caso: Caso) => void) {
  useEffect(() => {
    const channel = supabase
      .channel(`casos-${orgId}`)
      .on("postgres_changes", {
        event: "*",
        schema: "public",
        table: "casos",
        filter: `org_id=eq.${orgId}`,
      }, (payload) => onUpdate(payload.new as Caso))
      .subscribe()

    return () => { supabase.removeChannel(channel) }
  }, [orgId, onUpdate])
}
```

## Limitações e cuidados

| Limitação | Mitigação |
|---|---|
| RLS não se aplica ao Realtime por padrão | Habilitar: `supabase realtime enable-row-level-security` ou usar `filter` no subscribe |
| Máximo de canais simultâneos | Usar 1 canal com múltiplos eventos em vez de N canais |
| Sem guarantee de entrega | Combinar com polling de fallback para dados críticos |
| Payload limitado (~1 MB) | Enviar apenas ID no broadcast; buscar dados completos separado |
