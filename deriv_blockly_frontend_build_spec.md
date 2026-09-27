# Deriv + Google Blockly Frontend Bot Builder

## Complete Build Specification for a Functional HTML/CSS/JavaScript Web App

> **Purpose:** This document is a build specification for an autonomous
> coding agent such as Hermes. The agent should use it as the
> implementation contract for building a browser-only visual trading-bot
> platform based on Google Blockly and the Deriv API.
>
> **Target stack:** HTML5 + CSS3 + modern browser JavaScript (ES
> modules), Google Blockly, WebSocket/Fetch, browser storage. No custom
> backend.
>
> **Important:** Deriv's current developer documentation contains both
> newer authenticated Options API workflows and older/legacy WebSocket
> v3 workflows. The implementation must verify the exact API flow
> supported by the registered Deriv application before coding the
> trading layer. Do not silently mix authentication or endpoint models.

------------------------------------------------------------------------

# 1. Product Definition

Build a single-page web application that lets a user:

1.  Connect a Deriv account.
2.  Select demo or real account where supported by the authentication
    flow.
3.  Load live Deriv market metadata.
4.  Select an underlying symbol.
5.  View live ticks and historical ticks.
6.  Build trading logic visually using Google Blockly.
7.  Generate JavaScript from the Blockly workspace.
8.  Validate the generated bot before execution.
9.  Run the bot against Deriv market data.
10. Request a price proposal.
11. Buy a contract only when the bot's rules permit it.
12. Monitor the open contract.
13. Detect win/loss/expiry/error states.
14. Optionally sell an open contract early when the contract/API
    supports it.
15. Stop the bot safely.
16. Unsubscribe from streams cleanly.
17. Show account balance, trades, logs, errors and runtime state.
18. Save and load bot projects locally.
19. Export/import Blockly XML or JSON project data.
20. Operate in simulation/paper mode without sending live orders.
21. Prevent accidental duplicate orders and concurrent trade races.

The application is a **visual programming environment**, not merely a
Blockly editor.

------------------------------------------------------------------------

# 2. Core Architecture

Use this architecture:

``` text
Browser
│
├── index.html
├── css/
│   ├── app.css
│   ├── layout.css
│   ├── blocks.css
│   └── responsive.css
│
├── src/
│   ├── app.js
│   │
│   ├── config/
│   │   ├── app-config.js
│   │   └── contract-config.js
│   │
│   ├── deriv/
│   │   ├── deriv-client.js
│   │   ├── deriv-connection.js
│   │   ├── deriv-auth.js
│   │   ├── deriv-market.js
│   │   ├── deriv-trading.js
│   │   ├── deriv-subscriptions.js
│   │   └── deriv-errors.js
│   │
│   ├── engine/
│   │   ├── bot-engine.js
│   │   ├── bot-state.js
│   │   ├── bot-validator.js
│   │   ├── trade-manager.js
│   │   ├── signal-engine.js
│   │   └── risk-manager.js
│   │
│   ├── blockly/
│   │   ├── workspace.js
│   │   ├── toolbox.js
│   │   ├── blocks/
│   │   │   ├── bot.js
│   │   │   ├── market.js
│   │   │   ├── data.js
│   │   │   ├── conditions.js
│   │   │   ├── indicators.js
│   │   │   ├── logic.js
│   │   │   ├── trading.js
│   │   │   ├── risk.js
│   │   │   └── utility.js
│   │   └── generators/
│   │       └── javascript.js
│   │
│   ├── storage/
│   │   ├── project-store.js
│   │   ├── settings-store.js
│   │   └── secure-session.js
│   │
│   ├── ui/
│   │   ├── dashboard.js
│   │   ├── chart.js
│   │   ├── blockly-panel.js
│   │   ├── account-panel.js
│   │   ├── trade-panel.js
│   │   ├── log-panel.js
│   │   └── notifications.js
│   │
│   └── utils/
│       ├── ids.js
│       ├── time.js
│       ├── numbers.js
│       ├── events.js
│       └── logger.js
│
└── assets/
```

Do not put all functionality into one `script.js`.

------------------------------------------------------------------------

# 3. Critical Security Model

## 3.1 Browser-only limitation

There is no private backend.

Therefore:

-   Never put a Deriv client secret in JavaScript.
-   Never hard-code a user's API token into source code.
-   Never commit tokens to Git.
-   Never place secrets in the HTML.
-   Never send a token to an unrelated server.
-   Never create a fake backend endpoint.
-   Never claim browser-only code provides server-grade secret
    protection.

For a browser application, use Deriv's supported OAuth flow for
production web authentication.

Deriv currently documents OAuth 2.0 and Personal Access Token (PAT)
authentication. OAuth is the appropriate model for browser web
applications because it lets the user authorize the application without
giving the application their password. Deriv specifically describes
OAuth as the fit for web dashboards/browser apps. PAT is intended for
situations where manual token entry is acceptable.

## 3.2 OAuth production flow

Implement:

``` text
User clicks Connect Deriv
        ↓
Application generates OAuth state
        ↓
Application redirects to Deriv authorization page
        ↓
User authenticates with Deriv
        ↓
Deriv redirects to registered redirect URI
        ↓
Application receives authorization code
        ↓
Authorization code is exchanged according to Deriv's supported OAuth flow
        ↓
Application obtains access/session information
        ↓
Application establishes authenticated trading connection
```

Do not put a client secret in the browser.

If Deriv's current OAuth implementation requires a confidential exchange
that cannot safely be performed by a static frontend, the agent must not
invent a workaround. Instead, document that a minimal backend/serverless
token exchange is required for production.

## 3.3 Development PAT mode

Optionally implement a clearly marked development-only manual PAT mode.

UI:

``` text
Authentication
[ OAuth - Recommended ]
[ Development PAT ]

PAT input:
[________________________]

[ Connect ]

Warning:
Never paste a real trading token into an untrusted website.
```

Do not store PAT permanently in localStorage.

Prefer an in-memory variable and clear it on logout/page unload.

------------------------------------------------------------------------

# 4. Deriv API Documentation Rule

The agent must treat the official Deriv developer documentation as
authoritative.

Before implementing API calls, verify:

-   current endpoint
-   request schema
-   response schema
-   authentication requirement
-   subscription behavior
-   request limits
-   supported contract types
-   account currency
-   demo/real account behavior
-   current application registration requirements

Do not copy old snippets blindly.

Current official documentation describes:

-   `active_symbols`
-   `contracts_for`
-   `ticks`
-   `ticks_history`
-   `proposal`
-   `buy`
-   `proposal_open_contract`
-   `sell`
-   account/balance operations
-   subscription management

The official complete workflow is conceptually:

``` text
authenticate
→ active_symbols
→ ticks
→ proposal
→ buy
→ proposal_open_contract
→ optional sell
→ portfolio/balance
→ cleanup
```

------------------------------------------------------------------------

# 5. Connection Layer

Create one reusable `DerivConnection` class.

## Required responsibilities

-   open WebSocket
-   close WebSocket
-   reconnect
-   track connection state
-   generate request IDs
-   map request IDs to Promises
-   resolve/reject requests
-   route subscription messages
-   detect API errors
-   detect socket errors
-   timeout requests
-   send heartbeat/ping when necessary
-   clean pending requests on close

Do not create a new WebSocket for every tick request.

------------------------------------------------------------------------

# 6. Request/Response Correlation

Every request should receive a unique `req_id`.

Example:

``` js
let nextReqId = 1;

function createReqId() {
  return nextReqId++;
}
```

Use:

``` js
const reqId = createReqId();

ws.send(JSON.stringify({
  active_symbols: "brief",
  req_id: reqId
}));
```

Incoming response:

``` js
const data = JSON.parse(event.data);

if (data.req_id) {
  const pending = pendingRequests.get(data.req_id);

  if (pending) {
    pending.resolve(data);
    pendingRequests.delete(data.req_id);
  }
}
```

Never depend on response arrival order.

Bad:

``` js
await sendRequest(...);
await sendRequest(...);
```

is fine only if `sendRequest` actually correlates responses.

Never assume:

``` text
request A
request B
response A
response B
```

WebSocket messages can arrive asynchronously.

------------------------------------------------------------------------

# 7. Promise-based WebSocket API

Create:

``` js
async function request(payload, options = {}) {
    // validate socket
    // assign req_id
    // create Promise
    // register timeout
    // send request
    // resolve/reject from onmessage
}
```

Example:

``` js
const result = await deriv.request({
    balance: 1,
    subscribe: 1
});
```

The returned Promise must reject on:

-   API error
-   timeout
-   socket close
-   malformed response

------------------------------------------------------------------------

# 8. WebSocket Connection States

Use explicit states:

