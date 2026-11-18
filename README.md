# cartaregistada.pt MCP server

Send paper letters through CTT, Portugal's national postal service (physical mail, not email), from Claude, ChatGPT or any MCP client: registered mail with proof of receipt, registered, priority (azul) or regular mail. It serves any letter: complaints to insurers, telecoms and companies, rent updates, lease terminations and non-renewal notices, contract cancellations, condominium notices, letters to the tax office and social security, landlord and tenant notices, formal demands.

cartaregistada.pt is a Portuguese service that prints a letter, hands it to CTT (the
Portuguese postal operator) as registered mail with or without acknowledgement of receipt
(or as priority or standard mail), and tracks it until delivery. This repository documents
the remote MCP (Model Context Protocol) server that lets an AI assistant do that on behalf
of a person who has an account.

**No code here: the server runs at cartaregistada.pt.** This repository holds the
documentation, the `server.json` published to the official MCP registry, and a few example
messages.

- Server URL: `https://conta.cartaregistada.pt/mcp`
- Transport: Streamable HTTP (JSON-RPC 2.0 over POST, JSON responses, stateless)
- Authentication: OAuth 2.1 with dynamic client registration and PKCE (S256)
- Discovery: `https://conta.cartaregistada.pt/.well-known/oauth-authorization-server`
  and `https://conta.cartaregistada.pt/.well-known/oauth-protected-resource`
- For people: https://www.cartaregistada.pt/assistentes (English: https://www.cartaregistada.pt/en/ai-assistants)
- For agents: https://www.cartaregistada.pt/llms.txt

## Who it is for

Anyone in Portugal who needs to send a letter with proof of posting or delivery (landlords,
tenants, companies, condominium administrators, individuals) and would rather ask their
assistant than go to the post office. Destination addresses must be in Portugal.

## Connecting

The server speaks the standard MCP Streamable HTTP transport, so any client that supports
remote MCP servers with OAuth can use it. On first use the client opens the cartaregistada.pt
consent page: the person signs in to their free account (a code sent by email) and
authorises the application.

### Claude (claude.ai and desktop)

Settings > Connectors > Add custom connector, with the URL
`https://conta.cartaregistada.pt/mcp`. Claude handles the OAuth flow in the browser.

### Claude Code

```sh
claude mcp add --transport http cartaregistada https://conta.cartaregistada.pt/mcp
```

Then `/mcp` inside Claude Code to authenticate.

### ChatGPT

In ChatGPT, Settings > Connectors (or Apps) > Create, with the server URL
`https://conta.cartaregistada.pt/mcp` and OAuth as the authentication method. Availability
of custom connectors depends on your ChatGPT plan.

### Perplexity

Perplexity's connectors accept remote MCP servers in the settings of the desktop app; add
`https://conta.cartaregistada.pt/mcp` and complete the sign-in when prompted.

### Cursor, VS Code and other `.mcp.json` clients

```json
{
  "mcpServers": {
    "cartaregistada": {
      "type": "http",
      "url": "https://conta.cartaregistada.pt/mcp"
    }
  }
}
```

The same file is in this repository as [`.mcp.json`](.mcp.json). VS Code uses the
`servers` key instead of `mcpServers` in its `mcp.json`; the entry itself is the same.

## Tools

Nine tools. Only `enviar_carta` and `enviar_lote` create anything; none deletes.

| Tool | What it does |
|---|---|
| `orcamento` | Quote for one letter from this account (VAT and postage included) and the day it leaves for CTT. |
| `localidade_do_codigo_postal` | Town and municipality of a Portuguese postal code, to fill in the recipient. |
| `remetentes` | The senders saved in the account (who signs the envelope), with the default one. |
| `enviar_carta` | Creates one letter (plain text, which the service paginates into a PDF, or a PDF/image file) to a Portuguese address. Returns the Stripe payment link. |
| `estado_da_carta` | Where one letter is: unpaid (with the link), being prepared, at CTT, delivered or returned, with the CTT tracking number and dates. Never the text or the file. |
| `cartas` | The account's letters, most recent first, by tab, search and pages of 50. |
| `enviar_lote` | Creates a batch: the same letter to up to 200 Portuguese addresses, listed or taken from an account contact group, with a single payment link. |
| `estado_do_lote` | Where one batch is, with the aggregated tracking of its letters and each letter's status. Never the file. |
| `lotes` | The account's batches, most recent first, with status and counts. |

