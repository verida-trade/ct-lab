# 01 — CT Lab Desktop

O **CT Lab Desktop** é o aplicativo principal do ecossistema CT Lab. Construído
sobre Electron, ele roda nativamente em macOS, Linux e Windows e oferece uma
interface completa para conversar com a IA, visualizar gráficos e gerenciar
projetos de análise quantitativa.

---

## Requisitos do Sistema

| Sistema Operacional | Versão Mínima | Observações |
|---------------------|---------------|-------------|
| macOS | 11 — Big Sur | Intel e Apple Silicon |
| Ubuntu | 20.04 (Focal Fossa) | Também compatível com distribuições derivadas |
| Windows | 10 | Windows 10 build 19041+ |

**Recursos recomendados:**

- 4 GB de RAM (mínimo 2 GB)
- 500 MB de espaço em disco
- Conexão de internet ativa para consulta de dados de mercado

---

## Download

Acesse o site oficial para baixar o instalador:

> **[verida.trade/download](https://verida.trade)**

Selecione a versão correspondente ao seu sistema:

| Plataforma | Arquivo |
|------------|---------|
| macOS | `CT-Lab-x.x.x.dmg` (macOS universal) |
| Linux | `CT-Lab-x.x.x.AppImage` |
| Windows | `CT-Lab-x.x.x.Setup.exe` |

> ⚠️ **Sempre baixe do site oficial**. O CT Lab não distribui o aplicativo
> por canais de terceiros.

---

## Instalação

### macOS

1. Abra o arquivo `.dmg` baixado.
2. Arraste o ícone **CT Lab** para a pasta **Applications**.
3. Abra o Launchpad e clique em **CT Lab**.

> Na primeira execução, o macOS pode exibir um alerta de segurança. Clique em
> **Preferências do Sistema → Segurança e Privacidade → Abrir Mesmo Assim**
> para autorizar.

### Ubuntu / Linux

```bash
# Dê permissão de execução ao AppImage
chmod +x CT-Lab-x.x.x.AppImage

# Execute
./CT-Lab-x.x.x.AppImage
```

> Se o Ubuntu bloquear o AppImage por motivos de segurança, instale o FUSE:
> `sudo apt install libfuse2`

### Windows

1. Dê duplo clique no arquivo `CT-Lab-x.x.x.Setup.exe`.
2. O instalador solicitará permissão — clique em **Sim**.
3. Siga o assistente: localize o destino da instalação e clique em **Instalar**.
4. Ao término, clique em **Concluir** — o CT Lab abrirá automaticamente.

> Se o Windows Defender bloquear a instalação, clique em **Mais informações →
> Executar mesmo assim**. O instalador é assinado e seguro.

---

## Primeira Inicialização

Quando você abrir o CT Lab pela primeira vez, verá a seguinte tela:

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           CT LAB — Bem-vindo!                                │
│                                                                               │
│   Antes de começar, precisamos configurar algumas coisas:                    │
│                                                                               │
│   1. [●]  Configurar provedor de IA                   [Configurar →]         │
│   2. [○]  Conectar o servidor MCP ao CT Lab          [Conectar →]            │
│   3. [○]  Criar seu primeiro projeto                  [Começar →]            │
│                                                                               │
│   ───────────────────────────────────────────────────────────────────────    │
│   Pular assistente · Abrir Configurações                                      │
└───────────────────────────────────────────────────────────────────────────────┘
```

O assistente de boas-vindas guia você por três etapas:

1. **Configurar provedor de IA** — escolhe OpenAI, Anthropic, Google ou Ollama (ver documento 02).
2. **Conectar MCP** — vincula o servidor ao aplicativo (ver documento 03).
3. **Criar seu primeiro projeto** — roda uma análise prática (ver documento 04).

> 💡 Você pode pular o assistente e configurar tudo manualmente em **Configurações**.

---

## Painel de Configurações

Acesse em **CT Lab → Settings** (ou `Cmd/Ctrl + ,`):

| Seção | Descrição |
|-------|-----------|
| **General** | Idioma, tema (claro/escuro), diretório de dados |
| **AI Provider** | Escolher e configurar provedor (OpenAI, Anthropic, Google, Ollama) |
| **Extensions** | Adicionar e gerenciar servidores MCP |
| **Data Sources** | Conectar corretoras e provedores de dados (Binance, Yahoo) |
| **License** | Verificar status da licença, comprar plano Premium |
| **Advanced** | Logs, cache de séries, diagnóstico |

---

## Diretório de Dados

O CT Lab armazena configurações e cache de séries em um diretório local:

| Sistema | Caminho |
|---------|---------|
| macOS | `~/Library/Application Support/ct-lab` |
| Linux | `~/.config/ct-lab` |
| Windows | `%APPDATA%\ct-lab` |

> ⚠️ Não edite manualmente os arquivos neste diretório. Use a interface do
> CT Lab para todas as configurações.

---

## Licença

Na primeira inicialização, o CT Lab opera em modo **Gratuito**:

- ✅ 1 série temporal em cache
- ✅ 36 indicadores técnicos públicos
- ✅ Backtesting

Para desbloquear recursos premium (100 séries, indicadores CT, ML,
microestrutura), consulte a seção **License** nas configurações — veja
[verida.trade](https://verida.trade) para mais detalhes.

---

## Verificação

Após a instalação, verifique se o aplicativo está funcionando:

1. Abra o CT Lab.
2. Você deve ver o painel principal com o chat desativado e o assistente de boas-vindas.
3. Vá em **Settings → General** e confirme que o número da versão aparece corretamente.

> Se nada aparece ou há erros de inicialização, consulte os logs em
> **Settings → Advanced → Open Logs**.

---

## Próximos Passos

- ➡️ **[02 — Provedor de IA](./02-provider-ia)** — Configure a IA
- ⬅️ **[Voltar ao Índice](./README)**

---

_Last updated: 2026-08-11_
