# Wascer OpenAI Ads Pixel: guia de configuração

Guia em português para quem vai instalar a tag do zero, do jeito que o usuário
vê na tela. O `README.md` ao lado é a versão em inglês para a Community Gallery.

## Como as duas tags se encaixam

A Wascer tem duas tags de OpenAI Ads, e elas fazem coisas diferentes:

- **Wascer OpenAI Ads Pixel** roda no **container web**, dentro do navegador do
  visitante. É esta aqui.
- **Wascer OpenAI Ads Conversions** roda no **container server-side** e fala
  direto com a API da OpenAI. É a outra, documentada em
  `openai-ads-capi-wascer/CONFIGURACAO.md`.

Você pode usar só o pixel. O ideal é usar os dois, porque o servidor pega o que o
navegador perde: bloqueador de anúncio, aba fechada antes da confirmação, compra
que só existe no seu backend. As duas pontas mandando o mesmo `event_id` contam
**uma** conversão, não duas.

Comece pelo pixel. Ele é o mais rápido de colocar de pé e serve de referência
para o servidor depois.

---

## Antes de começar

Você precisa de três coisas:

1. Uma conta de anunciante no **OpenAI Ads** com acesso à aba de conversões.
2. Um **container web** do Google Tag Manager já instalado no site.
3. O arquivo `template.tpl` desta pasta.

Se o site usa GA4 com ecommerce, você já tem 90% do trabalho pronto, porque a tag
lê o mesmo dataLayer.

---

## Passo 1: pegar o Pixel ID

Entre no **OpenAI Ads Manager** e abra a aba **Conversions**. É lá que você cria
o Pixel ID. A documentação da OpenAI diz exatamente isso: "You can provision a
Pixel ID and Conversions API key from the conversions tab in Ads Manager."

Guarde o valor. O **mesmo** Pixel ID serve para esta tag e para a tag de
servidor. A API key que aparece na mesma tela é só do servidor, e o pixel de
navegador não usa chave nenhuma.

Se a aba não aparece na sua conta, é questão de acesso, não de configuração. O
caminho é falar com o contato da OpenAI Ads ou usar a rota de API partner.

---

## Passo 2: importar o template no GTM

No **container web**:

1. Menu lateral, **Modelos**.
2. Bloco **Modelos de tag**, botão **Novo**.
3. No editor que abrir, clique no menu **⋮** no canto superior direito e escolha
   **Importar**.
4. Selecione o arquivo `template.tpl`.
5. **Salvar**.

Pronto, a tag passa a aparecer na lista de tipos de tag do container.

> A importação é por container. Se você tem vários sites, repita em cada um, ou
> espere a publicação na Community Gallery para instalar pela galeria.

---

## Passo 3: rodar os testes do template

Ainda no editor do template, abra a aba **Testes** e clique em **Executar todos**.
São nove testes que já vêm escritos dentro do arquivo, e eles cobrem hash de
email, consentimento, conversão de moeda e reuso da fila. Leva segundos e pega
qualquer problema de importação antes de você mexer em tag.

Todos passando, pode fechar o editor.

---

## Passo 4: a primeira tag, o page view

1. Menu lateral, **Tags**, botão **Novo**.
2. Clique em **Configuração da tag** e escolha **Wascer OpenAI Ads Pixel**, na
   seção **Personalizado** do final da lista.
3. Preencha:

| Campo | Valor |
|---|---|
| OpenAI Pixel ID | o ID do passo 1 |
| Event name | **Standard event** |
| Event | `page_viewed` |
| Consent mode | deixe como está por enquanto |

4. Em **Acionamento**, escolha **All Pages**.
5. Dê um nome à tag, por exemplo `OpenAI Ads Page viewed`, e salve.

Só isso já mede tráfego e alimenta o cookie de atribuição. Os eventos de
conversão vêm no passo seguinte.

---

## Passo 5: os eventos de conversão

Cada conversão é **uma tag**, com o mesmo Pixel ID e um acionador próprio. O
mapeamento natural a partir do ecommerce do GA4:

| Evento no seu dataLayer | Event a escolher na tag |
|---|---|
| `view_item` | `contents_viewed` |
| `add_to_cart` | `items_added` |
| `begin_checkout` | `checkout_started` |
| `purchase` | `order_created` |
| `sign_up` | `registration_completed` |
| `generate_lead` | `lead_created` |
| agendamento | `appointment_scheduled` |
| início de assinatura | `subscription_created` |
| início de teste grátis | `trial_started` |

Para o acionador, use **Evento personalizado** com o nome do evento do dataLayer.
Exemplo para a compra: acionador do tipo Evento personalizado, nome do evento
`purchase`.

Deixe o checkbox **Read amount, currency and items from the data layer** ligado,
que é o padrão. A tag pega sozinha o `value`, o `currency` e os `items` do
dataLayer, e converte o valor para a unidade que a OpenAI espera. Você não
precisa criar variável nenhuma para isso.

### O formato que a tag espera

