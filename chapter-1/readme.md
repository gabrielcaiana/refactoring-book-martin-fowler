### Introdução
----
O projeto consiste em uma companhia de atores que sai para participar de varios
eventos apresentando peças. Em geral, os clientes solicitarão algumas peças e
a companhia cobrará deles com base no número de espectadores e no tipo de peça
encenada. Atualmente há dois tipos de peças que a companhia apresenta: **trágédias**
e **comédias**. Além de apresentar uma conta pela apresentação, a companhia dá "créditos
por volume" aos clientes, os que podem ser usados como descontos em futuras apresentaçōes -
pense nisso como um mecanismo de fidelidade do cliente.

```javascript
const mockInvoice = require('./mocks/invoices.json');
const mockPlays = require('./mocks/plays.json');

function statement(invoice, plays) {
  let totalAmount = 0;
  let volumeCredits = 0;
  let result = `Statement for ${invoice.customer}\n`;
  const format = new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    minimumFractionDigits: 2
  });

  for (let perf of invoice.performances) {
    const play = plays[perf.playID];
    let thisAmount = 0;
    switch (play.type) {
      case 'tragedy':
        thisAmount = 40000;
        if (perf.audience > 30) {
          thisAmount += 1000 * (perf.audience - 30);
        }
        break;
      case 'comedy':
        thisAmount = 30000;
        if (perf.audience > 20) {
          thisAmount += 10000 + 500 * (perf.audience - 20);
        }
        thisAmount += 300 * perf.audience;
        break;
      default:
        throw new Error(`unknown type: ${play.type}`);
    }

    // add volume credits
    volumeCredits += Math.max(perf.audience - 30, 0);
    // add extra credit for every ten comedy attendees
    if ('comedy' === play.type) volumeCredits += Math.floor(perf.audience / 5);
    // print line for this order
    result += ` ${play.name}: ${format.format(thisAmount / 100)} (${perf.audience} seats)\n`;
    totalAmount += thisAmount;
  }

  result += `Amount owed is ${format.format(totalAmount / 100)}\n`;
  result += `You earned ${volumeCredits} credits\n`;

  return result;
}

console.log(statement(mockInvoice, mockPlays))
```

A execução desse código nos arquivos de dados de teste anteriores resulta
na seguinte saída:

```javascript
  Statement for BigCo
  Hamlet: $650.00 (55 seats)
  A young lady like you: $580.00 (35 seats)
  Othello: $500.00 (40 seats)
  Amount owed is $1,730.00
  You earned 47 credits
```

## Refatoração

Extraíndo o método switch para um função isolada cujo a responsabilidade será apenas realizar o calculo dos valores e retorna-lo
no final:

```javascript
  function amountFor(perf, play) {
    let result = 0;
    switch (play.type) {
      case 'tragedy':
        result = 40000;
        if (perf.audience > 30) {
          result += 1000 * (perf.audience - 30);
        }
        break;
      case 'comedy':
        result = 30000;
        if (perf.audience > 20) {
          result += 10000 + 500 * (perf.audience - 20);
        }
        result += 300 * perf.audience;
        break;
      default:
        throw new Error(`unknown type: ${play.type}`);
    }
    return result;
  }
```

com isso a função statement vai utilizá-lo da seguinte forma:

```javascript
  function statement(invoice, plays) {
    let totalAmount = 0;
    let volumeCredits = 0;
    let result = `Statement for ${invoice.customer}\n`;
    const format = new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: 2
    });

    for (let perf of invoice.performances) {
      const play = plays[perf.playID];

      // add new function amountFor
      let thisAmount = amountFor(perf, play);
    
      // add volume credits
      volumeCredits += Math.max(perf.audience - 30, 0);
      // add extra credit for every ten comedy attendees
      if ('comedy' === play.type) volumeCredits += Math.floor(perf.audience / 5);
      // print line for this order
      result += ` ${play.name}: ${format.format(thisAmount / 100)} (${perf.audience} seats)\n`;
      totalAmount += thisAmount;
    }
    result += `Amount owed is ${format.format(totalAmount / 100)}\n`;
    result += `You earned ${volumeCredits} credits\n`;
    return result;
  }
```

----

Para aplicar mais clareza a função ```amountFor``` altero o nome da variável ```thisAmount``` para ```result```
com isso fica claro o papel dela dentro da função:

```javascript
 function amountFor(perf, play) {
    let result = 0;
    switch (play.type) {
      case 'tragedy':
        result = 40000;
        if (perf.audience > 30) {
          result += 1000 * (perf.audience - 30);
        }
        break;
      case 'comedy':
        result = 30000;
        if (perf.audience > 20) {
          result += 10000 + 500 * (perf.audience - 20);
        }
        result += 300 * perf.audience;
        break;
      default:
        throw new Error(`unknown type: ${play.type}`);
    }
    return result;
  }
```

