# Oracle DBv3

## Conexão local

A API lê `ConnectionStrings:Oracle` e `Oracle:Schema` do projeto `CadeMinhaConsulta.Api`. O provider existente, `Oracle.EntityFrameworkCore` com ODP.NET Core gerenciado, faz a conexão direta. O Oracle Client nativo não é necessário para essa configuração. [Configuração do ODP.NET Core](https://docs.oracle.com/en/database/oracle/oracle-database/21/odpnt/InstallCoreConfiguration.html)

Use **Gerenciar Segredos do Usuário** no Visual Studio ou execute `tools/Set-OracleLocalSecrets.ps1` interativamente no PowerShell 7. Informe o host, identificador, usuário e schema pelos parâmetros. A senha é solicitada pelo script e gravada em User Secrets por entrada padrão, fora dos argumentos do processo. Use apenas os dados fornecidos pelo proprietário.

O modo `ServiceName` define `CONNECT_DATA=(SERVICE_NAME=...)`. O modo `Sid` define `CONNECT_DATA=(SID=...)`. A porta TCP 1521 do Oracle FIAP respondeu nesta execução. O registro de `ORCL` no listener e a identidade da sessão ainda precisam de uma conexão autenticada. [Conexão ao Oracle Database](https://docs.oracle.com/en/database/oracle/oracle-database/26/odpnt/featConnecting.html)

Use uma conta de leitura da API quando houver uma credencial separada. A conta acadêmica de implantação pode ter privilégios de escrita. No código da aplicação, o EF Core trata `TB_GS_CONSULTA` como uma origem de leitura sem rastreamento de entidade, `SaveChanges` é bloqueado e o fluxo de notificações usa SELECT com filtro por titular. O projeto não aplica migrations.

## Inspeção do pacote e ordem de implantação

O ZIP corrigido contém 13 declarações `CREATE TABLE`, 9 chaves primárias, 17 chaves estrangeiras, 9 sequences e 9 triggers de IDs no DDL, além de 4 triggers de auditoria e data, 2 packages com especificação e corpo, e povoamento em 3 blocos PL/SQL. A versão do servidor FIAP e a compilação continuam dependentes do acesso autenticado.

1. Confirme `SESSION_USER` e `CURRENT_SCHEMA` na mesma sessão Oracle a ser usada para operações de implantação. Ambos devem ser exatamente o schema autorizado. Registre nomes, tipos e estados dos objetos próprios e preserve uma exportação, quando viável, em armazenamento local fora do Git.
2. Limpe apenas os objetos próprios inventariados. Trate dependências entre tabelas do próprio schema e respeite erros do Oracle que impeçam um DROP. A limpeza inicial do arquivo do ZIP contém `CASCADE CONSTRAINTS` e deve ser excluída da etapa de criação, pois uma referência de outro schema não deve ser removida.
3. Execute o trecho de criação do DDL, seguido por `TRIGGERS AUDITORIA E REGRA DE DATA FUTURA.sql`, `INSERCAO DE DADOS.sql`, `PACKAGE CONSULTA DADOS.sql` e `PACKAGE GESTAO DE CONSULTAS.sql`. O povoamento contém commits e só deve ser executado uma vez sobre o schema recém-criado.
4. Compare as tabelas em `USER_TABLES` com as 13 esperadas. Consulte `USER_CONSTRAINTS`, `USER_OBJECTS` e `USER_ERRORS`, registre contagens e valide os relacionamentos usando SELECT. Para testar o agendamento no banco de teste, use o overload com ID de saída sob transação controlada e rollback. O overload manual efetua commit.
5. Com MongoDB e Oracle saudáveis, confirme `/health/ready` 200. Use um JWT de desenvolvimento vinculado a uma consulta fictícia para testar o GET e a marcação de leitura no MongoDB, incluindo repetição e uma tentativa com outro titular.