É o ecommerce padrão do GA4. Um `purchase` completo:

```js
dataLayer.push({ ecommerce: null });
dataLayer.push({
  event: 'purchase',
  ecommerce: {
    transaction_id: 'T-12345',
    value: 129.90,
    currency: 'BRL',
    items: [
      { item_id: 'SKU-1', item_name: 'Plano Pro', price: 129.90, quantity: 1 }
    ]
  }
});
```

A tag lê `items` na raiz e, se não achar, dentro de `ecommerce`. Os dois formatos
funcionam.

Se o dataLayer não trouxer `value`, ela soma os itens. E ela nunca manda valor
sem moeda, porque a OpenAI recusa.

---

## Passo 6: Event ID, para não contar conversão dobrada

Este passo só importa se você **também** vai usar a tag de servidor. Se for usar
só o pixel, pule.

A OpenAI junta o evento do navegador com o do servidor quando os dois chegam com
o mesmo identificador. Aqui o campo se chama **Event ID**, e no servidor se chama
`id`.

Para a compra, o mais simples é usar o próprio id do pedido:

1. **Variáveis**, **Nova**, tipo **Variável da camada de dados**.
2. Nome da variável na camada de dados: `ecommerce.transaction_id`.
3. Salve como `DLV transaction_id`.
4. Na tag, campo **Event ID**, escolha essa variável.

Para eventos que não têm id natural, crie uma variável que gere um valor por
evento e mande esse mesmo valor ao servidor.

---

## Passo 7: dados do usuário (advanced matching)

Isso melhora bastante o casamento das conversões, e é opcional.

Ligue o checkbox **Send hashed user data from the data layer**. A tag passa a ler
o objeto `user_data` do dataLayer:

```js
dataLayer.push({
  event: 'purchase',
  user_data: {
    email: 'cliente@exemplo.com',
    address: {
      city: 'São Paulo',
      postal_code: '01310-100',
      country: 'BR'
    }
  },
  ecommerce: { /* ... */ }
});
```

**O email vai hasheado.** A tag aplica SHA-256 antes de enviar, e nada em texto
puro sai do navegador. Se você já manda o hash pronto do seu backend, ela
reconhece e não hasheia de novo.

Se preferir preencher na mão, use a tabela do grupo **User data**. O que você
digita lá vence o dataLayer. Os nomes aceitos são `email_sha256`,
`external_id_sha256`, `country`, `city` e `zip_code`.

---

## Passo 8: consentimento

O campo **Consent mode** tem duas posições:

- **Wait for ad_storage consent**, que é o padrão. A tag segura o evento e o SDK
  fica fora da página enquanto o visitante não aceitar. Assim que ele aceita, o
  evento sai sozinho, sem precisar de novo pageview.
- **Always send**, que dispara na hora.

Se o site tem banner de cookies integrado ao Consent Mode do Google, deixe no
padrão. Se o consentimento é tratado de outro jeito na sua operação, aí sim
avalie o **Always send**.

O checkbox **Opt this user out of personalization** existe para o caso de o
visitante pedir para não ser usado em personalização. Ele manda `opt_out: true` e
o evento continua contando como conversão.

---

## Passo 9: testar antes de publicar

Clique em **Visualizar**, abra o site, e confira três coisas.

**No painel do Tag Assistant:** a tag aparece como *Fired*. Se ficar em *Still
running*, é o consentimento que ainda não veio, e isso é o comportamento certo.

**Na aba Network do navegador**, filtre por `openai`. Tem que aparecer o
carregamento do SDK:

```
https://bzrcdn.openai.com/sdk/oaiq.min.js
```

e, logo depois, a chamada de evento que o SDK faz sozinho.

**No Console**, digite:

```js
window.oaiq.q
```

Essa é a fila que a tag monta, e ela mostra exatamente o que foi enviado, antes
de o SDK tocar em qualquer coisa. Você deve ver algo como
`['init', {...}]` e `['measure', 'order_created', {...}, {...}]`.

Quer conferir o hash do email? Compare com:

```sh
printf '%s' 'cliente@exemplo.com' | sha256sum
```

Tem que bater caractere por caractere.

O checkbox **debug**, no grupo Advanced, liga o log do SDK no console. Use na
montagem e desligue depois.

---

## Passo 10: publicar

**Enviar**, nome da versão, **Publicar**. Alguns minutos depois as conversões
começam a aparecer no Ads Manager.

Uma observação honesta: o pixel de navegador não tem modo de validação. Tudo que
ele disparar entra na conta de verdade. Se quiser validar o formato sem sujar
nada, monte primeiro a tag de servidor com o **Validate without saving** ligado.

---

## Os campos, um a um

### Configuration

| Campo | O que faz |
|---|---|
| OpenAI Pixel ID | Identifica o seu pixel. Sai da aba de conversões do Ads Manager. |
| Event name | Alterna entre evento padrão e evento seu. |
| Event | O evento padrão. São dez opções, e cada uma já leva o tipo de dado certo. |
| Custom event name | O nome do seu evento, quando você escolhe custom. Letras, números, underscore e hífen, até 64 caracteres. |
| Event ID | A chave que amarra este disparo ao do servidor. |

