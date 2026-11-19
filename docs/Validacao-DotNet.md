# Validação do backend .NET

## Revisão profissional de 18/11/2026

O código foi conferido contra as páginas 23–28 do edital. A revisão abrangeu os dois projetos, os três scripts PowerShell, endpoints, contratos, persistência, autenticação, filtros, tratamento de falhas e os testes. Os arquivos e os cenários de cada critério estão relacionados no [roteiro da banca](Roteiro-Demonstracao.md).

| Verificação executada | Resultado desta revisão |
|---|---|
| Restore e build de `CadeMinhaConsulta.slnx` | Sucesso; zero avisos e zero erros no build. |
| xUnit com MongoDB real em banco dedicado | 47 aprovados, zero falhas e zero pulados. O total permanece igual ao de referência. |
| Auditoria NuGet direta e transitiva | Nenhum pacote vulnerável reportado pelas fontes configuradas na consulta. |
| Swagger UI e OpenAPI no Kestrel | HTTP 200; 10 operações documentadas, com saúde pública e Bearer nas rotas protegidas. |
| Health Checks reais | Liveness 200; readiness 200, Oracle e MongoDB Healthy. |
| CRUD e erros de entrada | 200, 201, 204 e 404 no MongoDB real. Entrada inválida, ausência de chave e corpo JSON nulo retornaram 400. |
| Autorização | GET anônimo 401; escrita com CIDADAO 403; diagnóstico com token GESTOR 204. |
| Idempotência e rate limiting | Replay com 201 e corpo/Location originais; conflito 409; excesso 429 com Retry-After. Os testes MongoDB verificaram concorrência e reinício do host. |
| Notificações com Oracle real | GET 200 com consulta fictícia do titular. PATCH próprio 204 repetido; PATCH alheio 404; leitura persistida no MongoDB isolado. |
| Configuração local | User Secrets permaneceu idêntico antes e depois da validação HTTP. |

### Correções e limpeza

- O README foi refeito com apresentação CitizenTech, badges, arquitetura Mermaid, pré-requisitos, configuração, contratos, erros HTTP, exemplos JSON, testes e rubrica.
- Foram corrigidos registros de entrega desatualizados que ainda diziam que o repositório estava sem remoto e o link aguardava publicação.
- Não foi identificada falha funcional reproduzível que exigisse modificar os fontes, testes ou dependências nesta revisão. A arquitetura e o comportamento existente foram preservados.
- Nenhum arquivo versionado foi excluído. A documentação técnica e a tela de referência Mobile têm links e finalidade de entrega. Temporários, binários, backups e dados locais continuam fora do Git.
- O README não contém as seções de declaração de IA ou histórico de renomeação. Os identificadores necessários ao ambiente são documentados como configuração técnica.
- A triagem de segurança examinou os arquivos atuais e 69 blobs únicos em 6 commits acessíveis, sem padrões de tokens, chaves privadas ou credenciais. As exclusões de User Secrets, dados MongoDB e backups Oracle foram verificadas. A revisão manteve todos os testes, scripts e documentos de evidência.

### Histórico remoto e branch local

A inspeção remota encontrou `main` e `docs/revisao-final-edital-dotnet` em `de9551d`. Os dois commits posteriores a `acd74d4` modificavam o README. A árvore limpa foi atualizada por fast-forward antes da revisão. O trabalho atual está em `review/fechamento-profissional-dotnet`, preservando a branch de revisão existente. A publicação desta revisão depende de autorização.

## Ambiente e verificação