``` js
const ConnectionState = Object.freeze({
    DISCONNECTED: "disconnected",
    CONNECTING: "connecting",
    CONNECTED: "connected",
    AUTHENTICATING: "authenticating",
    AUTHENTICATED: "authenticated",
    RECONNECTING: "reconnecting",
    CLOSING: "closing",
    ERROR: "error"
});
```

UI must display the current state.

Example:

``` text
● Connected
● Authenticated
● Market stream active
● Bot running
```

------------------------------------------------------------------------

# 9. Reconnection

Implement exponential backoff:

``` text
1s
2s
4s
8s
16s
30s maximum
```

Do not reconnect forever without limits.

Reset retry count after successful connection.

When reconnecting:

1.  reconnect socket
2.  restore authentication/session according to the supported API
3.  restore required market subscriptions
4.  restore balance subscription
5.  restore open-contract monitoring if appropriate
6.  update UI
7.  resume bot only if the runtime policy allows it

Never automatically place a new trade merely because the socket
reconnects.

------------------------------------------------------------------------

# 10. Subscription Manager

Create a central subscription registry.

``` js
subscriptions = {
    ticks: new Map(),
    balance: null,
    contracts: new Map(),
    proposals: new Map()
};
```

Track:

-   subscription ID
-   request type
-   symbol
-   owner
-   created time
-   active state
-   callback

Required methods:

``` js
subscribeTicks(symbol, handler)
unsubscribe(subscriptionId)
unsubscribeTicks(symbol)
unsubscribeAll()
```

Use the Deriv `forget` mechanism where supported to stop subscriptions.

Never leave old tick subscriptions running when the user changes
markets.

------------------------------------------------------------------------

# 11. Market Data

## 11.1 Active symbols

Call `active_symbols` and normalize the response.

Create internal structure:

``` js
{
    symbol: "1HZ100V",
    displayName: "...",
    market: "...",
    submarket: "...",
    marketDisplayName: "...",
    submarketDisplayName: "...",
    exchangeIsOpen: true,
    pipSize: 2,
    raw: originalSymbolObject
}
```

Do not hard-code the entire symbol list.

Populate the UI dynamically.

## 11.2 Market categories

Create dynamic groups based on returned metadata.

Examples:

``` text
Forex
Crypto
Commodities
Stock Indices
Synthetic Indices
Volatility Indices
Crash/Boom
Jump
Range
Step
Basket
Other
```

The grouping must be data-driven.

If Deriv changes its symbol catalog, the UI should still work.

------------------------------------------------------------------------

# 12. Contracts for Symbol

When the user selects a symbol:

``` js
await deriv.contractsFor(symbol);
```

Use `contracts_for` to determine what contracts are actually available
for that symbol.

Do not show every contract type as universally available.

Build:

``` js
availableContractsBySymbol[symbol]
```

The Blockly dropdown should only expose valid choices where possible.

------------------------------------------------------------------------

# 13. Tick Stream

Subscribe using the supported Deriv tick endpoint.

Internal event:

``` js
marketStreamState = {
    symbol: null,
    subscriptionId: null,
    prices: [],
    times: [],
    maxPoints: 5000,
    latest: null,
    connectedAt: null
};
```

Tick normalized format:

``` js
{
    symbol: "R_100",
    quote: 123.4567,
    epoch: 1775921672,
    time: new Date(1775921672 * 1000),
    pipSize: 2
}
```

Keep raw response separately for debugging.

------------------------------------------------------------------------

# 14. Historical Data

Use `ticks_history` for initial seeding.

Example conceptual request:

``` js
{
    ticks_history: symbol,
    count: 500,
    end: "latest",
    style: "ticks"
}
```

Then:

``` text
historical ticks
      ↓
normalize
      ↓
seed marketStreamState
      ↓
subscribe live ticks
      ↓
append live ticks
```

Avoid duplicate points.

Use epoch/time plus quote carefully because identical quotes can occur
at different timestamps.

------------------------------------------------------------------------

# 15. Candle Aggregation

Blockly should support candles even when the primary source is tick
data.

Create a candle aggregator.

Input:

``` js
{
    quote,
    epoch
}
```

Output:

``` js
{
    open,
    high,
    low,
    close,
    volume: null,
    startTime,
    endTime,
    interval
}
```

Supported intervals:

``` text
1s
5s
10s
15s
30s
1m
5m
15m
30m
1h
4h
1d
```

Only expose intervals that can be correctly derived from available data.

Do not pretend tick-derived volume is real exchange volume.

------------------------------------------------------------------------

# 16. Digit Contract Support

The initial trading focus is digit contracts.

Support:

``` text
DIGITOVER
DIGITUNDER
DIGITMATCH
DIGITDIFF
DIGITODD
DIGITEVEN
```

For the initial user-facing strategy set, prioritize:

``` text
Digit Over
Digit Under
```

Example conceptual mapping:

``` text
Digit Over 3
contract_type = DIGITOVER
barrier = "3"
```

``` text
Digit Under 7
contract_type = DIGITUNDER
barrier = "7"
```

Do not hard-code the payout.

Always request a proposal.

------------------------------------------------------------------------

# 17. Last Digit Extraction

Do not assume a fixed decimal precision.

Use the symbol's `pip_size` or equivalent metadata to determine the
displayed price precision.

Example:

``` js
function getLastDigit(quote, pipSize) {
    const decimals = Number(pipSize);

    const scaled = Math.round(
        Number(quote) * Math.pow(10, decimals)
    );

    return Math.abs(scaled) % 10;
}
```

The implementation must be validated against actual Deriv symbol
formatting.

Do not use:

``` js
String(price).slice(-1)
```

because that is incorrect for decimal values.

------------------------------------------------------------------------

# 18. Proposal Engine

A trade must normally follow:

``` text
strategy signal
      ↓
risk validation
      ↓
contract validation
      ↓
proposal request
      ↓
proposal received
      ↓
proposal validation
      ↓
buy
```

Create:

``` js
async function getProposal(params) {}
```

Parameters:

``` js
{
    amount,
    basis,
    contract_type,
    currency,
    duration,
    duration_unit,
    barrier,
    underlying_symbol
}
```

For digit contracts, the barrier is the predicted digit.

The current Deriv proposal documentation lists `DIGITOVER`,
`DIGITUNDER`, `DIGITMATCH`, `DIGITDIFF`, `DIGITODD`, and `DIGITEVEN`
among supported contract types.

------------------------------------------------------------------------

# 19. Proposal Subscription

Where supported and useful, use proposal subscriptions to monitor
changing prices.

But do not create unlimited proposal streams.

Track every proposal subscription and remove it when:

-   trade is abandoned
-   contract is bought
-   bot stops
-   market changes
-   connection closes

------------------------------------------------------------------------

# 20. Buy Engine

Create:

``` js
async function buyProposal(proposalId, price) {}
```

Before buying:

``` text
Is bot running?
Is a trade already pending?
Is another contract open?
Is account authenticated?
Is market selected?
Is contract supported?
Is stake valid?
Is risk limit valid?
Is proposal still valid?
Has user enabled live trading?
```

If any condition fails, do not buy.

------------------------------------------------------------------------

# 21. Trade State Machine

Use an explicit state machine.

``` text
IDLE
 ↓
SIGNAL_READY
 ↓
RISK_CHECK
 ↓
PROPOSAL_REQUESTING
 ↓
PROPOSAL_READY
 ↓
BUY_PENDING
 ↓
OPEN
 ↓
MONITORING
 ↓
WON / LOST / SOLD / EXPIRED
 ↓
SETTLED
 ↓
IDLE
```

Error states:

``` text
PROPOSAL_ERROR
BUY_ERROR
CONNECTION_ERROR
AUTH_ERROR
VALIDATION_ERROR
RISK_BLOCKED
```

Never allow:

``` text
OPEN → BUY_PENDING
```

for a second simultaneous trade unless explicitly supported.

------------------------------------------------------------------------

# 22. Duplicate Trade Protection

Implement:

``` js
let tradeLock = false;
```

Better:

``` js
tradeManager = {
    status: "idle",
    activeContractId: null,
    proposalId: null,
    requestId: null,
    tradeLock: false
};
```

Before buy:

``` js
if (tradeManager.tradeLock) {
    throw new Error("Trade already in progress");
}
```

Set lock before asynchronous buy processing.

Release it only after the operation reaches a safe state.

------------------------------------------------------------------------

# 23. Open Contract Monitoring

After buying, subscribe to:

``` text
proposal_open_contract
```

Track:

``` js
{
    contractId,
    status,
    isSold,
    isExpired,
    profit,
    payout,
    buyPrice,
    currentSpot,
    entrySpot,
    exitSpot,
    contractType,
    symbol
}
```

Use the actual fields returned by the current Deriv API.

Never guess response property names.

------------------------------------------------------------------------

# 24. Sell

Implement early selling only where the contract and API support it.

