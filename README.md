# Protocolo de Espelhamento de Posições

## 1. Visão Geral
Este documento define um protocolo pelo qual um sistema (*origem*) compartilha dados de posição GPS de veículos com outro sistema (*destino*).

A integração é feita via API REST sobre HTTP. O sistema de destino implementa a API e responde chamadas a ela; o sistema de origem chama a API sempre que possuir novos dados para enviar.

Tipicamente, a posição GPS é obtida através de um hardware instalado nos veículos, que periodicamente transmite dados para um servidor central (o servidor de *origem* neste documento). É comum que a transmissão do veículo inclua outros dados, e inclua dados de períodos inteiros (ex: "todas as posições GPS da última hora").

O servidor de origem recebe os dados de vários veículos e eventualmente decide compartilhá-los com outro servidor (o servidor *destino* neste documento). Observe que os dados que o servidor de origem compartilha podem ser filtrados (ex: apenas alguns veículos estarão configurados para espelhar) ou modificados (ex: coordenadas redundantes podem ser removidas).

## 2. Códigos de Erro e Content-Type
O protocolo usa os códigos padrão HTTP:

    HTTP 200: para indicar chamadas realizadas com sucesso
    HTTP 400: para indicar inconsistências nos dados enviados
    HTTP 401: para indicar que o access_token informado não é válido
    HTTP 403: para indicar que o token é válido mas não pode ser usado para fazer uma certa chamada.
    HTTP 429: para indicar que o sistema origem está fazendo requests demais em um curto período de tempo.

Salvo especificação em contrário, o `Content-Type` de todo request e response deve ser `application/json`.

Salvo especificação em contrário, respostas de erro devem incluir pelo menos uma propriedade `error` com um código explicativo para o erro. Por exemplo:

    HTTP 401
    {"error": "BAD_ACCESS_TOKEN"}

Para facilitar testes, assume-se que os ambiente de produção (PRD) e homologação/QA (HLG) terão URLs distintas. Por exemplo:

    PRD: https://destino.exemplo.com/api/espelhamento
    HLG: https://destinohlg.exemplo.com/api/espelhamento

As chamadas descritas no restante do documento devem combinar o path com o valor acima. Por exemplo:

    POST http://destino/exemplo.com/api/espelhamento/positions

## 3. Autenticação
Ao invés de username/senha, todo request à API deve conter um header HTTP chamado `access_token`. O sistema destino deve usar este token para identificar o sistema origem e autorizar as chamadas; também pode cruzar esta informação para saber se um certo sistema origem é de fato "dono" de um certo veículo.

Como é o sistema de destino que valida o token, é ele também que deve fornecê-lo. Para um primeiro momento, isto pode ser um cadastro manual, e o token pode ser enviado para os sistemas origem por um canal externo (telefone ou email). Futuramente, o fluxo OAuth completo poderia ser usado no lugar.

Caso o token não seja válido, a resposta será `HTTP 403` (Forbidden). Se estiver faltando, será `HTTP 401` (Unauthorized).

Os `access_token`s devem ser diferentes entre os ambientes de produção e homologação, para evitar que acessos de um tipo sejam por engano processados pelo outro.

## 4. Validação de Veículos
Quando um sistema origem envia dados, o sistema destino *PODE* validar se o veículo informado realmente pertence ao sistema origem. Se não pertencer, o resultado deve ser HTTP 400, com um corpo indicando este erro:

    HTTP 400
    {"error": "NO_SUCH_VEHICLE"}

Opcionalmente, o sistema destino pode aceitar um registro de posição de um veículo que ainda não existe, e criá-lo/associá-lo automaticamente. Se preferir, no entanto, o sistema destino pode exigir que o cadastro de veículos seja feito antes das transmissões poderem ser aceitas. Isto não é descrito neste documento.
  
## 5. APIs

### 5.1. API para envio de posições
```javascript
POST /positions

{
  "vehicle": {
    "code": "TST-1234"
  },
  "positions": [
    {
      "timestamp": "2017-02-01T12:00:00-0200",
      "lat": -23.004388,
      "lng": -47.116368
    },
    {
      "timestamp": "2017-02-01T12:00:01-0200",
      "lat": -23.004388, 
      "lng": -47.116368
    },
    ...
  ]
}
```

O request acima é um exemplo para compartilhar duas posições do veículo **TST-1234**.

A primeira parte (`vehicle: {}`) define o veículo:

Propriedade | Descrição
----------- | ---------
`code`      | **OBRIGATÓRIO**. Placa ou identificador único do veículo. Observe que cada chamada transmite os dados de apenas um veículo. Note que este código é *alfanumérico*. Em particular, *NÃO DEVE* ser um ID gerado por sequence em uma tabela no sistema destino; isto amarra e engessa os bancos de dados dos sistemas.

A segunda parte é um array de posições (`positions: []`), onde cada elemento possui:

Propriedade | Descrição
----------- | -------------
`timestamp` | **OBRIGATÓRIO**. A data/hora da posição. O formato é sempre ISO-8601: `YYYY-MM-DDTHH:mm:ss-0000` (onde "T" é uma literal e -0000 indica um offset UTC. Pode-se usar o offset "Z" para referenciar UTC).
 `lat, lng` | **OBRIGATÓRIOS**. A latitude/longitude sendo relatada.

**Resultado**

Espera-se que o sistema destino receba estas informações e siga pelo menos os seguintes passos:

* Valide o `access_token`, identificando o sistema origem
* Valide que os dados passados estão no formato correto e são de veículos permitidos
* Grave as posições informadas. Se alguma posição já existir (isto é, o sistema de origem está re-enviando dados), o sistema destino pode aceitar as novas posições (apagando/atualizando as antigas), ou desprezá-las, mas **NÃO DEVE** retornar um erro
* Grave a data do recebimento. Isto é, para um certo veículo, existe a lista de posições, e existe a data da última chamada que o sistema origem fez com dados deste veículo. 
* Retorne um código de sucesso

Quando o sistema origem recebe o código de sucesso, ele assume que todos os dados enviados foram recebidos e armazenados. Dessa forma, o próprio sistema origem pode controlar quais dados ele deve enviar na próxima chamada.

***

## Melhorias Futuras
O protocolo pode ser melhorado com algumas novas features:

* "Recibo" de entrega: um identificador retornado pelo sistema destino, para permitir conferir os recebimentos
* Consulta de posições: A API acima representa apenas a entrega dos dados. Espera-se que o sistema destino também possua APIs para consultar esses dados, mas por enquanto elas não são necessárias para este protocolo. No entanto, pode ser que uma versão futura defina essas APIs, para permitir, por exemplo, que os sistemas de origem descubram a partir de qual data eles *deveriam* enviar dados de um certo veículo.
* Flag de ignição: Além da latitude/longitude, cada registro de posição poderia incluir algumas flags, por exemplo, `engineOn` para dizer se o veículo está ligado ou não, `hasSpeed` para indicar se está em movimento ou não, etc. Observe, no entanto, que o objetivo **primário** deste protocolo é apenas espelhar as posições GPS; para outros propósitos, como por exemplo detectar violações de velocidade máxima, seria necessário outro protocolo com outros objetivos, restrições, e soluções.