- Visual Studio Community 2026 com workload web, SDK .NET 11.0.100 e runtime ASP.NET Core 10.0.15. A solução continua compilando com alvo `net10.0`.
- MongoDB Community 8.0.38 portátil, extraído do ZIP oficial e verificado pelo SHA-256 publicado pelo fornecedor. O processo roda em loopback e usa `.local/mongodb/data`. O banco de desenvolvimento está em User Secrets. Compass, mongosh e serviço do Windows são dispensáveis para este roteiro.
- O processo MongoDB foi desligado e iniciado por um PowerShell novo. O script reutilizou os dados e recusou uma segunda instância na mesma porta. O arquivo User Secrets permaneceu idêntico byte a byte. A verificação ocorreu sem reiniciar o Windows e sem configurar início automático na entrada da sessão.
- Após a renomeação, `dotnet restore`, `dotnet build` e `dotnet test` passaram na nova solução, com **47 aprovados, zero falhas e zero pulados**. O build teve zero avisos. Na auditoria anterior, a consulta do NuGet não reportou pacotes diretos ou transitivos vulneráveis nas fontes configuradas. As dependências não mudaram nesta revisão.
- Dois testes xUnit usaram um MongoDB real. Um testou regras de lembrete, e o outro testou notificações com consultas fictícias fornecidas pela suíte. Os demais cenários cobrem unidades e HTTP em memória, assinaturas JWT, BSON e SQL gerado.
- A API foi iniciada pelo perfil `CadeMinhaConsulta.Api`. O Kestrel recebeu requisições HTTP em todos os endpoints. O MongoDB foi desligado normalmente e reiniciado no mesmo diretório de dados. Após reiniciar o servidor e o Kestrel, conferimos o estado de leitura de uma notificação e a resposta original de uma criação de regra.
- Na revisão da renomeação, o Visual Studio abriu `CadeMinhaConsulta.slnx`. No Kestrel, Swagger exibiu o novo título, o diagnóstico retornou 401 sem token e 204 com JWT de desenvolvimento, a leitura MongoDB retornou 200 e os User Secrets permaneceram idênticos. A prontidão retornou 503 com MongoDB saudável e Oracle não saudável. O login Oracle retornou `ORA-01017` no check de leitura. Essa revisão não implantou objetos Oracle.
- Após a atualização da senha, a implantação do DBv3 no Oracle FIAP foi concluída em `RM561450`. No Kestrel, `/health/live` e `/health/ready` retornaram 200 com Oracle e MongoDB saudáveis.
- No fechamento, `restore`, `build` e `test` passaram novamente na solução. Resultado de **47 testes aprovados, zero falhas e zero pulados**, com as duas integrações MongoDB reais. O build teve zero avisos e zero erros.
- Na checagem final pelo Kestrel, liveness, readiness e Swagger retornaram 200. Com uma regra fictícia em MongoDB isolado, confirmei CRUD, erros 400/401/403/404, replay com 201 idêntico, conflito 409 e 429 com `Retry-After`. `GET /api/reminder-rules` retornou 200 com JWT `CIDADAO` e o corpo da regra foi conferido. A chamada foi feita por HTTP no backend e ainda precisa ser repetida no aplicativo React Native.
- Na preparação Mobile, o cliente HTTP Node.js confirmou `GET /api/reminder-rules` 200 com JWT `CIDADAO` e os campos esperados da regra. O mesmo token de demonstração recebeu 401 em `Production`. A suíte xUnit passou novamente com 47 aprovados. A [tela de exemplo](examples/ReminderRulesScreen.tsx) e os passos para obter a evidência no app estão no [contrato Mobile](Contrato-Mobile.md).

## Evidência por critério FIAP

| Requisito | Implementação | Teste executado | Evidência | Pendência | Risco de desconto |
|---|---|---|---|---|---|
| Minimal API RESTful e Swagger · 20 pts | CRUD de regras, notificações, validação, respostas HTTP e OpenAPI para rotas públicas e autenticadas. | Kestrel com Oracle e MongoDB reais e asserts no OpenAPI. | CRUD e notificações em HTTP, incluindo GET que coincide com SELECTs por titular e PATCH 204. Rotas de saúde públicas no Swagger. | Executar o fluxo no React Native. | Médio se a demonstração ficar restrita ao Swagger. |
| JWT e autorização · 20 pts | Assinatura RS256, emissor, audiência, expiração, perfis e isolamento por titular. | JWT de desenvolvimento no Kestrel e testes xUnit com JWT assinado. | 401 anônimo, 403 para escrita com CIDADAO e escrita com GESTOR. | Confirmar contrato de produção com o Java e testar um JWT real. | Médio para a integração Mobile autenticada. |
| Idempotência · 15 pts | ID estável e gravação atômica no MongoDB para regras e notificações. | Concorrência nos dois testes com MongoDB real. Replay via Kestrel antes e depois de reiniciar o host e o banco. | Um único registro por chave, corpo e Location preservados, 409 para conteúdo diferente. | Nenhuma no cenário validado. | Baixo. |
| Rate Limiting · 15 pts | Cota configurável por identidade e `Retry-After`. | Excesso de chamadas no Kestrel e teste de isolamento por usuário. | 429 com `Retry-After`. Chamadas anônimas retornaram 401. | Nenhuma no cenário validado. | Baixo. |
| xUnit · 20 pts | Testes unitários, HTTP em memória e integrações com MongoDB. | `dotnet test` com banco real dedicado. Um teste HTTP independente comparou Oracle real com a API. | 47 aprovados, zero falhas e zero pulados, incluindo dois cenários reais MongoDB. O xUnit de notificações usa consultas fornecidas pela suíte. | O cenário com Oracle FIAP foi executado separadamente no Kestrel. | Baixo. |
| Health Checks · 10 pts | Liveness público e readiness com estado separado Oracle/MongoDB. Qualquer falha ou degradação retorna 503. | Kestrel com ambos os bancos reais. Auditorias anteriores testaram falha e recuperação do MongoDB e configuração Oracle inválida. | Liveness 200 e readiness 200 com ambos `Healthy`. O readiness 503 também foi observado com dependência indisponível. | Manter ambos os bancos acessíveis durante a apresentação. | Baixo se as dependências estiverem disponíveis na demonstração. |

