# ARQUITETURA PROJETO CARTÃO

## Visão geral

A ideia dessa arquitetura é suportar o fluxo de solicitação de um cartão de crédito onde o cliente pode escolher alguns benefícios durante a jornada. Durante o processo o sistema precisa validar se o cliente é elegível para a oferta escolhida e também verificar se os benefícios selecionados são compatíveis com essa oferta.

Para organizar melhor o fluxo, a solução foi pensada usando **microserviços e comunicação por eventos**. Assim cada serviço fica responsável por uma parte do processo e a comunicação entre eles acontece de forma assíncrona através de mensageria (representada no diagrama como Kafka/Rabbit).

Esse modelo ajuda a manter os serviços desacoplados e facilita a evolução da solução.

---
# Desenho Arquitetura

https://imgur.com/a/XnmmH7B

---
# Fluxo da proposta

Tudo começa quando o cliente envia uma solicitação contendo algumas informações como:

* dados pessoais
* renda
* investimentos
* tempo de conta
* oferta escolhida
* benefícios selecionados

Essa requisição chega primeiro no serviço responsável por registrar a proposta e iniciar o fluxo.

A partir desse ponto os próximos passos acontecem através de eventos publicados na fila de mensageria.

---

# Serviços da arquitetura

## MS-Proposta

Esse serviço é a porta de entrada do fluxo.

Ele recebe os dados da solicitação do cliente, faz uma validação inicial e registra a proposta. Depois disso publica um evento indicando que uma nova proposta foi criada para que os próximos serviços possam continuar o processamento.

---

## MS-Elegibilidade

Esse serviço é responsável por verificar se o cliente pode contratar a oferta escolhida.

As regras são as seguintes:

**Oferta A**

* renda maior que R$ 1.000

**Oferta B**

* renda maior que R$ 15.000
* investimentos maiores que R$ 5.000

**Oferta C**

* renda maior que R$ 50.000
* conta corrente com mais de 2 anos

Se o cliente não atender aos critérios, a proposta é recusada. Nesse caso um evento é publicado informando que a proposta foi negada.

Esse evento pode ser consumido por outros serviços, como o de notificação e o de histórico.

---

## MS-Cartão

Quando o cliente é considerado elegível, o fluxo continua para o serviço responsável pela criação do cartão.

Esse serviço cria a conta do cartão vinculada ao cliente e publica um evento informando que o cartão foi criado com sucesso.

---

## MS-Benefícios

Depois da criação do cartão entra o serviço responsável por validar e ativar os benefícios escolhidos pelo cliente.

Algumas regras importantes são aplicadas aqui:

* cashback não pode ser selecionado junto com pontos
* pontos não pode ser selecionado junto com cashback
* seguro viagem só está disponível para a oferta C
* sala VIP só está disponível para ofertas B e C

Se os benefícios escolhidos forem válidos, eles são ativados e um novo evento é publicado.

---

## MS-Notificação

Esse serviço é responsável por informar o cliente sobre o andamento da proposta.

Ele pode consumir eventos do fluxo e enviar notificações quando algo importante acontece, por exemplo:

* proposta recusada
* cartão criado
* benefícios ativados
* conclusão da solicitação

---

## MS-Histórico

Esse serviço registra os eventos gerados ao longo do processo.

A ideia é manter um histórico da proposta para fins de auditoria e também para facilitar análises futuras sobre o fluxo.

---

# Comunicação entre serviços

A comunicação entre os serviços acontece através de mensageria (Kafka/Rabbit).

Isso permite que cada serviço processe os eventos no seu próprio tempo, sem depender diretamente dos outros serviços. Também ajuda em casos de falha, já que as mensagens ficam armazenadas na fila até serem processadas.

---

# Segurança

Como o sistema trabalha com dados sensíveis, alguns cuidados precisam ser considerados, como:

* criptografia de informações sensíveis
* evitar expor dados pessoais em logs
* controle de acesso entre serviços

---

# Resultado da proposta

No final do processo o cliente recebe o status da proposta informando se o cartão foi criado e quais benefícios foram ativados.

Um exemplo de resposta seria algo assim:

```json
{   
  "clienteId": 12,
  "propostaId": 123,
  "status": "APROVADA",
  "cartaoCriado": true,
  "beneficios": [
    "SALA_VIP",
    "SEGURO_VIAGEM"
  ]
}
```
