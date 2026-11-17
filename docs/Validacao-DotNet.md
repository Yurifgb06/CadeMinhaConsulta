# Validação do backend .NET

## Ambiente e verificação

- Visual Studio Community 2026 com workload web, SDK .NET 11.0.100 e runtime ASP.NET Core 10.0.15. A solução continua compilando com alvo `net10.0`.
- MongoDB Community 8.0.38 portátil, extraído do ZIP oficial e verificado pelo SHA-256 publicado pelo fornecedor. O processo roda em loopback e usa `.local/mongodb/data`. O banco de desenvolvimento está em User Secrets. Compass, mongosh e serviço do Windows são dispensáveis para este roteiro.
- O processo MongoDB foi desligado e iniciado por um PowerShell novo. O script reutilizou os dados e recusou uma segunda instância na mesma porta. O arquivo User Secrets permaneceu idêntico byte a byte. A verificação ocorreu sem reiniciar o Windows e sem configurar início automático na entrada da sessão.
- `dotnet --info`, `dotnet restore`, `dotnet build` e `dotnet test` executados na revisão final, com **47 aprovados, zero falhas e zero pulados**. O build teve zero avisos. Na auditoria anterior, a consulta do NuGet não reportou pacotes diretos ou transitivos vulneráveis nas fontes configuradas. As dependências não mudaram nesta revisão.
- Dois testes xUnit usaram um MongoDB real. Um testou regras de lembrete, e o outro testou notificações com consultas fictícias fornecidas pela suíte. Os demais cenários cobrem unidades e HTTP em memória, assinaturas JWT, BSON e SQL gerado.
- A API foi iniciada pelo perfil `VagaCerta.Api`. O Kestrel recebeu requisições HTTP em todos os endpoints. O MongoDB foi desligado normalmente e reiniciado no mesmo diretório de dados. Após reiniciar o servidor e o Kestrel, conferimos o estado de leitura de uma notificação e a resposta original de uma criação de regra.

## Evidência por critério FIAP

| Requisito | Implementação | Teste executado | Evidência | Pendência | Risco de desconto |
|---|---|---|---|---|---|
| Minimal API RESTful e Swagger · 20 pts | CRUD de regras, notificações, validação, respostas HTTP e OpenAPI para rotas públicas e autenticadas. | Kestrel com MongoDB real e asserts no OpenAPI. | CRUD completo, leitura marcada, 200/201/204/400/404. Rotas públicas de saúde visíveis no Swagger sem exigir Bearer. | GET de notificações com Oracle real e chamada pelo Mobile. | Médio se a demonstração ficar restrita ao Swagger. |
| JWT e autorização · 20 pts | Assinatura RS256, emissor, audiência, expiração, perfis e isolamento por titular. | JWT de desenvolvimento no Kestrel e testes xUnit com JWT assinado. | 401 anônimo, 403 para escrita com CIDADAO e escrita com GESTOR. | Confirmar contrato de produção com o Java e testar um JWT real. | Médio para a integração Mobile autenticada. |
| Idempotência · 15 pts | ID estável e gravação atômica no MongoDB para regras e notificações. | Concorrência nos dois testes com MongoDB real. Replay via Kestrel antes e depois de reiniciar o host e o banco. | Um único registro por chave, corpo e Location preservados, 409 para conteúdo diferente. | Nenhuma no cenário validado. | Baixo. |
| Rate Limiting · 15 pts | Cota configurável por identidade e `Retry-After`. | Excesso de chamadas no Kestrel e teste de isolamento por usuário. | 429 com `Retry-After`. Chamadas anônimas retornaram 401. | Nenhuma no cenário validado. | Baixo. |
| xUnit · 20 pts | Testes unitários, HTTP em memória e integrações com MongoDB. | `dotnet test` com banco real dedicado configurado. | 47 aprovados, zero falhas e zero pulados, incluindo dois cenários reais MongoDB. | Leitura real do DBv3. O cenário de notificações usou consultas fictícias fornecidas pelo teste. | Baixo para o critério xUnit, com o limite de integração Oracle explícito. |
| Health Checks · 10 pts | Liveness público e readiness com estado separado Oracle/MongoDB. Qualquer falha ou degradação retorna 503. | Kestrel com MongoDB real, Oracle indisponível e dois testes com estados controlados para 200 e 503. A auditoria anterior exercitou a parada e a recuperação do MongoDB. | Liveness 200. Readiness 503 com `mongo: Healthy` e `oracle: Unhealthy`. 200 somente com ambos em estado saudável no teste controlado. | Demonstrar readiness 200 com os dois bancos reais ativos. | Alto se a banca exigir a comprovação positiva das duas dependências. |

A porta 1521 do Oracle FIAP respondeu por TCP. O verificador local parou por ausência de `ConnectionStrings:Oracle`. Sem login autenticado, SID/service name, inventário do schema, contagens e compilação continuam pendentes. O ZIP corrigido foi extraído em área ignorada, e o DDL de criação foi separado do trecho de limpeza. O teste de mapeamento confirma o SQL do SELECT de `TB_GS_CONSULTA` e o filtro por titular. O readiness e o GET de notificações retornaram 503. Nenhum SQL foi executado no Oracle. Veja a [preparação DBv3](Oracle-DBv3.md).

## Roteiro de demonstração

1. Mostre Swagger, `/health/live` e `/health/ready`. Explique o estado de cada banco no JSON.
2. Sem token, liste regras e mostre 401. Autorize CIDADAO, liste novamente e tente um POST para mostrar 403.
3. Autorize GESTOR, envie um POST inválido e mostre 400. Crie uma regra com chave inédita e use o `Location` para consultar e editar.
4. Reenvie o POST original e confira o mesmo 201, `Location` e corpo. Reutilize a chave com outros dados válidos para mostrar 409. Exclua a regra e confira 204 e depois 404 no GET.
5. Configure o limite de dois pedidos por minuto e mostre o 429 com `Retry-After`.
6. Execute a suíte com MongoDB real como no [README](../README.md). No Mobile, siga o [contrato React Native](Contrato-Mobile.md) para ler uma regra com CIDADAO pelo endpoint que depende apenas do MongoDB. Com Oracle disponível e uma consulta fictícia no DBv3 de teste, acrescente a listagem e o PATCH de notificações.

## Conferência da entrega

- A pasta de trabalho contém a solução, código, testes, documentação e o primeiro commit local na branch `main`, sem remoto. A revisão do conjunto versionável procurou credenciais em texto, tokens, chaves privadas e binários, e conferiu as exclusões com `git check-ignore`. A entrega ainda precisa de um repositório GitHub acessível ao professor.
- O TXT de identificação foi preparado na pasta local de entrega, ignorada pelo Git. O ZIP de submissão aguarda o link real do GitHub. Inclua o TXT com RM, nome completo e turma de cada integrante e o PowerPoint, se houver. A ausência do TXT pode receber o desconto de 100 pontos indicado pelo edital. A equipe precisa revisar a declaração de uso de IA no README.
- O ZIP técnico local foi extraído e compilado. Na pasta de extração, o Controle de Aplicativos do Windows bloqueou o carregamento de `VagaCerta.Api.dll` com `0x800711C7`. Os 47 testes aprovados correspondem à execução no workspace principal.
- Exclua do ZIP as pastas `.local`, `.vs`, `bin`, `obj` e quaisquer arquivos com segredos. O envio pelo Portal FIAP está previsto até **19/11/2026 às 23h55**.