``` js
await deriv.sell({
    contract_id: contractId,
    price
});
```

Never automatically sell just because profit is positive unless the user
explicitly configured that behavior.

------------------------------------------------------------------------

# 25. Balance

Subscribe to account balance after authentication where supported.

UI:

``` text
Account
----------------
Account: DEMO
Currency: USD
Balance: 10000.00
```

Keep:

``` js
accountState = {
    loginid: null,
    currency: null,
    balance: null,
    isDemo: null,
    authenticated: false
};
```

Do not derive account identity from UI text.

------------------------------------------------------------------------

# 26. Bot Runtime

Create:

``` js
class BotEngine {
    start()
    stop()
    pause()
    resume()
    reset()
    getState()
}
```

Runtime states:

``` text
STOPPED
STARTING
RUNNING
PAUSED
STOPPING
ERROR
```

------------------------------------------------------------------------

# 27. Bot Execution Model

The Blockly program should not directly manipulate WebSocket internals.

Blockly generated code should call a stable runtime API.

Example:

``` js
await bot.market("R_100");

const tick = await bot.latestTick();

if (bot.digit(tick) > 3) {
    await bot.trade({
        type: "DIGITOVER",
        barrier: 3,
        stake: 1,
        duration: 1,
        durationUnit: "t"
    });
}
```

This abstraction is critical.

Blockly blocks generate bot-runtime calls, not raw WebSocket calls.

------------------------------------------------------------------------

# 28. Generated Code Architecture

Example:

``` js
async function generatedBot(bot) {

    await bot.selectMarket("R_100");

    while (bot.isRunning()) {

        const tick = await bot.waitForNextTick();

        const digit = bot.lastDigit(tick);

        if (digit > 3) {

            await bot.placeTrade({
                contractType: "DIGITOVER",
                barrier: 3,
                stake: 1,
                duration: 1,
                durationUnit: "t"
            });
        }
    }
}
```

Do not generate:

``` js
new WebSocket(...)
```

inside every block.

------------------------------------------------------------------------

# 29. Blockly Architecture

Use Google Blockly as a visual programming layer.

Blockly generates code strings; it does not execute the blocks itself.
The JavaScript generator exposes:

``` js
javascriptGenerator.workspaceToCode(workspace)
```

and individual block generators are registered through:

``` js
javascriptGenerator.forBlock["block_type"] = function(block, generator) {
    ...
};
```

The generated code must execute through a controlled application
runtime.

------------------------------------------------------------------------

# 30. Blockly Workspace

Create:

``` js
const workspace = Blockly.inject("blocklyDiv", {
    toolbox,
    trashcan: true,
    zoom: {
        controls: true,
        wheel: true,
        startScale: 1,
        maxScale: 2,
        minScale: 0.5
    },
    grid: {
        spacing: 20,
        length: 3,
        colour: "#ccc",
        snap: true
    }
});
```

Adjust styling to the final application theme.

------------------------------------------------------------------------

# 31. Toolbox Categories

Create these categories:

``` text
BOT
├── Start Bot
├── Stop Bot
├── Wait
└── Repeat

MARKET
├── Select Market
├── Current Market
├── Current Tick
├── Tick Price
├── Last Digit
└── Market Status

DATA
├── Historical Ticks
├── Tick Count
├── Price
├── Previous Price
├── Highest
├── Lowest
└── Average

INDICATORS
├── SMA
├── EMA
├── RSI
├── Volatility
├── Momentum
└── Custom Digit Statistics

LOGIC
├── if
├── if/else
├── comparison
├── boolean
├── AND
├── OR
└── NOT

DIGITS
├── Digit Over
├── Digit Under
├── Digit Match
├── Digit Differs
├── Digit Odd
└── Digit Even

TRADING
├── Get Proposal
├── Buy
├── Sell
├── Wait for Contract
├── Contract Status
└── Last Trade Result

RISK
├── Stake
├── Max Stake
├── Max Consecutive Losses
├── Daily Loss Limit
├── Cooldown
└── Stop After N Trades

VARIABLES
├── Set Variable
├── Get Variable
└── Counters

UTILITY
├── Log
├── Alert
├── Timestamp
├── Random
└── Math
```

------------------------------------------------------------------------

# 32. Main Bot Block

Create a single top-level block:

``` text
START BOT
    [setup]
    [strategy]
    [trade management]
```

Type:

``` js
main_block
```

It must be unique in the workspace.

If a second main block is created:

1.  detect it
2.  prevent duplicate execution
3.  show a warning
4.  optionally delete the duplicate

Never execute multiple top-level bots accidentally.

------------------------------------------------------------------------

# 33. Main Block Fields

Use:

``` text
Market
Category
Contract
Candle interval
Stake
Duration
Duration unit
Restart on buy/sell error
Restart last trade on error
```

Do not use typo-prone internal names such as:

``` text
frist_market
frist_catogory
```

Use clear names:

``` text
market
category
contractType
candleInterval
```

If compatibility with an existing project requires old names, create a
migration layer instead of spreading typos through the codebase.

------------------------------------------------------------------------

# 34. Dynamic Dropdowns

Market dropdowns must come from `globalMarketData` or the normalized
market store.

When market changes:

``` text
market changed
    ↓
load contracts_for
    ↓
filter supported contracts
    ↓
update category dropdown
    ↓
update contract dropdown
    ↓
validate existing blocks
```

When programmatically changing a Blockly field, trigger the appropriate
Blockly event/change mechanism rather than assuming the UI has
automatically reacted.

Never call:

``` js
field.addChangeListener(...)
```

when `field` may be null.

Always validate the field first.

------------------------------------------------------------------------

# 35. Contract-Specific Settings

Create dynamic trade settings.

## Digit Over / Under

``` js
{
    contractType,
    stake,
    duration,
    durationUnit,
    barrier
}
```

Example:

``` text
Digit Over
Stake: 1
Duration: 1
Unit: tick
Barrier: 3
```

## Digit Match / Differs

``` text
Barrier: 0–9
```

## Odd / Even

No digit barrier is required unless the current API says otherwise.

------------------------------------------------------------------------

# 36. Strategy Block Example

Block:

``` text
IF
    last digit of current tick
    >
    3
THEN
    Digit Over 3
```

Generated code:

``` js
if (bot.lastDigit(await bot.currentTick()) > 3) {
    await bot.placeTrade({
        contractType: "DIGITOVER",
        barrier: 3,
        stake: 1,
        duration: 1,
        durationUnit: "t"
    });
}
```

------------------------------------------------------------------------

# 37. Tick Statistics

Provide a reusable statistics engine.

Input:

``` js
digits = [0, 1, 4, 4, 7, 3, 4, 9, ...]
```

Functions:

``` js
countDigit(digit)
frequencyDigit(digit)
percentageDigit(digit)
lastDigit()
lastNDigits(n)
oddPercentage()
evenPercentage()
overPercentage(barrier)
underPercentage(barrier)
```

Example:

``` js
digitFrequency(3, 100)
```

returns:

``` js
{
    digit: 3,
    count: 11,
    total: 100,
    percentage: 11
}
```

Do not call this a prediction.

It is a historical statistic.

------------------------------------------------------------------------

# 38. Strategy Safety

The platform must clearly distinguish:

``` text
Observed data
Historical statistic
Signal
Prediction
Execution
```

For example:

``` text
Digit 3 appeared 14% in last 100 ticks.
```

must not be displayed as:

``` text
Digit 3 will appear next.
```

No statistical pattern should be represented as guaranteed.

------------------------------------------------------------------------

# 39. AI Extension Point

Create an optional strategy adapter:

``` js
strategyEngine.register("custom-ai", {
    async predict(context) {
        return {
            signal: "...",
            confidence: null,
            metadata: {}
        };
    }
});
```

Do not make an AI model mandatory for the base app.

The base platform must work without external AI.

If an AI service is later added, remember that a browser-only app
exposes any API key sent to the browser. Use an appropriate secure
architecture for production AI API credentials.

------------------------------------------------------------------------

# 40. Risk Manager

Every live trade must pass:

``` js
riskManager.canTrade(context)
```

Rules:

``` text
bot enabled
authenticated
correct account
market available
contract available
stake <= max stake
daily loss < daily loss limit
consecutive losses < max
cooldown elapsed
no active trade
proposal valid
```

Example:

``` js
const riskConfig = {
    maxStake: 2,
    maxDailyLoss: 20,
    maxConsecutiveLosses: 3,
    cooldownMs: 5000,
    maxTradesPerSession: 100
};
```

Make these configurable.

------------------------------------------------------------------------

# 41. Never Implement Martingale by Default

Do not create automatic stake doubling as a default strategy.

If a progressive staking block is eventually provided, it must have
explicit limits:

``` text
maximum stake
maximum steps
maximum session loss
reset conditions
```

