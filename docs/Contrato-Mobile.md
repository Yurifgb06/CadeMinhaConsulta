# Contrato REST para React Native

## URL base e autenticação

| Ambiente | URL base |
|---|---|
| Swagger e PowerShell no Windows | `http://localhost:5050` |
| Emulador Android oficial na mesma máquina | `http://10.0.2.2:5050` no build Android de desenvolvimento. |
| Android físico por USB | `http://127.0.0.1:5050` após executar `adb reverse tcp:5050 tcp:5050` com o aparelho de teste conectado. |
| Dispositivo físico pela rede e avaliação | URL HTTPS atribuída à API e acessível ao dispositivo. O endereço definitivo depende do ambiente da equipe. |

`10.0.2.2` aponta para o loopback do computador no Android Emulator. O perfil `CadeMinhaConsulta.Api` abre o Kestrel na porta local 5050. Para Android físico por USB, confirme o aparelho com `adb devices` antes de configurar o encaminhamento. Se o build bloquear HTTP local, restrinja qualquer configuração de cleartext à variante de depuração usada com dados fictícios. Para uso pela rede, publique apenas a API em um endpoint HTTPS com certificado aceito pelo aparelho. O MongoDB continua em loopback. [Rede do Android Emulator](https://developer.android.com/studio/run/emulator-networking-address) · [React Native em dispositivo](https://reactnative.dev/docs/running-on-device) · [Segurança de rede Android](https://developer.android.com/privacy-and-security/security-config)

O `fetch` nativo do React Native não depende de CORS. Se a equipe também executar uma versão Web no navegador, configure no ASP.NET Core apenas as origens necessárias, os métodos usados e o header `Authorization`. Exponha `Retry-After` nessa política caso a tela Web precise lê-lo. [Rede no React Native](https://reactnative.dev/docs/network) · [CORS no ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/cors?view=aspnetcore-10.0)

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

```

Trate `401` como sessão ausente ou inválida, `403` como acesso negado e `503` como indisponibilidade temporária. Para `429`, use o valor de `Retry-After` em segundos quando presente. Na resposta de notificações, `observedAtUtc` é o instante em que a API observou o status. `appointmentAt` preserva o horário do Oracle sem offset até a confirmação do fuso pelo ambiente.

## Exemplo de tela para GET /api/reminder-rules

Passe a URL da tabela e o JWT da sessão ao componente. Em uma demonstração local, gere o token `CIDADAO` pelo [roteiro de desenvolvimento](Roteiro-Demonstracao.md). O issuer de demonstração só é aceito em `Development`. Passe o token em tempo de execução, sem salvá-lo no código, no build ou em logs.

```tsx
import React, {useEffect, useState} from 'react';
import {ActivityIndicator, Button, FlatList, Text, View} from 'react-native';

type Rule = {
  id: string;
  name: string;
  daysBefore: number;
  message: string;
  isActive: boolean;
};

type ScreenState =
  | {kind: 'loading'}
  | {kind: 'error'; message: string}
  | {kind: 'ready'; rules: Rule[]};

function describeError(response: Response): string {
  switch (response.status) {
    case 401:
      return 'Sessão inválida ou expirada. Entre novamente.';
    case 403:
      return 'Seu perfil não pode consultar as regras.';
    case 429: {
      const seconds = response.headers.get('Retry-After');
      return seconds
        ? 'Muitas tentativas. Aguarde ' + seconds + ' s.'
        : 'Muitas tentativas. Aguarde antes de tentar novamente.';
    }
    case 503:
      return 'Serviço temporariamente indisponível.';
    default:
      return 'Falha ao consultar as regras (HTTP ' + response.status + ').';
  }
}

export function ReminderRulesScreen({
  baseUrl,
  accessToken,
}: {
  baseUrl: string;
  accessToken: string;
}) {
  const [state, setState] = useState<ScreenState>({kind: 'loading'});
  const [attempt, setAttempt] = useState(0);

  useEffect(() => {
    if (!accessToken) {
      setState({kind: 'error', message: 'Entre para consultar as regras.'});
      return;
    }

    const request = new AbortController();
    const endpoint = baseUrl.replace(/\/+$/, '') + '/api/reminder-rules';
    setState({kind: 'loading'});

    async function load() {
      try {
        const response = await fetch(endpoint, {
          headers: {
            Authorization: 'Bearer ' + accessToken,
            Accept: 'application/json',
          },
          signal: request.signal,
        });
        const result: ScreenState = response.ok
          ? {kind: 'ready', rules: (await response.json()) as Rule[]}
          : {kind: 'error', message: describeError(response)};
        if (!request.signal.aborted) setState(result);
      } catch {
        if (!request.signal.aborted) {
          setState({kind: 'error', message: 'Falha de rede ao consultar as regras.'});
        }
      }
    }

    void load();
    return () => request.abort();
  }, [baseUrl, accessToken, attempt]);

  return (
    <View style={{flex: 1, padding: 16}}>
      <Text>Regras de lembrete</Text>
      <Button
        title="Atualizar"
        onPress={() => setAttempt(value => value + 1)}
        disabled={state.kind === 'loading'}
      />
      {state.kind === 'loading' && <ActivityIndicator />}
      {state.kind === 'error' && <Text>{state.message}</Text>}
      {state.kind === 'ready' && (
        <FlatList
          data={state.rules}
          keyExtractor={rule => rule.id}
          ListEmptyComponent={<Text>Nenhuma regra cadastrada.</Text>}
          renderItem={({item}) => (
            <View style={{marginTop: 12}}>
              <Text>{item.name}</Text>
              <Text>{item.daysBefore} dia(s) antes da consulta</Text>
              <Text>{item.message}</Text>
              <Text>{item.isActive ? 'Ativa' : 'Inativa'}</Text>
            </View>
          )}
        />
      )}
    </View>
  );
}
```

## Evidência Mobile pendente

1. Configurar a URL base no projeto React Native da equipe e abrir um build em dispositivo ou emulador. Ainda não há execução Mobile registrada nesta solução .NET.
2. Para uma demonstração independente do Oracle, criar uma regra pelo Swagger como `GESTOR` em um MongoDB de teste. Na tela React Native, chamar `GET /api/reminder-rules` com um token `CIDADAO` de teste e mostrar o dado retornado e o status 200.
3. Repetir no React Native o fluxo já validado no Kestrel com Oracle e MongoDB reais. Usar uma consulta fictícia do DBv3 e um JWT aceito no ambiente com a claim de titularidade. Registrar GET 200 e PATCH 204 no app. Manter tokens fora de capturas de tela e logs.