Each tool has a Portuguese description with a short English sentence, an `inputSchema`, and
the MCP annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`).
See [`examples/`](examples/) for an `initialize`, a `tools/list` and a sample `orcamento`
response.

### How a letter is sent

1. The assistant calls `orcamento` and shows the price to the person.
2. With the person's consent, it calls `enviar_carta` with `confirmo_preco: true`
   (the tool refuses otherwise).
3. The response carries a Stripe payment link. The person opens it and pays.
4. After payment the letter is printed and handed to CTT within the stated deadline;
   delivery is by CTT, usually 1 to 7 days, not guaranteed.
5. `estado_da_carta` follows it until delivery or return.

The same letter to several recipients (a building, a contact group) is one batch
(`enviar_lote`) with a single payment.

## What the assistant never does

- **Pay.** Payment is always a Stripe link the person opens; the assistant never handles
  card details. (An optional charge on a saved card exists behind a separate OAuth scope
  that is granted only to specific applications and is off by default.)
- **Send outside Portugal.** Destination addresses must be Portuguese, with a `0000-000`
  postal code.
- **Give legal advice.** The content of the letter is the person's.

## Authentication

- OAuth 2.1 authorization server at `https://conta.cartaregistada.pt`, discovered through
  RFC 8414 (`/.well-known/oauth-authorization-server`) and RFC 9728
  (`/.well-known/oauth-protected-resource`).
- Dynamic client registration (RFC 7591) at the `registration_endpoint`; redirect URIs
  must be `https://` (no local hosts in production).
- Authorization code flow with PKCE, S256 only. Refresh tokens rotate on every use.
- Scopes: `cartas:enviar` and `cartas:ler` (the defaults). A third scope,
  `pagamentos:cobrar`, is only granted to allow-listed applications.
- The person signs in to their free account at `conta.cartaregistada.pt` with a code sent by
  email (or a password, if set) and authorises the application on the consent page.
- Access can be revoked at any time in the account, under Conta > Seguranca.

## Pricing

Prices (VAT and CTT postage included) are listed at
https://www.cartaregistada.pt/assistentes. The `orcamento` tool returns the exact price for
the account before anything is created.

## Links

- Website: https://www.cartaregistada.pt
- For AI assistants (PT): https://www.cartaregistada.pt/assistentes
- For AI assistants (EN): https://www.cartaregistada.pt/en/ai-assistants
- Instructions for agents: https://www.cartaregistada.pt/llms.txt
- Privacy: https://www.cartaregistada.pt/privacidade
- Terms: https://www.cartaregistada.pt/termos
- Contact: ola@cartaregistada.pt

## License

The documentation in this repository is licensed under CC BY 4.0 (see [LICENSE](LICENSE)).
The service itself and its code are not covered by this license.

---

## Em portugues

O cartaregistada.pt envia cartas registadas (com ou sem aviso de recepcao), e tambem por
correio azul ou normal, pelos CTT, para moradas em Portugal. Este repositorio documenta o
servidor MCP remoto que deixa um assistente de IA (o Claude, o ChatGPT, o Claude Code, o
Cursor, qualquer cliente MCP) enviar e seguir cartas em nome de uma pessoa com conta.

**Aqui nao ha codigo: o servidor corre no cartaregistada.pt.** O que esta neste
repositorio e a documentacao, o `server.json` publicado no registo oficial do MCP e alguns
exemplos de mensagens.

- Endereco do servidor: `https://conta.cartaregistada.pt/mcp`
- Autenticacao: OAuth 2.1 com registo dinamico de clientes e PKCE (S256). A pessoa entra
  na conta gratis em conta.cartaregistada.pt com um codigo enviado por email e autoriza a
  aplicacao; revoga o acesso quando quiser em Conta > Seguranca.
- Nove ferramentas: `orcamento`, `localidade_do_codigo_postal`, `remetentes`,
  `enviar_carta`, `estado_da_carta`, `cartas`, `enviar_lote`, `estado_do_lote` e `lotes`.
  So as duas `enviar_*` criam alguma coisa; nenhuma apaga.
- O assistente nunca paga: o pagamento e sempre um link do Stripe que a pessoa abre. Nunca
  envia para fora de Portugal. Nunca da aconselhamento juridico.
- Os precos estao em https://www.cartaregistada.pt/assistentes.
- Para ligar no Claude Code: `claude mcp add --transport http cartaregistada https://conta.cartaregistada.pt/mcp`.
  Para o Cursor, o VS Code e outros, o [`.mcp.json`](.mcp.json) deste repositorio.
- Contacto: ola@cartaregistada.pt. Privacidade: https://www.cartaregistada.pt/privacidade.
  Termos: https://www.cartaregistada.pt/termos.