The UI must clearly identify it as high-risk behavior.

------------------------------------------------------------------------

# 42. Simulation Mode

This is mandatory before live trading.

Modes:

``` text
SIMULATION
DEMO
REAL
```

Simulation:

``` text
market data = live
trade execution = fake
proposal = optionally fetched
buy = NOT sent
```

Display:

``` text
SIMULATION — NO REAL ORDERS
```

Demo:

``` text
DEMO ACCOUNT
```

Real:

``` text
REAL ACCOUNT
```

Never hide the mode.

------------------------------------------------------------------------

# 43. Live Trading Confirmation

Before enabling real trading:

``` text
REAL ACCOUNT

This bot is allowed to send real trading orders.

Account: CRxxxx
Currency: USD
Stake: 1.00
Max daily loss: 20.00

[ Cancel ]
[ Enable Real Trading ]
```

Require an explicit user action.

------------------------------------------------------------------------

# 44. Stop Button

The stop button must be globally visible.

``` text
[ STOP BOT ]
```

Stopping should:

1.  set runtime state to STOPPING
2.  prevent new signals
3.  prevent new proposals
4.  prevent new buys
5.  optionally cancel pending proposal streams
6.  stop generated loop
7.  keep existing open contract monitoring if configured
8.  unsubscribe market streams if no longer needed
9.  set state to STOPPED

Do not close the entire WebSocket automatically if other application
panels still need it.

------------------------------------------------------------------------

# 45. Emergency Stop

Add:

``` text
[ EMERGENCY STOP ]
```

It must:

-   stop bot
-   block new trades
-   release trade locks
-   cancel bot timers
-   unsubscribe bot-owned subscriptions
-   leave account data readable where possible

If the platform supports an API-level cancellation action, only use it
when appropriate to the contract/API semantics.

------------------------------------------------------------------------

# 46. Generated JavaScript Validation

Before executing generated code:

``` text
workspace
 ↓
generate JavaScript
 ↓
syntax validation
 ↓
Blockly structure validation
 ↓
security validation
 ↓
runtime API validation
 ↓
execution
```

Do not simply do:

``` js
eval(code);
```

with unrestricted generated code.

Blockly's documentation notes that execution of generated JavaScript is
application-specific and points developers toward safe execution
mechanisms such as JSInterpreter where applicable.

For this trading platform, the preferred architecture is to generate
calls against a constrained bot runtime rather than execute arbitrary
JavaScript with full browser privileges.

------------------------------------------------------------------------

# 47. Bot Runtime API

Expose only approved methods:

``` js
bot.selectMarket()
bot.currentTick()
bot.waitForNextTick()
bot.lastDigit()
bot.getHistory()
bot.countDigit()
bot.getProposal()
bot.placeTrade()
bot.getOpenContract()
bot.sellContract()
bot.sleep()
bot.log()
bot.isRunning()
bot.stop()
bot.getBalance()
```

Do not expose:

``` js
window
document
localStorage
fetch
WebSocket
crypto
location
```

to generated bot code unless explicitly required and safely wrapped.

------------------------------------------------------------------------

# 48. Execution Context

Generated code should receive:

``` js
const botRuntime = createBotRuntime({
    deriv,
    marketState,
    accountState,
    tradeManager,
    riskManager,
    logger
});
```

Then:

``` js
await generatedBot(botRuntime);
```

The generated program cannot directly access internal objects.

------------------------------------------------------------------------

# 49. Bot Validator

Validate before start.

Checks:

``` text
✓ Main bot exists
✓ Exactly one main bot
✓ Market selected
✓ Contract selected
✓ Stake valid
✓ Duration valid
✓ Contract supported
✓ No impossible block combination
✓ No trade block outside executable context
✓ No infinite unsafe loop
✓ Risk settings present
✓ Live mode confirmation
```

Errors should point to blocks.

Example:

``` text
Cannot start bot:
Digit Over block is missing a barrier.
```

------------------------------------------------------------------------

# 50. Blockly Serialization

Support:

``` text
Save Project
Load Project
Export XML
Import XML
Export JSON metadata
```

Blockly XML should be the primary workspace serialization.

Store:

``` js
{
    version: 1,
    appVersion: "...",
    savedAt: "...",
    projectName: "...",
    workspaceXml: "...",
    settings: {...}
}
```

------------------------------------------------------------------------

# 51. Local Storage

Use localStorage only for non-sensitive preferences:

``` text
theme
selected UI layout
last project metadata
simulation preference
chart settings
```

For larger projects use IndexedDB.

Never store long-lived trading credentials in localStorage by default.

------------------------------------------------------------------------

# 52. IndexedDB

Database:

``` text
DerivBlocklyDB
```

Stores:

``` text
projects
tradeHistory
settings
```

Project:

``` js
{
    id,
    name,
    version,
    createdAt,
    updatedAt,
    workspaceXml,
    config
}
```

Trade record:

``` js
{
    id,
    timestamp,
    accountMode,
    symbol,
    contractType,
    stake,
    proposalId,
    contractId,
    entrySpot,
    exitSpot,
    profit,
    status
}
```

------------------------------------------------------------------------

# 53. Trade History

Create a table:

``` text
Time
Symbol
Contract
Stake
Entry
Exit
Result
Profit/Loss
Contract ID
```

Filters:

``` text
Today
Session
Symbol
Contract type
Win
Loss
```

Do not claim historical data is complete unless the application actually
loaded it.

------------------------------------------------------------------------

# 54. Dashboard Layout

Recommended layout:

``` text
┌────────────────────────────────────────────────────────────┐
│ HEADER                                                     │
│ Logo | Connection | Account | Balance | DEMO/REAL | STOP │
├──────────────┬─────────────────────────────┬───────────────┤
│ TOOLBOX      │ BLOCKLY WORKSPACE           │ INSPECTOR     │
│              │                             │               │
│ Bot          │                             │ Market        │
│ Market       │                             │ Contract      │
│ Logic        │                             │ Strategy      │
│ Digits       │                             │ Risk          │
│ Trading      │                             │               │
├──────────────┴─────────────────────────────┴───────────────┤
│ CHART / TICKS                                               │
├─────────────────────────────────────────────────────────────┤
│ GENERATED CODE / VALIDATION                                 │
├─────────────────────────────────────────────────────────────┤
│ TRADE HISTORY / LOGS                                        │
└─────────────────────────────────────────────────────────────┘
```

------------------------------------------------------------------------

# 55. Header

Show:

``` text
Deriv Blockly Bot
Connection: ● Connected
Account: CRxxxx
Mode: DEMO
Currency: USD
Balance: 10000.00

[Run]
[Pause]
[Stop]
```

When real:

``` text
⚠ REAL ACCOUNT
```

------------------------------------------------------------------------

# 56. Chart

The chart should display:

-   tick price
-   selected interval
-   latest price
-   optional candles
-   trade entry markers
-   trade exit markers
-   current contract state

Do not make chart rendering part of the trading engine.

Use a separate chart adapter.

------------------------------------------------------------------------

# 57. Tick Panel

Show:

``` text
Symbol: R_100
Latest: 1234.56
Last digit: 6
Tick time: 13:45:32
Ticks stored: 500
```

For digit strategies:

``` text
Last 100 digits
0: 11%
1: 8%
2: 13%
...
9: 9%
```

Label this as observed historical frequency, not prediction.

------------------------------------------------------------------------

# 58. Logs

Levels:

``` text
DEBUG
INFO
WARN
ERROR
TRADE
API
```

Example:

``` text
[13:45:01] INFO Connected to Deriv
[13:45:02] INFO Authenticated
[13:45:03] INFO Loaded 245 symbols
[13:45:05] INFO Tick subscription active: R_100
[13:45:15] TRADE Proposal requested
[13:45:16] TRADE Proposal received
[13:45:17] TRADE Contract purchased
```

Never log access tokens.

------------------------------------------------------------------------

# 59. Error Handling

Normalize Deriv errors:

``` js
{
    code,
    message,
    details,
    requestId,
    timestamp,
    endpoint
}
```

Display human-readable messages.

Example:

``` text
API ERROR

Code: InvalidContract
Message: The requested contract is not available for this market.

Action:
Select another contract or refresh market data.
```

Do not show raw secrets or sensitive request headers.

------------------------------------------------------------------------

# 60. Timeouts

Every request needs a timeout.

Example:

``` js
const DEFAULT_TIMEOUT = 15000;
```

For critical trading requests, choose appropriate values.

Never leave Promises pending forever.

------------------------------------------------------------------------

# 61. API Rate Awareness

Do not repeatedly call:

``` text
active_symbols
contracts_for
proposal
```

on every tick.

Cache:

``` text
active_symbols
contracts_for
```

with refresh policies.

Proposal requests should be event-driven.

Tick data should use subscriptions.

