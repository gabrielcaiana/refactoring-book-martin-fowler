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