### Data layer

| Campo | O que faz |
|---|---|
| Read amount, currency and items from the data layer | Lê `value`, `currency` e `items`, com fallback dentro de `ecommerce`. Ligado por padrão. |
| Send hashed user data from the data layer | Lê `user_data` e transforma nos campos que a OpenAI aceita. Desligado por padrão. |

### User data

Tabela para preencher identificador na mão. O que você digita aqui vence o
dataLayer. Aceita `email_sha256`, `external_id_sha256`, `country`, `city` e
`zip_code`.

O tratamento de cada um:

| Campo | Normalização | Hash |
|---|---|---|
| `email_sha256` | trim e minúscula | sim |
| `external_id_sha256` | trim | sim |
| `country` | trim e maiúscula | não |
| `city` | trim, minúscula, corte em 128 caracteres | não |
| `zip_code` | trim, corte em 32 caracteres | não |

### Consent

| Campo | O que faz |
|---|---|
| Consent mode | Esperar o `ad_storage` ou mandar sempre. |
| Opt this user out of personalization | Envia `opt_out: true` nas opções do evento. |

### Advanced

**Log SDK debug output to the browser console** liga o modo debug do SDK e
imprime o que a tag montou. Só para a fase de montagem.

---

## Eventos e o tipo de dado

| Evento | Tipo | Pode levar |
|---|---|---|
| `page_viewed` | `contents` | `amount`, `currency`, `contents` |
| `contents_viewed` | `contents` | `amount`, `currency`, `contents` |
| `items_added` | `contents` | `amount`, `currency`, `contents` |
| `checkout_started` | `contents` | `amount`, `currency`, `contents` |
| `order_created` | `contents` | `amount`, `currency`, `contents` |
| `lead_created` | `customer_action` | `amount`, `currency` |
| `registration_completed` | `customer_action` | `amount`, `currency` |
| `appointment_scheduled` | `customer_action` | `amount`, `currency` |
| `subscription_created` | `plan_enrollment` | `plan_id`, `amount`, `currency`, `contents` |
| `trial_started` | `plan_enrollment` | `plan_id`, `amount`, `currency`, `contents` |
| evento custom | `custom` | `plan_id`, `amount`, `currency`, `contents` |

`app_installed` e `app_opened` não estão na lista de propósito. O pixel de
navegador não aceita esses dois, e eles só saem pela tag de servidor.

---

## Conversão de valor

A OpenAI recebe valor em unidade menor. R$ 25,99 vira `2599`. O GA4 manda em
unidade regular, então a tag multiplica, respeitando a moeda:

- **100** para a maioria, incluindo BRL, USD e EUR
- **1** para moedas sem centavos: BIF, CLP, DJF, GNF, IDR, ISK, JPY, KMF, KRW,
  MGA, PYG, RWF, UGX, VND, VUV, XAF, XOF e XPF
- **1000** para moedas de três casas: BHD, IQD, JOD, KWD, LYD, OMR e TND

Ou seja, `value: 2599` em JPY continua `2599`.

---

## Content Security Policy

Se o site roda CSP, libere:

```
script-src  https://bzrcdn.openai.com
connect-src https://bzr.openai.com https://bzrcdn.openai.com
img-src     https://bzr.openai.com
```

Sem isso, o SDK não carrega e a tag falha sem motivo aparente.

---

## Quando não funciona

**A tag fica em *Still running*.** É o consentimento. Com o modo `Wait for
ad_storage consent`, a tag registra um ouvinte e só dispara quando o visitante
aceita. Aceite no banner e veja se ela conclui.

**O SDK não aparece no Network.** Três suspeitos, nesta ordem: consentimento
negado, CSP bloqueando `bzrcdn.openai.com`, ou bloqueador de anúncio na extensão
do navegador. Teste numa janela anônima sem extensões.

**`oaiq.q` está vazio ou não existe.** A tag não chegou a rodar. Confira o
acionador no Tag Assistant.

**O valor chegou errado por 100.** Alguém já está mandando em centavos no
dataLayer. Nesse caso divida por 100 na variável, porque a tag assume unidade
regular, que é o contrato do GA4.

**Aparecem duas conversões.** O `event_id` do navegador não está batendo com o
`id` do servidor. Confira se a variável resolve para o mesmo valor nas duas
pontas.

**O `user` sai vazio.** O checkbox de advanced matching está desligado, ou o
`user_data` não está no dataLayer no momento em que a tag dispara. Lembre que o
push precisa vir **antes** do evento que aciona a tag.

---

## Referências

- Pixel de navegador: https://developers.openai.com/ads/measurement-pixel
- Eventos suportados: https://developers.openai.com/ads/supported-events
- Conversions API: https://developers.openai.com/ads/conversions-api