------------------------------------------------------------------------

# 62. Market Change Workflow

When user changes market:

``` text
pause bot
↓
unsubscribe old ticks
↓
clear old market cache
↓
request contracts_for(newSymbol)
↓
validate contract
↓
seed history
↓
subscribe new ticks
↓
resume only if user confirms/resume policy allows
```

Never let old market ticks trigger a new-market trade.

------------------------------------------------------------------------

# 63. Bot Start Workflow

Exact sequence:

``` text
User presses RUN
        ↓
validate workspace
        ↓
validate authentication
        ↓
validate account mode
        ↓
validate risk settings
        ↓
validate market
        ↓
validate contract
        ↓
initialize bot state
        ↓
ensure required subscriptions
        ↓
generate code
        ↓
validate code
        ↓
start runtime
        ↓
RUNNING
```

------------------------------------------------------------------------

# 64. Trade Workflow

Exact sequence:

``` text
tick received
↓
update market state
↓
strategy evaluates
↓
signal generated
↓
risk manager checks
↓
trade lock acquired
↓
proposal requested
↓
proposal validated
↓
buy request
↓
buy response
↓
contract ID stored
↓
contract subscription started
↓
OPEN
↓
monitor
↓
settled
↓
record result
↓
release trade lock
↓
strategy continues
```

------------------------------------------------------------------------

# 65. Error During Buy

If buy fails:

``` text
trade lock
↓
capture error
↓
classify error
↓
release lock
↓
record failed attempt
↓
apply retry policy
```

Do not blindly repeat a buy.

A retry must verify that the previous request did not actually succeed.

------------------------------------------------------------------------

# 66. Error During Contract Monitoring

If monitoring disconnects:

``` text
DO NOT assume contract is lost.
```

After reconnect:

1.  re-authenticate if necessary
2.  query/open contract state
3.  restore subscription
4.  reconcile with local trade state
5.  determine actual contract status
6.  only then continue

------------------------------------------------------------------------

# 67. State Reconciliation

Local state is not authoritative.

Deriv's account/contract response is authoritative.

After reconnect:

``` text
local state
vs
Deriv state
```

Resolve discrepancies.

Example:

``` text
Local:
contract = OPEN

Deriv:
contract = SOLD

→ update local state to SOLD
```

------------------------------------------------------------------------

# 68. Bot Variables

Provide Blockly variables:

``` text
stake
barrier
duration
lastDigit
lastPrice
previousPrice
tradeCount
winCount
lossCount
profit
consecutiveLosses
```

Generated code should use runtime-scoped variables rather than globals.

------------------------------------------------------------------------

# 69. Conditions

Support:

``` text
price > value
price < value
price == value
lastDigit > value
lastDigit < value
digit == value
frequency > value
winCount >= value
lossCount >= value
profit > value
```

------------------------------------------------------------------------

# 70. Loops

Support:

``` text
repeat N times
repeat while
wait until tick
for each historical tick
```

Avoid allowing an unrestricted:

``` text
while(true)
```

unless the bot runtime controls it and checks cancellation on every
iteration.

------------------------------------------------------------------------

# 71. Cancellation

Every long-running generated operation must be cancellable.

Use:

``` js
AbortController
```

or a bot runtime cancellation token.

Conceptually:

``` js
while (bot.isRunning()) {
    ...
}
```

and:

``` js
bot.stop()
```

must cause pending loops to terminate.

------------------------------------------------------------------------

# 72. No Fixed Delays for API Sequencing

Do not use:

``` js
await sleep(3000);
```

to wait for Deriv.

Wrong:

``` js
sendAuthorize();
await sleep(3000);
sendBalance();
```

Correct:

``` js
await authorize();
await balance();
```

Use Promise-based request correlation.

Delays may be used only for intentional strategy cooldowns, UI effects,
or reconnection backoff.

------------------------------------------------------------------------

# 73. Authentication Sequence

Use the currently supported authentication workflow.

For a legacy-compatible v3 session, the conceptual sequence may look
like:

``` text
WebSocket connect
↓
authorize
↓
account information
↓
balance
↓
market data/trading
```

However, current Deriv documentation also describes a newer flow for
Options trading:

``` text
PAT/OAuth authorization
↓
REST OTP request
↓
authenticated Options WebSocket URL
↓
WebSocket connection
↓
trading
```

The implementation must determine which flow belongs to the registered
application and selected Deriv API.

Do not combine:

``` text
new OAuth/OTP flow
```

with:

``` text
old authorize-token v3 flow
```

without verifying compatibility.

------------------------------------------------------------------------

# 74. Account Selection

If the authenticated user has multiple supported accounts:

``` text
Select account

○ Demo
○ Real
```

After account selection:

``` text
load account metadata
load currency
load balance
establish correct trading connection
```

Never assume USD.

Currency must come from the account.

------------------------------------------------------------------------

# 75. Currency

Proposal requests must use the account-holder's currency.

Do not hard-code:

``` js
currency: "USD"
```

Use:

``` js
currency: accountState.currency
```

unless the current API explicitly requires another format.

------------------------------------------------------------------------

# 76. Contract Duration

Support:

``` text
ticks
seconds
minutes
hours
days
```

But only display units supported by the selected contract.

For digit contracts, ticks are especially important.

Example:

``` text
Duration: 1
Unit: tick
```

------------------------------------------------------------------------

# 77. Contract Validation

Before proposal:

``` js
validateContract({
    symbol,
    contractType,
    duration,
    durationUnit,
    barrier,
    stake
});
```

Validate:

``` text
symbol exists
contract exists
duration > 0
stake > 0
barrier in valid range
currency exists
```

For digit barrier:

``` text
0–9
```

------------------------------------------------------------------------

# 78. Proposal vs Buy Price

Do not confuse:

``` text
stake
payout
proposal ask price
buy price
```

Use the current API response fields.

Do not calculate payout manually when the API supplies the proposal.

------------------------------------------------------------------------

# 79. Trade Result

After settlement:

``` js
{
    result: "won" | "lost" | "sold" | "expired" | "unknown",
    profit,
    payout,
    stake,
    contractId,
    timestamp
}
```

Increment:

``` text
tradeCount
winCount
lossCount
profit
consecutiveLosses
```

Reset consecutive losses after a confirmed win if that is the selected
risk policy.

------------------------------------------------------------------------

# 80. Daily/session Risk

Track:

``` js
sessionStats = {
    trades: 0,
    wins: 0,
    losses: 0,
    grossProfit: 0,
    maxDrawdown: 0,
    consecutiveLosses: 0
};
```

Daily limit should use a clearly defined timezone.

Do not assume browser timezone silently.

------------------------------------------------------------------------

# 81. UI Notifications

Use:

``` text
success
info
warning
error
trade
```

Example:

``` text
✓ Bot started
⚠ Risk limit reached
✕ Proposal failed
✓ Contract settled: profit 0.95
```

Notifications must not block the trading engine.

------------------------------------------------------------------------

# 82. Accessibility

Implement:

-   keyboard navigation
-   visible focus
-   ARIA labels
-   readable contrast
-   accessible buttons
-   screen-reader labels
-   no color-only status indicators

Example:

``` text
● Connected
```

must also have:

``` html
aria-label="Connected"
```

------------------------------------------------------------------------

# 83. Responsive Design

Desktop:

``` text
3-column workspace
```

Tablet:

``` text
2-column
```

Mobile:

``` text
tabs:
Blockly
Chart
Trade
Logs
```

Do not make mobile a scaled-down desktop layout.

------------------------------------------------------------------------

# 84. Project Versioning

Store:

``` js
const PROJECT_SCHEMA_VERSION = 1;
```

When loading:

``` text
if old version:
    migrate()
```

Example future migration:

``` text
v1
→ v2
```

Never break old user projects silently.

------------------------------------------------------------------------

# 85. Import Validation

Never blindly load imported XML.

Check:

``` text
schema
block types
field values
workspace size
unexpected extensions
```

Reject malformed project files.

------------------------------------------------------------------------

# 86. Default Demo Project

Ship a default project:

``` text
START BOT

Market: dynamically selected demo market

WAIT FOR TICK

SET:
digit = last digit

IF digit > 3
    LOG "Digit is above 3"
```

This should not automatically trade.

------------------------------------------------------------------------

# 87. Example Digit Strategy Project

Create:

``` text
START BOT
    |
    +-- Select Market
    |
    +-- Set Stake
    |
    +-- Wait For Tick
    |
    +-- Get Last Digit
    |
    +-- IF
            last digit < 7
            THEN
                Digit Under 7
```

For real execution, the trade block must still pass risk validation and
proposal/buy checks.

------------------------------------------------------------------------

# 88. Example Generated Runtime

Conceptual generated code:

