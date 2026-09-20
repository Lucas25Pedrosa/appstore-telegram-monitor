# appstore-telegram-monitor

Camada pública de **orquestração do App Store Monitor e do Emoji Manager**. Este repositório contém workflows do GitHub Actions; a lógica principal do monitor e dos emojis fica no repositório privado `appstore-telegram-core`, enquanto configurações compartilhadas ficam em `ipa-shared-config-core`.

O Cloudflare Worker que recebe comandos do Telegram é um componente externo ao conteúdo versionado neste repositório.

## Arquitetura

```text
Cloudflare Worker / Telegram
          ↓
appstore-telegram-monitor
          ├── checkout appstore-telegram-core
          ├── checkout ipa-shared-config-core
          ↓
GitHub Actions
          ├── App Store Monitor
          ├── Emoji Manager
          ├── testes Azure
          └── teste Telegram
```

## Estrutura do repositório

| Arquivo | Função |
| --- | --- |
| `.github/workflows/appstore-monitor.yml` | Workflow principal do monitor de versões da App Store. |
| `.github/workflows/emoji-manager.yml` | Workflow multifuncional para backup, criação, atualização, migração e remoção de custom emojis. |
| `.github/workflows/azure-translator-test.yml` | Teste isolado da ponte detector local → Azure Translator. |
| `.github/workflows/azure-translator-metrics-test.yml` | Teste da autenticação OIDC e consulta das métricas oficiais do Azure Translator. |
| `.github/workflows/test-r2-telegram-bot.yml` | Teste de conectividade/envio do bot Telegram usando o Core privado. O nome “r2” é legado. |
| `README.md` | Esta documentação. |

## `appstore-monitor.yml`

Workflow de produção do monitor.

### Fluxo

1. Espera brevemente para detectar execuções simultâneas.
2. Cancela logicamente a execução atual se já houver run anterior em fila/em andamento.
3. Faz checkout de `appstore-telegram-core`.
4. Faz checkout de `ipa-shared-config-core`.
5. Instala Python/dependências.
6. Sincroniza `versions.json` com os IDs presentes em `apps.json`.
7. Executa `monitor.py`.
8. Persiste alterações do Emoji Manager no shared config.
9. Consulta métricas do Azure Translator via OIDC quando disponível.
10. Atualiza estado/logs/relatórios no Core privado.

### Por que sincronizar `versions.json`

Se um app sai de `apps.json`, sua entrada antiga não deve continuar indefinidamente no estado do monitor. O workflow remove IDs não monitorados de `versions.json` antes da execução principal.

### Concorrência

A verificação inicial reduz risco de duas execuções do monitor escreverem estado ao mesmo tempo.

## `emoji-manager.yml`

Workflow administrativo do sistema de emojis.

Modos atuais incluem:

- `bootstrap`
- `app`
- `preview`
- `telegram-test`
- `bulk-telegram`
- `version-check-test`
- `version-change-dry-run`
- `add-app`
- `delete-app`
- `visual-hash-backfill`

### Operações principais

#### Backup/preview

Usa `emoji_manager.py` do Core para:

- consultar o ícone oficial;
- preservar os bytes originais;
- calcular SHA-256;
- gerar PNG mascarado quando solicitado.

#### Atualização por versão

Usa `emoji_version_update.py` para comparar o ícone atual com a referência visual e só substituir o custom emoji quando o desenho realmente mudou.

#### `/add`

O modo `add-app` usa duas fases:

```text
prepare
  ↓
persiste pending
  ↓
apply
  ↓
valida Telegram
  ↓
promove custom_emojis.json
```

A persistência antes da mutação permite retomada segura em timeout/interrupção.

#### `/del`

O modo `delete-app` segue o caminho inverso:

```text
prepare
  ↓
persiste pending
  ↓
remove no Telegram
  ↓
confirma ausência
  ↓
remove entrada ativa do JSON
```

O histórico de ícones não é apagado.

#### Migração em lote

`bulk-telegram` constrói um estado de retomada e um `custom_emojis.next.json`. A versão ativa só é promovida após validações de integridade.

## `azure-translator-test.yml`

Teste manual da lógica de idioma.

Serve para validar:

- detector local;
- decisão de quando traduzir;
- chamada ao Azure Translator;
- resposta traduzida.

Não faz parte do ciclo principal de monitoramento.

## `azure-translator-metrics-test.yml`

Valida:

- login Azure via OIDC;
- acesso ao recurso configurado;
- consulta da métrica de caracteres traduzidos;
- logout.

É útil para separar erro de credencial/permissão de erro no monitor.

## `test-r2-telegram-bot.yml`

Faz checkout do Core privado e executa o teste de Telegram.

Apesar do nome histórico, sua função prática é verificar o bot/canal usados pelo ecossistema do monitor, não validar o bucket R2 em profundidade.

## Repositórios dependentes

### `Lucas25Pedrosa/appstore-telegram-core`

Contém:

- `monitor.py`;
- scripts do Emoji Manager;
- `versions.json`;
- logs;
- relatórios;
- estado de consumo do Azure.

### `Lucas25Pedrosa/ipa-shared-config-core`

Contém:

- `apps.json`;
- custom emojis;
- hashes visuais;
- backup de ícones;
- configuração de emojis;
- estado transacional compartilhado.

## Cloudflare Worker

O bot/Worker de comandos como `/add`, `/del`, `/uso`, `/status`, `/ativar` etc. é implantado separadamente na Cloudflare e **não existe como arquivo versionado neste repositório na estrutura atual**.

Consequência importante: alterações em contratos entre Worker e GitHub Actions precisam ser coordenadas. Um workflow pode estar correto e ainda assim quebrar o comando se o Worker publicado estiver usando inputs antigos.

## Secrets e permissões

Categorias usadas pelos workflows:

- acesso ao `appstore-telegram-core`;
- acesso de escrita ao `ipa-shared-config-core`;
- Telegram;
- Azure Translator;
- Azure OIDC/subscription.

Valores reais nunca devem ser versionados.

## Regras de manutenção

1. Manter a separação **repositório público de workflow / Core privado**.
2. Não duplicar lógica de negócio grande no YAML quando ela pertence ao Core.
3. Preservar as etapas de `prepare` e `apply` das mutações de emoji.
4. Nunca promover `custom_emojis.next.json` sem validar estado completo.
5. Alterações em `apps.json` devem continuar sincronizando `versions.json`.
6. Mudanças no Worker precisam ser compatíveis com os inputs de `workflow_dispatch`.
7. Não transformar testes Azure/Telegram em dependência obrigatória da execução de produção.

## Relação com o armazenamento da IPA Library

O App Store Monitor também conversa com a estrutura de armazenamento por meio do Worker e de comandos administrativos. Hoje algumas rotas ainda usam conceitos/nomenclaturas de R2. Quando o backend de IPA mudar, os workflows deste repositório só devem ser modificados onde houver contrato real; a lógica do storage pertence principalmente a `ipa-r2-automation`/`ipa-r2-core` e ao Worker.