O próximo passo é alterar o nome do parâmetro ```perf``` para ```aPerformance``` com isso
fica mais claro qual é seu papel dentro da função:

```javascript
 function amountFor(aPerformance, play) {
    let result = 0;
    switch (play.type) {
      case 'tragedy':
        result = 40000;
        if (aPerformance.audience > 30) {
          result += 1000 * (aPerformance.audience - 30);
        }
        break;
      case 'comedy':
        result = 30000;
        if (aPerformance.audience > 20) {
          result += 10000 + 500 * (aPerformance.audience - 20);
        }
        result += 300 * aPerformance.audience;
        break;
      default:
        throw new Error(`unknown type: ${play.type}`);
    }
    return result;
  }
```

### Removendo a variável play

observando os parâmetros de ```amountFor```, observa-se de onde eles vêm, ```aPerformance```
é proveniente da variável do laço, portanto mudará naturalmente a cada iteração. Entretando,
```play``` é obtido da apresentação, portanto não é necessário passá-lo como parâmetro.
Podemos simplesmente o recalcular em ```amountFor```.

```javascript
  function playFor(aPerformance) {
    return plays[aPerformance.playID];
  }
```

Com isso agora a função ```statement``` fica da seguinte forma:

```javascript
  function statement(invoice) {
    let totalAmount = 0;
    let volumeCredits = 0;
    let result = `Statement for ${invoice.customer}\n`;
    const format = new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: 2
    });

    for (let perf of invoice.performances) {
      const play = playFor(perf);
      let thisAmount = amountFor(perf, play);
    
      // add volume credits
      volumeCredits += Math.max(perf.audience - 30, 0);
      // add extra credit for every ten comedy attendees
      if ('comedy' === play.type) volumeCredits += Math.floor(perf.audience / 5);
      // print line for this order
      result += ` ${play.name}: ${format.format(thisAmount / 100)} (${perf.audience} seats)\n`;
      totalAmount += thisAmount;
    }
    result += `Amount owed is ${format.format(totalAmount / 100)}\n`;
    result += `You earned ${volumeCredits} credits\n`;
    return result;
  }
```

Podemos melhorar mais ainda a função ```statement``` removendo o parâmetro ```play``` e
mudando para **declaração de função** em ```amountFor```, com isso a função fica da seguinte forma:

```javascript
function amountFor(aPerformance) {
  let result = 0;
  switch (playFor(aPerformance).type) {
    case 'tragedy':
      result = 40000;
      if (aPerformance.audience > 30) {
        result += 1000 * (aPerformance.audience - 30);
      }
      break;
    case 'comedy':
      result = 30000;
      if (aPerformance.audience > 20) {
        result += 10000 + 500 * (aPerformance.audience - 20);
      }
      result += 300 * aPerformance.audience;
      break;
    default:
      throw new Error(`unknown type: ${playFor(aPerformance).type.type}`);
  }
  return result;
}
```

```javascript
  function statement(invoice) {
    let totalAmount = 0;
    let volumeCredits = 0;
    let result = `Statement for ${invoice.customer}\n`;
    const format = new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: 2
    });

    for (let perf of invoice.performances) {
      let thisAmount = amountFor(perf);
    
      // add volume credits
      volumeCredits += Math.max(perf.audience - 30, 0);
      // add extra credit for every ten comedy attendees
      if ('comedy' === playFor(perf).type) volumeCredits += Math.floor(perf.audience / 5);
      // print line for this order
      result += ` ${playFor(perf).name}: ${format.format(thisAmount / 100)} (${perf.audience} seats)\n`;
      totalAmount += thisAmount;
    }
    result += `Amount owed is ${format.format(totalAmount / 100)}\n`;
    result += `You earned ${volumeCredits} credits\n`;
    return result;
  }
```

Por fim podemos remover a variável local ```thisAmount``` e utilizar do conceito de
**internalizar variável (inline variable)** com isso o código fica da seguinte forma:

```javascript
  function statement(invoice) {
    let totalAmount = 0;
    let volumeCredits = 0;
    let result = `Statement for ${invoice.customer}\n`;
    const format = new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: 2
    });

    for (let perf of invoice.performances) {  
      // add volume credits
      volumeCredits += Math.max(perf.audience - 30, 0);
      // add extra credit for every ten comedy attendees
      if ('comedy' === playFor(perf).type) volumeCredits += Math.floor(perf.audience / 5);
      // print line for this order
      result += ` ${playFor(perf).name}: ${format.format(amountFor(perf) / 100)} (${perf.audience} seats)\n`;
      totalAmount += amountFor(perf);
    }
    result += `Amount owed is ${format.format(totalAmount / 100)}\n`;
    result += `You earned ${volumeCredits} credits\n`;
    return result;
  }
```