``` js
async function generatedBot(bot) {

    await bot.selectMarket("SELECTED_MARKET");

    while (bot.isRunning()) {

        const tick = await bot.waitForNextTick();

        const digit = bot.lastDigit(tick);

        if (digit < 7) {

            await bot.placeTrade({
                contractType: "DIGITUNDER",
                barrier: 7,
                stake: 1,
                duration: 1,
                durationUnit: "t"
            });

            await bot.waitForContractSettlement();
        }
    }
}
```

The generated code should not contain API authentication tokens.

------------------------------------------------------------------------

# 89. Blockly Block Naming Convention

Use:

``` text
db_start
db_select_market
db_current_tick
db_last_digit
db_digit_over
db_digit_under
db_get_proposal
db_buy
db_sell
db_contract_status
db_wait_tick
db_wait_settlement
db_risk_check
db_log
```

Prefix all custom blocks with:

``` text
db_
```

to prevent collisions.

------------------------------------------------------------------------

# 90. Generator Naming Convention

Example:

``` js
javascriptGenerator.forBlock["db_last_digit"] =
    function(block, generator) {

        return [
            "await bot.lastDigit(await bot.currentTick())",
            javascriptGenerator.ORDER_AWAIT
        ];
    };
```

Follow the currently installed Blockly generator API rather than
assuming an older API.

------------------------------------------------------------------------

# 91. Toolbox Dynamic Configuration

The toolbox should be generated from application state where necessary.

Example:

``` js
function buildMarketBlocks(markets) {
    // create dynamic UI
}
```

But block definitions themselves should remain stable.

Do not create hundreds of block types dynamically for every symbol.

Use a dropdown field populated from current market data.

------------------------------------------------------------------------

# 92. Workspace Event Handling

Listen for:

``` text
BLOCK_CREATE
BLOCK_DELETE
BLOCK_CHANGE
BLOCK_MOVE
```

On change:

``` text
validate
generate code
update code panel
update error markers
save draft debounce
```

Do not regenerate excessively during drag operations.

Blockly documentation explicitly supports workspace event listeners for
continuous code generation.

------------------------------------------------------------------------

# 93. Autosave

Use debounce:

``` text
user changes workspace
↓
wait 500–1000ms
↓
save draft
```

Do not write to storage on every mouse movement.

------------------------------------------------------------------------

# 94. Code Panel

Show generated code in read-only mode by default.

Controls:

``` text
[Generated JS]
[Copy]
[Download]
[Validate]
```

Do not let editing generated code alter the Blockly workspace unless a
deliberate reverse-engineering system is implemented.

------------------------------------------------------------------------

# 95. Validation Panel

Example:

``` text
BOT VALIDATION

✓ Main bot found
✓ Market selected
✓ Contract selected
✓ Stake valid
✓ Duration valid
✓ Risk limits configured

READY TO RUN
```

Or:

``` text
BLOCKED

✕ Real account selected but live trading is not enabled.
✕ Digit Under block has no barrier.
```

------------------------------------------------------------------------

# 96. App Configuration

Create:

``` js
export const APP_CONFIG = {
    appName: "Deriv Blockly Bot",
    appVersion: "1.0.0",
    environment: "production",
    defaultMode: "simulation",
    maxTickBuffer: 5000,
    requestTimeoutMs: 15000,
    reconnect: {
        enabled: true,
        initialDelayMs: 1000,
        maxDelayMs: 30000
    }
};
```

Never put secrets here.

------------------------------------------------------------------------

# 97. Environment Configuration

For static frontend deployment:

``` text
APP_ID
OAuth client identifier
redirect URI
```

may be public configuration depending on Deriv's authentication
requirements.

Never treat:

``` text
client_secret
PAT
private signing key
```

as frontend configuration.

------------------------------------------------------------------------

# 98. Deployment

The application should work on:

``` text
GitHub Pages
Cloudflare Pages
Netlify
Vercel static hosting
Oracle Cloud static web server
```

provided the selected authentication flow supports the deployment
architecture.

Use HTTPS in production.

------------------------------------------------------------------------

# 99. Local Development

Recommended:

``` text
npm install
npm run dev
```

or a simple static server.

Do not develop OAuth against:

``` text
file://
```

Use:

``` text
http://localhost:PORT
```

with the exact redirect URI configured in the Deriv developer
application where required.

------------------------------------------------------------------------

# 100. Dependencies

Prefer modern package imports.

Possible dependencies:

``` text
blockly
```

Optional:

``` text
chart.js
```

or another charting library.

Do not add unnecessary frameworks.

Initial implementation should remain:

``` text
HTML
CSS
JavaScript
Blockly
```

without React/Vue unless there is a strong implementation reason.

------------------------------------------------------------------------

# 101. File-by-File Responsibilities

## index.html

Contains:

``` text
header
dashboard
Blockly container
chart container
account panel
trade panel
logs
generated code panel
modals
```

No large application logic.

## app.js

Bootstraps:

``` text
config
UI
Blockly
Deriv connection
state
event handlers
```

## deriv-client.js

Public API:

``` js
connect()
disconnect()
request()
subscribe()
forget()
```

## deriv-auth.js

Handles:

``` text
OAuth
PAT development mode
session state
logout
```

## deriv-market.js

Handles:

``` text
active symbols
contracts_for
ticks
ticks_history
normalization
```

## deriv-trading.js

Handles:

``` text
proposal
buy
proposal_open_contract
sell
portfolio
```

## bot-engine.js

Handles:

``` text
start
stop
pause
resume
generated program
```

## risk-manager.js

Handles:

``` text
stake
limits
cooldown
trade count
loss limits
```

## blockly/workspace.js

Handles:

``` text
Blockly initialization
XML
events
code generation
```

------------------------------------------------------------------------

# 102. Global State

Do not create a giant mutable global object.

Use separate stores:

``` js
connectionState
accountState
marketState
botState
tradeState
riskState
uiState
projectState
```

Each should expose controlled update methods.

------------------------------------------------------------------------

# 103. Event Bus

Create:

``` js
eventBus.emit("tick", tick);
eventBus.emit("trade:opened", contract);
eventBus.emit("trade:closed", result);
eventBus.emit("connection:changed", state);
eventBus.emit("bot:stateChanged", state);
```

This prevents UI code from being tightly coupled to Deriv API code.

------------------------------------------------------------------------

# 104. Runtime Event Flow

``` text
Deriv WebSocket
      ↓
DerivConnection
      ↓
DerivMarket
      ↓
MarketState
      ↓
EventBus
      ↓
BotRuntime
      ↓
Strategy
      ↓
RiskManager
      ↓
TradeManager
      ↓
DerivTrading
      ↓
Deriv WebSocket
```

------------------------------------------------------------------------

# 105. Testing Strategy

Create test layers.

## Unit tests

Test:

``` text
last digit
frequency
statistics
risk limits
stake validation
contract validation
state transitions
request ID matching
reconnection backoff
```

## Integration tests

Test against Deriv demo account:

``` text
connect
authenticate
symbols
ticks
proposal
buy
contract monitoring
settlement
balance
```

## UI tests

Test:

``` text
Blockly creation
block deletion
dropdown updates
run/stop
market change
project save/load
```

------------------------------------------------------------------------

# 106. Simulation Tests

A deterministic tick fixture:

``` js
[
    { quote: 100.1, epoch: 1 },
    { quote: 100.2, epoch: 2 },
    { quote: 100.3, epoch: 3 }
]
```

should produce predictable last digits according to symbol precision.

Do not use random ticks for unit tests.

------------------------------------------------------------------------

# 107. Trading State Tests

Test:

``` text
IDLE → PROPOSAL_REQUESTING
PROPOSAL_REQUESTING → PROPOSAL_READY
PROPOSAL_READY → BUY_PENDING
BUY_PENDING → OPEN
OPEN → SETTLED
```

Also:

``` text
BUY_PENDING → ERROR
OPEN → CONNECTION_LOST → RECONCILING → OPEN
OPEN → RECONCILING → SETTLED
```

------------------------------------------------------------------------

# 108. Duplicate Request Tests

Verify:

``` text
two simultaneous buy signals
```

result in:

``` text
one accepted trade
one blocked signal
```

unless multi-position mode is explicitly implemented.

------------------------------------------------------------------------

# 109. Connection Failure Tests

Simulate:

``` text
WebSocket closes during tick stream
WebSocket closes during proposal
WebSocket closes immediately after buy
WebSocket closes during contract settlement
```

Verify state recovery.

------------------------------------------------------------------------

# 110. Security Tests

Verify:

``` text
no token in generated JS
no token in logs
no token in localStorage
no secret in Git
no secret in error messages
real trading requires explicit confirmation
```

------------------------------------------------------------------------

# 111. Performance

Limit:

``` text
tick history buffer
DOM log entries
chart points
trade history UI rows
Blockly event processing
```

For logs, use a ring buffer:

