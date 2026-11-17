# Contrato REST para React Native

## URL base e autenticação

| Ambiente | URL base |
|---|---|
| Swagger e PowerShell no Windows | `http://localhost:5050` |
| Emulador Android oficial na mesma máquina | `http://10.0.2.2:5050`, se o build de desenvolvimento permitir HTTP local. |
| Dispositivo real e avaliação | URL HTTPS acessível ao dispositivo. O endereço definitivo ainda precisa ser definido. |

`10.0.2.2` aponta para o loopback do computador no emulador Android. Android e iOS podem bloquear HTTP sem TLS, por isso use HTTPS para a configuração de avaliação. [Rede do Android Emulator](https://developer.android.com/studio/run/emulator-networking-address) · [Rede no React Native](https://reactnative.dev/docs/network)

Envie `Authorization: Bearer <token>`. Para notificações, o JWT precisa de `sub` e da claim assinada configurada em `Jwt:OracleUserIdClaim`, que representa o `ID_USUARIO` do DBv3. Para ler regras, o token precisa de `CIDADAO` ou `GESTOR`. Escrever regras exige `GESTOR`. O script de desenvolvimento emite tokens somente para uma demonstração local. A configuração de produção depende de confirmar o contrato com Java.

## Chamadas e respostas

| Operação | Resultado |
|---|---|
| `GET /api/notifications/me` | `200` com array de notificações do titular. O array pode estar vazio. Exige Oracle e MongoDB. |
| `PATCH /api/notifications/{id}/read` | `204` sem corpo. Pode ser repetido. `404` se a notificação estiver ausente ou pertencer a outro titular. |
| `GET /api/reminder-rules` | `200` com array de regras. A operação depende apenas do MongoDB. |

```typescript
const headers = {
  Authorization: `Bearer ${accessToken}`,
  Accept: 'application/json',
};

const notifications = await fetch(`${baseUrl}/api/notifications/me`, {headers});
if (notifications.ok) {
  const items = await notifications.json();
  // Renderize o array, inclusive o estado vazio.
}

const read = await fetch(
  `${baseUrl}/api/notifications/${encodeURIComponent(notificationId)}/read`,
  {method: 'PATCH', headers},
);
if (read.status === 204) {
  // Atualize isRead na tela.
}

const rules = await fetch(`${baseUrl}/api/reminder-rules`, {headers});
if (rules.ok) {
  const items = await rules.json();
  // Renderize as regras retornadas pelo ambiente de teste.
}
```

Trate `401` como sessão ausente ou inválida, `403` como acesso negado e `503` como indisponibilidade temporária. Para `429`, use o valor de `Retry-After` em segundos quando presente. Na resposta de notificações, `observedAtUtc` é o instante em que a API observou o status. `appointmentAt` preserva o horário do Oracle sem offset até a confirmação do fuso pelo ambiente.

## Evidência Mobile pendente

1. Configurar a URL base no projeto React Native da equipe e abrir um build em dispositivo ou emulador. Ainda não há execução Mobile registrada nesta solução .NET.
2. Para uma demonstração independente do Oracle, criar uma regra pelo Swagger como `GESTOR` em um MongoDB de teste. Na tela React Native, chamar `GET /api/reminder-rules` com um token `CIDADAO` de teste e mostrar o dado retornado e o status 200.
3. Para demonstrar notificações de ponta a ponta, disponibilizar Oracle DBv3 com conta de leitura, uma consulta fictícia do usuário e um JWT aceito no ambiente com a claim de titularidade. Registrar GET 200 e PATCH 204 no app. Manter tokens fora de capturas de tela e logs.