## Oracle → .NET → MongoDB

O DBv3 tem 13 tabelas ativas. O Kestrel retornou, respectivamente, uma e duas notificações de dois titulares fictícios. Os IDs das consultas, os status e as datas coincidiram com SELECTs diretos no Oracle filtrados pelos mesmos titulares.

- GET sem autenticação retornou 401. O GET repetido com o primeiro titular retornou o mesmo corpo e os mesmos IDs.
- PATCH com o token do outro titular retornou 404 e preservou o estado da notificação. Dois PATCHs com o token do titular retornaram 204.
- Uma consulta direta ao MongoDB isolado confirmou três documentos, um único registro por notificação, índice `_id_` e o estado de leitura persistido. Os User Secrets permaneceram idênticos durante o teste.

Veja os [metadados e a preparação do DBv3](Oracle-DBv3.md). O teste de ponta a ponta usou os dois bancos reais e JWTs de desenvolvimento.

## Demonstração

O [roteiro da banca](Roteiro-Demonstracao.md) relaciona cada critério aos arquivos implementados, aos testes existentes e aos passos para demonstrá-lo. Também prepara respostas sobre arquitetura, autenticação, EF Core e idempotência. O [contrato React Native](Contrato-Mobile.md) inclui URL por ambiente e uma tela de exemplo para listar regras.

A evidência de consumo pelo aplicativo depende da execução no projeto da equipe Mobile. A verificação HTTP no Kestrel comprova o comportamento da API .NET. A chamada e a captura da tela React Native continuam pendentes.

## Conferência da entrega

- O repositório público é [Yurifgb06/CadeMinhaConsulta](https://github.com/Yurifgb06/CadeMinhaConsulta). O remoto `origin` está configurado. A revisão profissional está registrada no [PR #1](https://github.com/Yurifgb06/CadeMinhaConsulta/pull/1); seu histórico permite conferir a publicação e o merge.
- O ZIP definitivo contém somente `Identificacao-do-grupo.txt`, com os cinco integrantes, e `Link-do-GitHub.txt`, com a URL pública verdadeira. O link aponta para o conteúdo da branch `main` no GitHub.
- O ZIP técnico anterior à renomeação foi extraído e compilado. Na pasta de extração, o Controle de Aplicativos do Windows bloqueou o carregamento da DLL da API com `0x800711C7`. Os 47 testes aprovados correspondem à execução no workspace principal.
- Exclua do ZIP as pastas `.local`, `.vs`, `bin`, `obj` e quaisquer arquivos com segredos. O envio pelo Portal FIAP está previsto até **19/11/2026 às 23h55**.

## Riscos e critérios de apresentação

| Item do edital | Consequência indicada | Situação e ação da equipe |
|---|---|---|
| Prazo ou envio fora do Portal | Desconto de 30 pontos | Enviar até 19/11/2026 às 23h55. |
| Arquivo corrompido, incompleto ou sem identificação | Desconto de 100 pontos | Conferir o ZIP final e manter o TXT com os cinco integrantes. |
| Solução sem relação com CitizenTech | Desconto de 100 pontos | Apresentar o problema das consultas públicas e a utilidade ao cidadão. |
| API que não executa ou falha impedindo seu uso | Desconto de 20 pontos | Backend validado; garantir acesso aos bancos e tokens válidos na sessão da banca. |
| Falta de organização mínima ou credenciais expostas | Desconto de 20 pontos | Árvore, histórico e exclusões revisados; User Secrets e dados locais fora do Git. |
| Consumo .NET pelo Mobile, citado na página 23 | A página não separa uma pontuação própria | Execução no React Native pendente da equipe Mobile. O teste HTTP não foi apresentado como execução no app. |

A página 28 chama a apresentação .NET de online, entre 23 e 27/11, valendo 30% da nota. Dentro da apresentação, o slide distribui 20 pontos para funcionamento, 20 para questões de estrutura, 20 para decisões de design, 20 para fluxos principais, 10 para componentes .NET e 10 para CitizenTech. O roteiro técnico cobre esses tópicos. A interoperabilidade JWT de produção com Java continua dependente do contrato da equipe; a autenticação local validada funciona como demonstração em Development.