``` text
maximum 1000 visible logs
```

For ticks:

``` text
maximum 5000 in-memory points
```

Make configurable.

------------------------------------------------------------------------

# 112. Memory Management

On market change:

``` text
unsubscribe old stream
remove old listeners
clear old tick arrays
destroy old chart dataset
remove obsolete contract listeners
```

On bot stop:

``` text
cancel runtime timers
remove bot-specific listeners
stop strategy loop
```

------------------------------------------------------------------------

# 113. Browser Lifecycle

Handle:

``` js
window.addEventListener("beforeunload", ...)
```

But do not claim that browser unload guarantees a trade cancellation.

When leaving:

``` text
stop bot
unsubscribe where possible
clear sensitive in-memory credentials
```

For real trading, show a warning if an active contract exists.

------------------------------------------------------------------------

# 114. Clock Handling

Use:

``` js
Date.now()
```

for local timestamps and Deriv epoch timestamps for API market events.

Do not use local browser clock to invent market event times.

Display:

``` text
UTC
or
user-selected timezone
```

consistently.

------------------------------------------------------------------------

# 115. Logging API Requests

For debug mode, log:

``` text
request type
req_id
timestamp
```

Do not log:

``` text
authorization token
PAT
OAuth access token
client secret
```

Sanitize objects before logging.

------------------------------------------------------------------------

# 116. Error Retry Policy

Classify errors:

``` text
AUTH
RATE_LIMIT
NETWORK
INVALID_REQUEST
MARKET_UNAVAILABLE
CONTRACT_UNAVAILABLE
PROPOSAL
BUY
ACCOUNT
UNKNOWN
```

Retry only transient categories.

Never retry:

``` text
invalid contract
invalid stake
invalid barrier
authentication failure
```

without correcting the input/state.

------------------------------------------------------------------------

# 117. Request Queue

For normal data requests, concurrent requests are acceptable if
correlation is correct.

For sensitive trading actions:

``` text
proposal
buy
sell
```

use a trade operation queue/lock.

Do not globally serialize all WebSocket messages.

Ticks must remain real-time.

------------------------------------------------------------------------

# 118. Contract Ownership

Each contract watcher must have:

``` js
{
    contractId,
    botId,
    subscriptionId,
    startedAt,
    state
}
```

If multiple bots are eventually supported, each bot must own its
subscriptions.

For version 1, enforce:

``` text
one active bot
one active trade
```

This dramatically reduces race conditions.

------------------------------------------------------------------------

# 119. One Bot Policy

Version 1 should enforce:

``` text
one main_block
one bot runtime
one active strategy
one active trade
```

Design interfaces so multi-bot support can be added later.

------------------------------------------------------------------------

# 120. Main Blockly Execution Contract

The top-level generated function must be:

``` js
async function runBot(bot) {
    // generated code
}
```

The application calls:

``` js
await botEngine.start();
```

The engine owns the lifecycle.

Blockly must never own:

``` text
WebSocket lifecycle
authentication
application navigation
UI rendering
```

------------------------------------------------------------------------

# 121. Recommended Build Order

Hermes must implement in this exact order.

## Phase 1 --- Project skeleton

Create:

``` text
index.html
css/
src/
```

Implement basic layout.

## Phase 2 --- Blockly

Implement:

``` text
Blockly workspace
toolbox
main block
custom blocks
JavaScript generation
code preview
```

Do not connect trading yet.

## Phase 3 --- Deriv public data

Implement:

``` text
connection
active_symbols
contracts_for
ticks_history
ticks
```

Display live data.

## Phase 4 --- Authentication

Implement the current supported OAuth/browser authentication flow.

Add development-only PAT mode if required.

## Phase 5 --- Account

Implement:

``` text
account
balance
account mode
currency
```

## Phase 6 --- Proposal

Implement:

``` text
proposal
proposal subscription where useful
```

## Phase 7 --- Demo trading

Implement:

``` text
buy
proposal_open_contract
sell where supported
```

Only demo account.

## Phase 8 --- Bot runtime

Connect Blockly-generated code to the runtime.

## Phase 9 --- Risk

Implement:

``` text
stake limits
daily loss
consecutive loss
cooldown
trade count
```

## Phase 10 --- Simulation

Implement fake execution.

## Phase 11 --- Real trading protection

Implement:

``` text
real account confirmation
emergency stop
```

## Phase 12 --- Persistence

Implement:

``` text
IndexedDB
Blockly XML
project import/export
```

## Phase 13 --- Reconnection/recovery

Implement:

``` text
reconnect
resubscribe
state reconciliation
```

## Phase 14 --- Testing

Run all tests.

## Phase 15 --- Deployment

Deploy static frontend over HTTPS.

------------------------------------------------------------------------

# 122. Definition of Done

The app is not complete until all of the following work:

## Blockly

``` text
[ ] Blockly loads
[ ] Toolbox loads
[ ] Custom blocks work
[ ] Main block uniqueness enforced
[ ] JavaScript generated
[ ] Code validation works
[ ] Project save works
[ ] Project load works
[ ] XML export works
[ ] XML import works
```

## Deriv market

``` text
[ ] WebSocket connects
[ ] active_symbols works
[ ] contracts_for works
[ ] tick history works
[ ] live ticks work
[ ] subscriptions tracked
[ ] unsubscribe works
```

## Authentication

``` text
[ ] OAuth flow configured
[ ] callback handled
[ ] authentication state tracked
[ ] logout works
[ ] tokens not logged
[ ] no client secret in frontend
```

## Trading

``` text
[ ] proposal works
[ ] proposal validation works
[ ] buy works in demo
[ ] contract monitoring works
[ ] settlement detected
[ ] sell supported where applicable
[ ] trade history works
```

## Bot

``` text
[ ] start
[ ] stop
[ ] pause
[ ] resume
[ ] emergency stop
[ ] cancellation
[ ] duplicate trade prevention
```

## Risk

``` text
[ ] max stake
[ ] daily loss
[ ] consecutive loss
[ ] cooldown
[ ] maximum trades
```

## Reliability

``` text
[ ] request timeout
[ ] reconnect
[ ] subscription recovery
[ ] state reconciliation
[ ] API errors displayed
```

------------------------------------------------------------------------

# 123. Important Implementation Anti-Patterns

Never do this:

``` js
setTimeout(() => buy(), 3000);
```

to wait for an API response.

Never do this:

``` js
const token = "REAL_TOKEN";
```

in source.

Never do this:

``` js
localStorage.setItem("token", token);
```

for a persistent trading credential.

Never do this:

``` js
eval(userGeneratedCode);
```

without a carefully controlled execution model.

Never do this:

``` js
if (digit > 3) buy();
```

without proposal/risk/trade-lock validation.

Never do this:

``` js
new WebSocket(...)
```

inside every Blockly block.

Never do this:

``` js
currency: "USD"
```

unless the current account actually uses USD.

Never assume a contract is available just because its name exists.

Always use `contracts_for`/current API metadata.

Never automatically restart a trade after an uncertain buy response
without reconciling the account/contract state.

------------------------------------------------------------------------

# 124. Initial User Experience

On first load:

``` text
DERIV BLOCKLY BOT

Build trading bots visually.

[ Connect Deriv ]
[ Try Simulation ]
```

Simulation can immediately open the Blockly builder.

After connecting:

``` text
CONNECTED

Account: DEMO
Currency: USD
Balance: ...

[ Build Bot ]
```

------------------------------------------------------------------------

# 125. Simulation First

The first tutorial should be:

``` text
1. Open Simulation
2. Select market
3. Add Wait For Tick
4. Add Last Digit
5. Add comparison
6. Add Log
7. Run
8. Observe ticks
9. Observe generated JavaScript
```

Then:

``` text
10. Get proposal
11. Demo trade
```

Only after that:

``` text
12. Real trading
```

------------------------------------------------------------------------

# 126. Recommended Internal Interfaces

## Deriv client

``` js
deriv.connect()
deriv.disconnect()
deriv.request(payload)
deriv.subscribe(payload, callback)
deriv.forget(subscriptionId)
```

## Market service

``` js
market.loadSymbols()
market.loadContracts(symbol)
market.loadHistory(symbol, options)
market.subscribeTicks(symbol)
market.unsubscribeTicks(symbol)
market.getLatestTick()
```

## Trading service

``` js
trading.proposal(params)
trading.buy(proposalId, price)
trading.watchContract(contractId)
trading.sell(contractId, price)
```

## Risk service

``` js
risk.canTrade(context)
risk.recordTrade(result)
risk.getState()
risk.resetSession()
```

## Bot runtime

``` js
bot.selectMarket(symbol)
bot.waitForNextTick()
bot.currentTick()
bot.lastDigit(tick)
bot.placeTrade(params)
bot.waitForContractSettlement()
bot.stop()
bot.isRunning()
```

