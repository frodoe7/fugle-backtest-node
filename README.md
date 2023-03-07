# Fugle Backtest

[![NPM version][npm-image]][npm-url]
[![Build Status][action-image]][action-url]
<!-- [![Coverage Status][codecov-image]][codecov-url] -->

> A trading strategy backtesting library in Node.js based on [Danfo.js](https://github.com/javascriptdata/danfojs) and inspired by [backtesting.py](https://github.com/kernc/backtesting.py).

## Installation

```sh
$ npm install --save @fugle/backtest
```

## Importing

```js
// Using Node.js `require()`
const { Backtest, Strategy } = require('@fugle/backtest');

// Using ES6 imports
import { Backtest, Strategy } from '@fugle/backtest';
```

## Quick Start

The following example use [technicalindicators](https://github.com/anandanand84/technicalindicators) to calculate the indicators and signals, but you can replace it with any library.


```js
import { Backtest, Strategy } from '@fugle/backtest';
import { SMA, CrossUp, CrossDown } from 'technicalindicators';

class TestStrategy extends Strategy {
  init() {
    const sma60 = SMA.calculate({
      period: 60,
      values: this.data['close'].values,
    });
    this.addIndicator('SMA60', sma60);

    const crossUp = CrossUp.calculate({
      lineA: this.data['close'].values,
      lineB: this.getIndicator('SMA60'),
    });
    this.addSignal('CrossUp', crossUp);

    const crossDown = CrossDown.calculate({
      lineA: this.data['close'].values,
      lineB: this.getIndicator('SMA60'),
    });
    this.addSignal('CrossDown', crossDown);
  }

  next(ctx) {
    const { index, signals } = ctx;
    if (index === 0) this.buy({ size: 1000 });
    if (index < 60) return;
    if (signals.get('CrossDown')) this.sell({ size: 1000 });
    if (signals.get('CrossUp')) this.buy({ size: 1000 });
  }
}

const data = require('./data.json');  // historical OHLCV data

const backtest = new Backtest(data, TestStrategy, {
  cash: 1000000,
  tradeOnClose: true,
});

backtest.run()        // run the backtest
  .then(results => {
    results.print();  // print the results
    results.plot();   // plot the equity curve
  });
```

Results in:

```
╔════════════════════════╤════════════╗
║ Start                  │ 2020-01-02 ║
╟────────────────────────┼────────────╢
║ End                    │ 2022-12-30 ║
╟────────────────────────┼────────────╢
║ Duration               │ 1093       ║
╟────────────────────────┼────────────╢
║ Exposure Time [%]      │ 99.863946  ║
╟────────────────────────┼────────────╢
║ Equity Final [$]       │ 1216000    ║
╟────────────────────────┼────────────╢
║ Equity Peak [$]        │ 1682500    ║
╟────────────────────────┼────────────╢
║ Return [%]             │ 21.6       ║
╟────────────────────────┼────────────╢
║ Buy & Hold Return [%]  │ 32.300885  ║
╟────────────────────────┼────────────╢
║ Return (Ann.) [%]      │ 6.935051   ║
╟────────────────────────┼────────────╢
║ Volatility (Ann.) [%]  │ 17.450299  ║
╟────────────────────────┼────────────╢
║ Sharpe Ratio           │ 0.397417   ║
╟────────────────────────┼────────────╢
║ Sortino Ratio          │ 0.660789   ║
╟────────────────────────┼────────────╢
║ Calmar Ratio           │ 0.215082   ║
╟────────────────────────┼────────────╢
║ Max. Drawdown [%]      │ -32.243685 ║
╟────────────────────────┼────────────╢
║ Avg. Drawdown [%]      │ -4.486974  ║
╟────────────────────────┼────────────╢
║ Max. Drawdown Duration │ 708        ║
╟────────────────────────┼────────────╢
║ Avg. Drawdown Duration │ 66         ║
╟────────────────────────┼────────────╢
║ # Trades               │ 32         ║
╟────────────────────────┼────────────╢
║ Win Rate [%]           │ 18.75      ║
╟────────────────────────┼────────────╢
║ Best Trade [%]         │ 95.9184    ║
╟────────────────────────┼────────────╢
║ Worst Trade [%]        │ -10.3245   ║
╟────────────────────────┼────────────╢
║ Avg. Trade [%]         │ 1.847315   ║
╟────────────────────────┼────────────╢
║ Max. Trade Duration    │ 308        ║
╟────────────────────────┼────────────╢
║ Avg. Trade Duration    │ 53         ║
╟────────────────────────┼────────────╢
║ Profit Factor          │ 2.293327   ║
╟────────────────────────┼────────────╢
║ Expectancy [%]         │ 3.230916   ║
╟────────────────────────┼────────────╢
║ SQN                    │ 0.579594   ║
╚════════════════════════╧════════════╝
```

![](./assets/equity-curve.png)
![](./assets/list-of-trades.png)

## Usage

To perform backtesting, you need to prepare historical data, implement a trading strategy, and then run a backtest on that strategy to obtain the results.

### Preparing historical data

First, prepare the historical OHLCV (Open, High, Low, Close, Volume) data of any financial instrument (such as stocks, futures, forex, cryptocurrencies, etc.). The input historical data will be converted to Danfo.js [DataFrame](https://danfo.jsdata.org/api-reference/dataframe), and the data format can be either `Array<Candle>` or `CandleList` type as follows:

```ts
interface Candle {
  date: string;
  open: number;
  high: number;
  low: number;
  close: number;
  volume?: number;
}

interface CandleList {
  date: string[];
  open: number[];
  high: number[];
  low: number[];
  close: number[];
  volume?: number[];
}

type HistoricalData = Array<Candle> | CandleList;
```

### Implementing trading strategy

You can implement your own trading strategy by inheriting the `Strategy` class and overriding its two abstract methods:

- `Strategy.init(data)`: This method is called before running the strategy. You can pre-calculate all indicators and signals that the strategy depends on.
- `Strategy.next(context)`: This method will be iteratively called when running the strategy with the `Backtest` instance, and the `context` parameter represents the current candle and technical indicators and signals. You can decide whether to make buy or sell actions based on the current price, indicators, and signals.

Here's an example of implementing a simple moving average (SMA) strategy. The strategy sets the SMA parameter period to 20 by default, and when the closing price of a stock or commodity crosses above the moving average, it buys 1 trading unit. Conversely, when the closing price crosses below the moving average, it sells 1 trading unit.

```js
import { Backtest, Strategy } from '@fugle/backtest';
import { SMA, CrossUp, CrossDown } from 'technicalindicators';

class SmaStrategy extends Strategy {
  params = { period: 20 };

  init(data) {
    const sma = SMA.calculate({
      period: this.params.period,
      values: this.data['close'].values,
    });
    this.addIndicator('SMA', sma);

    const crossUp = CrossUp.calculate({
      lineA: this.data['close'].values,
      lineB: this.getIndicator('SMA'),
    });
    this.addSignal('CrossUp', crossUp);

    const crossDown = CrossDown.calculate({
      lineA: this.data['close'].values,
      lineB: this.getIndicator('SMA'),
    });
    this.addSignal('CrossDown', crossDown);
  }

  next(ctx) {
    const { index, signals } = ctx;
    if (index < this.params.period) return;
    if (signals.get('CrossUp')) this.buy({ size: 1000 });
    if (signals.get('CrossDown')) this.sell({ size: 1000 });
  }
}
```

### Running the backtest

After preparing historical data and implementing the trading strategy, you can run the backtest. Calling the `Backtest.run()` method will execute the backtest and return a `Stats` instance, which includes the simulation results of our strategy and related statistical data.

```js
const backtest = new Backtest(data, SmaStrategy, {
  cash: 1000000,
  tradeOnClose: true,
});

backtest.run()        // run the backtest
  .then(results => {
    results.print();  // print the results
    results.plot();   // plot the equity curve
  });
```

### Optimizing the parameters

In the above strategy, we provided a variable parameter `params.period`, which represents the period of the moving average. We can optimize this parameter, or find the best combination of multiple parameters, by calling the `Backtest.optimize()` method. Setting the `params` option in this method can change the parameter settings provided by the `Strategy`, and `Backtest.optimize()` will return the best combination of parameters provided.

```js
backtest.optimize({
  params: {
    period: [5, 10, 20, 60],
  },
})
  .then(results => {
    results.print();  // print out the results of the optimized parameters
    results.plot();   // plot the equity curve of the optimized parameters
  });
```

## Documentation

See [`/doc/fugle-backtest.md`](./doc/fugle-backtest.md) for Node.js-like documentation of `@fugle/backtest` classes.

## License

[MIT](LICENSE)

[npm-image]: https://img.shields.io/npm/v/@fugle/backtest.svg
[npm-url]: https://npmjs.com/package/@fugle/backtest
[action-image]: https://img.shields.io/github/actions/workflow/status/fugle-dev/fugle-backtest-node/node.js.yml?branch=master
[action-url]: https://github.com/fugle-dev/fugle-backtest-node/actions/workflows/node.js.yml
<!-- [codecov-image]: https://img.shields.io/codecov/c/github/fugle-dev/fugle-backtest-node.svg
[codecov-url]: https://codecov.io/gh/fugle-dev/fugle-backtest-node -->
