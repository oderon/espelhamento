# Protocolo de Espelhamento de Posições

## 1. Visão Geral
Este documento define um protocolo pelo qual um sistema (*origem*) compartilha dados de posição GPS de veículos com outro sistema (*destino*).

A integração é feita via API REST sobre HTTP. O sistema de destino implementa a API e responde chamadas a ela; o sistema de origem chama a API sempre que possuir novos dados para enviar.

![Diagrama 1](https://github.com/oderon/espelhamento/raw/master/img/protocolo-espelhamento.png "Diagrama 1")

Tipicamente, a posição GPS é obtida através de um hardware instalado nos veículos, que periodicamente transmitem dados para um servidor central (o sistema de *origem* neste documento). É comum que a transmissão do veículo inclua outros dados (por exemplo, sensores de velocidade e RPM), e inclua dados de períodos inteiros (por exemplo, todas as posições GPS dos últimos 15 minutos podem ser enviados para o servidor de uma vez).

O sistema origem recebe os dados de vários veículos e eventualmente decide compartilhá-los com outro sistema (o sistema *destino* neste documento). Observe que os dados que o servidor de origem compartilha podem ser filtrados (por exemplo, apenas alguns veículos estarão configurados para espelhar) ou modificados (ex: coordenadas redundantes podem ser removidas).

Além disso, a *frequência* com que os dados são transmitidos pode ser controlada. Por exemplo, um sistema origem pode agregar dados de dezenas de veículos -- que, agrupados, fazem dezenas de requisições por segundo -- mas envia para o sistema de destino apenas a cada 5 minutos com o aglomerado de dados até então.

## 2. Exemplo

Quando o sistema origem decide enviar os dados para um sistema destino, ele envia a seguinte requisição para um endpoint REST *do sistema destino*:

```javascript
POST http://sistema-destino.exemplo.com/positions

{
  "auth": "8e0e5rvj2501rp",
  "positions": [
    {
      "vehicle": "TST-1234",
      "timestamp": "2017-02-01T12:00:00-0200",
      "lat": -23.004388,
      "lng": -47.116368
    },
    {
      "vehicle": "TST-1234",
      "timestamp": "2017-02-01T12:00:01-0200",
      "lat": -23.004388, 
      "lng": -47.116368
    },
    {
      "vehicle": "TST-9999",
      "timestamp": "2017-02-01T12:00:01-0200",
      "lat": -23.004388, 
      "lng": -47.116368
    },
    ...
  ]
}
```

O sistema de destino deve responder com:

```javascript
HTTP 200
{
  "id": "65ral3a39n5e80"    // opcional, para facilitar rastreio/debug
}
```

## 3. Detalhes Técnicos

### 3.1. Códigos de Erro e Content-Type
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

    PRD: https://sistema-destino.exemplo.com/api/espelhamento
    HLG: https://sistema-destino.teste.exemplo.com/api/espelhamento


### 3.2. Autenticação
Este protocolo utiliza um elemento `"auth"` dentro da requisição. Seu conteúdo é apenas um token que identifica o sistema de origem. O sistema de destino deve manter um registro de tokens; a forma como isto é implementado não é relevante aqui. De fato, se o sistema destino quiser usar uma abordagem com usuários e senhas, basta definir que o token é da forma "username:password", ou uma hash desse valor.

Como é o sistema de destino que valida o token, é ele também que deve fornecê-lo para os sistemas-origem que irão enviar os dados.

O sistema destino deve usar este token para identificar o sistema origem (se houver vários enviando dados para ele) e autorizar as chamadas; também pode cruzar esta informação para saber se um certo sistema origem é de fato "dono" de um certo veículo.

Caso o token não seja válido, a resposta deve ser `HTTP 403` (Forbidden). Se estiver faltando, deve ser `HTTP 401` (Unauthorized).

Os tokens devem ser diferentes entre os ambientes de produção e homologação, para evitar que acessos de um tipo sejam por engano processados pelo outro.

### 3.3. Validação de Veículos
Quando um sistema origem envia dados, ele identifica o veículo através de uma string (no exemplo acima, há dois veículos, "TST-1234" e "TST-9999"). Se o sistema destino desejar, ele pode validar se o veículo informado realmente existe e pertence ao sistema origem. Se não existir/pertencer, o resultado deve ser `HTTP 400`, com um corpo indicando este erro:

    HTTP 400
    {"error": "NO_SUCH_VEHICLE"}
  
## 4. Detalhamento da API

### 4.1. API para envio de posições

```javascript
POST http://sistema-destino.exemplo.com/positions

{
  "auth": "8e0e5rvj2501rp",
  "positions": [
    {
      "vehicle": "TST-1234",
      "timestamp": "2017-02-01T12:00:00-0200",
      "lat": -23.004388,
      "lng": -47.116368
    },
    {
      "vehicle": "TST-1234",
      "timestamp": "2017-02-01T12:00:01-0200",
      "lat": -23.004388, 
      "lng": -47.116368
    },
    {
      "vehicle": "TST-9999",
      "timestamp": "2017-02-01T12:00:01-0200",
      "lat": -23.004388, 
      "lng": -47.116368
    },
    ...
  ]
}
```

O request acima é um exemplo para compartilhar três posições: duas do veículo **TST-1234** e uma do veículo **TST-9999**.

Cada posição contém:


Propriedade | Descrição
----------- | ---------
`vehicle`   | **OBRIGATÓRIO**. Placa ou identificador único do veículo. Note que este código é *alfanumérico*. Em particular, *NÃO DEVE* ser um ID gerado por sequence em uma tabela no sistema destino; isto amarra e engessa os bancos de dados dos sistemas.
`timestamp` | **OBRIGATÓRIO**. A data/hora da posição. O formato é sempre ISO-8601: `YYYY-MM-DDTHH:mm:ss-0000` (onde "T" é uma literal e -0000 indica um offset UTC. Pode-se usar o offset "Z" para referenciar UTC).
 `lat, lng` | **OBRIGATÓRIOS**. A latitude/longitude sendo relatada.

**Resultado**

Espera-se que o sistema destino receba estas informações e siga pelo menos os seguintes passos:

* Valide o token
* Valide que os dados passados estão no formato correto e (opcionalmente) se são de veículos conhecidos/permitidos
* Grave as posições informadas. Se alguma posição já existir (isto é, o sistema de origem está re-enviando dados), o sistema destino pode aceitar as novas posições (apagando/atualizando as antigas), ou desprezá-las, mas **NÃO DEVE** retornar um erro.
* Grave a data do recebimento. Isto é, para um certo veículo, existe a lista de posições, e existe a data da última chamada que o sistema origem fez com dados deste veículo. 
* Opcionalmente, retorne um ID de mensagem, se o processamento dos dados não for imediato (é comum que os dados sejam guardados em um local temporário, como uma fila de mensagens, antes de serem inseridos no banco de dados; neste caso, retornar o ID da mensagem gerada ajuda a rastrear as chamadas e investigar bugs).

Quando o sistema origem recebe uma resposta de sucesso, ele assume que todos os dados enviados foram recebidos e armazenados. Dessa forma, o próprio sistema origem pode controlar quais dados ele deve enviar na próxima chamada.