------------------------------------------------------------------------

# 127. Example `DerivConnection` Skeleton

Use this as architectural guidance, not as a substitute for validating
the current API:

``` js
export class DerivConnection {

    constructor({ url }) {
        this.url = url;
        this.ws = null;
        this.reqId = 0;
        this.pending = new Map();
        this.subscriptions = new Map();
    }

    nextReqId() {
        this.reqId += 1;
        return this.reqId;
    }

    connect() {
        return new Promise((resolve, reject) => {

            this.ws = new WebSocket(this.url);

            this.ws.onopen = () => resolve();

            this.ws.onerror = (error) => reject(error);

            this.ws.onmessage = (event) => {
                this.handleMessage(event);
            };

            this.ws.onclose = () => {
                this.rejectPending(new Error("WebSocket closed"));
            };
        });
    }

    request(payload, timeoutMs = 15000) {

        const req_id = this.nextReqId();

        return new Promise((resolve, reject) => {

            const timer = setTimeout(() => {
                this.pending.delete(req_id);
                reject(new Error(`Request timeout: ${req_id}`));
            }, timeoutMs);

            this.pending.set(req_id, {
                resolve: (data) => {
                    clearTimeout(timer);
                    resolve(data);
                },
                reject: (error) => {
                    clearTimeout(timer);
                    reject(error);
                }
            });

            this.ws.send(JSON.stringify({
                ...payload,
                req_id
            }));
        });
    }

    handleMessage(event) {

        const data = JSON.parse(event.data);

        if (data.error) {
            const pending = this.pending.get(data.req_id);

            if (pending) {
                pending.reject(data.error);
                this.pending.delete(data.req_id);
            }

            return;
        }

        if (data.req_id) {
            const pending = this.pending.get(data.req_id);

            if (pending) {
                pending.resolve(data);
                this.pending.delete(data.req_id);
            }
        }

        // Route subscription messages separately.
    }

    rejectPending(error) {

        for (const pending of this.pending.values()) {
            pending.reject(error);
        }

        this.pending.clear();
    }

    disconnect() {

        if (this.ws) {
            this.ws.close();
        }
    }
}
```

The production version must add subscription routing, reconnect logic,
authentication/session recovery, cleanup, state transitions and
API-specific behavior.

------------------------------------------------------------------------

# 128. Example Blockly Generator

``` js
javascriptGenerator.forBlock["db_last_digit"] =
    function(block, generator) {

        return [
            "await bot.lastDigit(await bot.currentTick())",
            javascriptGenerator.ORDER_ATOMIC
        ];
    };
```

For a trade block:

``` js
javascriptGenerator.forBlock["db_digit_under"] =
    function(block, generator) {

        const barrier = block.getFieldValue("BARRIER");
        const stake = block.getFieldValue("STAKE");
        const duration = block.getFieldValue("DURATION");

        return `
await bot.placeTrade({
    contractType: "DIGITUNDER",
    barrier: ${Number(barrier)},
    stake: ${Number(stake)},
    duration: ${Number(duration)},
    durationUnit: "t"
});
`;
    };
```

The runtime, not the block, validates risk and API availability.

------------------------------------------------------------------------

# 129. Example Runtime Trade Method

``` js
async placeTrade(params) {

    if (!this.isRunning()) {
        throw new Error("Bot is not running");
    }

    if (this.tradeManager.tradeLock) {
        throw new Error("Another trade is already in progress");
    }

    await this.riskManager.assertAllowed(params);

    this.tradeManager.tradeLock = true;

    try {

        const proposal = await this.trading.proposal({
            amount: params.stake,
            basis: "stake",
            contract_type: params.contractType,
            currency: this.account.currency,
            duration: params.duration,
            duration_unit: params.durationUnit,
            barrier: String(params.barrier),
            underlying_symbol: this.market.symbol
        });

        this.riskManager.validateProposal(proposal);

        const bought = await this.trading.buy(
            proposal.proposal.id,
            proposal.proposal.ask_price
        );

        const contractId = bought.buy.contract_id;

        return await this.trading.watchContract(contractId);

    } finally {

        this.tradeManager.tradeLock = false;
    }
}
```

Adapt field names to the exact current API response schema.

------------------------------------------------------------------------

# 130. Required API Documentation References

Hermes should consult the official documentation before implementing or
updating each integration:

-   Deriv API documentation: https://developers.deriv.com/docs/
-   Deriv authentication:
    https://developers.deriv.com/docs/intro/authentication/
-   Deriv market data: https://developers.deriv.com/docs/data/
-   Active symbols:
    https://developers.deriv.com/docs/data/active-symbols/
-   Contracts for symbol:
    https://developers.deriv.com/docs/data/contracts-for/
-   Proposal: https://developers.deriv.com/docs/trading/proposal/
-   Buy: https://developers.deriv.com/docs/trading/buy/
-   Open contract:
    https://developers.deriv.com/docs/trading/proposal-open-contract/
-   Trading operations: https://developers.deriv.com/docs/trading/
-   Complete workflows: https://developers.deriv.com/docs/workflows/
-   Blockly code generation:
    https://docs.blockly.com/guides/get-started/code-generation/
-   Blockly generate/run:
    https://docs.blockly.com/guides/app-integration/run-code/

These URLs are reference destinations; the agent must verify their
current content at implementation time.

------------------------------------------------------------------------

# 131. Final Hermes Instruction

Build the application incrementally.

Do NOT generate a fake finished trading application with placeholder
buttons.

Every button that claims to work must call a real implementation.

Do not fabricate Deriv response fields.

Do not hard-code market lists when the API can provide them.

Do not hard-code account currency.

Do not use arbitrary delays for API synchronization.

Do not use old API examples without checking the current official
documentation.

Do not expose secrets.

Do not implement real trading until simulation and demo trading are
working.

Do not automatically retry uncertain trading operations.

Do not allow duplicate buys.

Do not let Blockly generated code directly control WebSockets.

Do not use unrestricted `eval()` as the bot execution architecture.

Use a controlled runtime abstraction.

------------------------------------------------------------------------

# 132. Final Build Milestone Structure

Hermes should report these milestones separately:

``` text
MILESTONE 01
Project skeleton + responsive UI

MILESTONE 02
Blockly workspace + custom blocks

MILESTONE 03
JavaScript generator + validator

MILESTONE 04
Deriv public WebSocket market data

MILESTONE 05
Authentication

MILESTONE 06
Account + balance

MILESTONE 07
Proposal engine

MILESTONE 08
Demo trading

MILESTONE 09
Contract monitoring

MILESTONE 10
Bot runtime

MILESTONE 11
Risk manager

MILESTONE 12
Simulation engine

MILESTONE 13
Project persistence

MILESTONE 14
Reconnect/reconciliation

MILESTONE 15
Testing

MILESTONE 16
Production hardening

MILESTONE 17
HTTPS static deployment
```

At the end of each milestone:

``` text
- list files changed
- list implemented functions
- list API endpoints used
- list tests performed
- list known limitations
- do not claim functionality that has not been tested
```

------------------------------------------------------------------------

# 133. Non-Negotiable Architecture Summary

The final application should follow:

``` text
                         ┌──────────────────────┐
                         │      HTML/CSS UI     │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │   Blockly Workspace  │
                         └──────────┬───────────┘
                                    │
                         workspaceToCode()
                                    │
                         ┌──────────▼───────────┐
                         │ Generated JS Program │
                         └──────────┬───────────┘
                                    │
                         controlled runtime API
                                    │
               ┌────────────────────▼───────────────────┐
               │              Bot Engine                │
               ├────────────────────────────────────────┤
               │ Strategy │ Risk │ Trade │ State │ Stop │
               └────────────────────┬───────────────────┘
                                    │
                         ┌──────────▼───────────┐
                         │    Deriv Services    │
                         ├──────────────────────┤
                         │ Auth                 │
                         │ Market               │
                         │ Proposal             │
                         │ Buy                  │
                         │ Contract Monitor     │
                         │ Sell                 │
                         │ Balance              │
                         └──────────┬───────────┘
                                    │
                              WebSocket/API
                                    │
                         ┌──────────▼───────────┐
                         │       Deriv          │
                         └──────────────────────┘
```

This separation is the central design rule.

------------------------------------------------------------------------

# 134. Success Criteria

A successful build is a real browser application where:

``` text
User
 ↓
connects Deriv
 ↓
sees real market data
 ↓
opens Blockly
 ↓
builds a strategy
 ↓
sees generated JavaScript
 ↓
validates it
 ↓
runs simulation
 ↓
runs demo trading
 ↓
sees proposal
 ↓
sees contract
 ↓
sees settlement
 ↓
sees balance/trade history
 ↓
can safely stop the bot
```

Only after this entire path is reliable should real-account trading be
enabled.
