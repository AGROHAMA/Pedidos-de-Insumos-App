# Pedidos de Insumos Agrohama: publicação e fluxo do Power Automate

O app é um PWA igual ao checklist de máquinas: HTML no celular de cada gerente, com fila offline. Ele manda um JSON para um fluxo do Power Automate, e o fluxo posta no Teams e cria o card no Planner.

## 1. Publicar o app (GitHub Pages)

1. Crie um repositório em AGROHAMA, por exemplo `pedidos-insumos-app` (público, como o do checklist).
2. Envie para a raiz: `index.html`, `manifest.json`, `service-worker.js` e a pasta `icons/`.
3. Em Settings > Pages, escolha a branch `main`, pasta `/ (root)`.
4. O endereço fica `https://agrohama.github.io/pedidos-insumos-app/`.
5. Cada gerente abre o endereço no celular e usa "Adicionar à tela inicial". Depois, abre em Ajustes e cola o endereço do fluxo (passo 2).
6. Ao publicar uma versão nova, troque `CACHE_VERSION` em `service-worker.js` (v1 para v2).

## 2. Fluxo do Power Automate (recebe os pedidos e as NFs)

Crie um fluxo "Quando uma solicitação HTTP é recebida" (método POST) e use o JSON abaixo como exemplo de esquema. Copie o URL gerado para a tela Ajustes do app.

Corpo enviado pelo app:

```json
{
  "evento": "pedido_novo | recebimento | teste",
  "texto": "texto pronto para o Teams (várias linhas)",
  "registro": { }
}
```

`registro` em `pedido_novo`: `codigo, gerente, produtor, fazenda, fornecedor, itens[{nome,qtd,un,preco}], total, vencimento, prazo, uso, obs, criadoEm, foto` (foto em base64, ou null).

`registro` em `recebimento`:
- `pedido`: `{codigo, fornecedor, produtor, fazenda, itens}`
- `recebimento`: `{tipo (unica | mae | filha), nf, maeNf, fornecedor, itens[quantidade por item, na ordem do pedido], por, em, foto}`, com `foto` em base64
- `saldoMae`: lista `{item, saldo}` (nula quando a mãe não foi registrada neste aparelho)

### Passos do fluxo

1. **Condição por `evento`:**
   - `pedido_novo`: posta no canal "pedidos de insumos" (ação "Postar mensagem em um chat ou canal") com o `texto`. Cria o card no Planner no bucket "Pedidos", com o título `codigo - produto` e a descrição igual ao `texto`.
   - `recebimento`: acha o card pelo `codigo` (Planner, "Listar tarefas", filtrando o título que contém o código). Responde na mesma postagem do Teams, ou comenta/atualiza o card, com o `texto`. Anexe a foto da NF: converta o base64 com `dataUriToBinary`, salve no SharePoint e coloque o link no card.
   - `teste`: só posta o `texto` num canal de testes.
2. **Aviso de lançamento:** crie um segundo fluxo com o gatilho do Planner "Quando uma tarefa é concluída". Ele posta uma mensagem no canal com a pessoa do escritório e a pessoa visitante marcadas. Assim, quem lança na cidade só marca o card como concluído, sem abrir app nenhum.

Observações:
- O app só considera enviado quando o fluxo responde com sucesso (200 ou 202). Em caso de erro, mantém o item na fila e tenta de novo.
- O app usa o mesmo tipo de chamada do checklist de máquinas (POST com `Content-Type: application/json`). Se o checklist funciona nos celulares, este também funciona.

## 3. Limites desta versão

- Cada celular guarda só os pedidos que ele mesmo criou ou recebeu. Quem recebe na fazenda usa "Outro pedido (digitar o código)", com o código que aparece na postagem do Teams.
- O saldo da NF mãe é calculado por item e só no aparelho em que a mãe e as filhas foram registradas. Use o mesmo celular para receber todas as NFs de um pedido. O texto enviado ao Teams já leva o saldo.
- Para ver os pedidos de todos no celular, o próximo passo é um backend com banco de dados, como o do `server.js` do checklist.
