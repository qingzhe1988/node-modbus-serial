# modbus-serial — Agent Guide

Pure JS MODBUS-RTU (Serial + TCP) for Node.js. v8.0.x.

## Project structure

```
index.js               ← Entry point; ModbusRTU class + FC handlers
ModbusRTU.d.ts          ← Main type declarations
index.d.ts              ← Re-exports ModbusRTU, ServerTCP, ServerSerial
apis/                   ← Mixin modules that augment ModbusRTU.prototype
  connection.js          ← connectRTU / connectTCP / connectUDP / … shorthands
  promise.js             ← Callback → Promise wrapper (dual API)
  worker.js              ← Worker (polling, concurrent requests) integration
ports/                   ← Transport implementations (all extend EventEmitter)
  tcpport.js             ← TCP (net.Socket)
  udpport.js             ← UDP (dgram)
  rtubufferedport.js     ← RTU over buffered serial
  asciiport.js           ← ASCII serial
  bleport.js             ← Bluetooth LE (Web Bluetooth API)
  c701port.js            ← C701 UDP-to-serial bridge
  tcprtubufferedport.js  ← TCP + RTU framing
  telnetport.js          ← Telnet
  testport.js            ← In-memory test port
servers/                 ← Server implementations
  servertcp.js           ← Modbus TCP server
  serverserial.js        ← Modbus Serial server
utils/                   ← crc16.js, lrc.js, buffer_bit.js
worker/                  ← Polling / concurrent request worker class
scripts/                 ← build, clean, copy-doc-examples (maintainer tasks)
test/                    ← Mocha tests
  mocks/                 ← netMock, dgramMock, SerialPortMock (mockery)
```

## Essential commands

| Command | Purpose |
|---|---|
| `npm ci` | Install (committed lockfile; do NOT `npm install` after clone) |
| `npm test` | `nyc --reporter=lcov --reporter=text mocha --recursive` |
| `npm run lint` | ESLint flat config (eslint.config.mjs) |
| `npm run lint:fix` | Auto-fix lint |
| `npm run clean` | Remove `modbus-serial/` and `docs/gen/` (gitignored) |
| `npm run build` | Copy `apis/ ports/ servers/ utils/` into `modbus-serial/` |
| `npm run doc` | JSDoc 4 + clean-jsdoc-theme → `docs/gen/` |
| `npm run publish:prepare` | `build` + `docs` — mirrors published tree |

**Important**: After editing `package.json`, run `npm install` (not `npm ci`) and commit lockfile.

## Code conventions

- **CommonJS** everywhere (no ESM except `eslint.config.mjs`)
- `"use strict"` in every `.js` file
- **Callback-first API** with optional **Promise** return (dual pattern, see `apis/promise.js`)
  - If last arg is a function → callback mode
  - Otherwise → returns a Promise
- **No classes** in `index.js` — uses constructor functions and prototype assignment
- **Port classes** extend `EventEmitter` and must implement: `open(cb)`, `close(cb)`, `write(data)`, emit `"data"` on receive
- Error objects: `PortNotOpenError` (errno `ECONNREFUSED`), `BadAddressError` (errno `ECONNREFUSED`), `TransactionTimedOutError` (errno `ETIMEDOUT`)
- Debug via `require("debug")("modbus-serial")`
- No TypeScript; `.d.ts` files are hand-written

## serialport is optional

- `serialport` (13.x, requires Node 20+) is an **optional** dependency
- `npm install modbus-serial --no-optional` works; TCP/UDP/BLE/Telnet usage works without it
- Serial RTU/ASCII APIs fail at **runtime** (not install time) if missing:
  - `getPorts()`, `connectRTU`, `connectRTUBuffered`, `connectAsciiSerial`
  - `RTUBufferedPort`, `ServerSerial` (not exported if serialport missing)
- For TypeScript: `ServerSerial.d.ts` imports `serialport` types. With `--no-optional`, use `skipLibCheck` or add `serialport` as devDependency.
- To add serial later: `npm install serialport`

## Testing

- **Mocha** + **Chai** + **Sinon** + **Mockery** + **NYC** (code coverage)
- Tests mock native modules (`net`, `dgram`, `serialport`) via `mockery`
  - Enable mockery in `before()` via `test/mocks/{netMock,dgramMock,SerialPortMock}.js`
  - Mock objects expose `receive(buffer)` to simulate incoming data
  - Always call `mockery.disable()` in `after()`
- `TestPort` — in-memory port used by `test/test.js` for ModbusRTU integration tests
- Coverage output to `coverage/` and `.nyc_output/` (gitignored)

## CI (GitHub Actions)

- Runs on `main` pushes and all PRs
- Matrix: Node 22.x + 24.x on ubuntu-latest
- Steps: `npm ci` → `npm run lint` → `npm test`

## Key gotchas

- `readCoils` / `readDiscreteInputs` return booleans; `readHoldingRegisters` / `readInputRegisters` return numbers. All response objects have `.data` (parsed array) and `.buffer` (raw Buffer).
- `writeFC*` methods (low-level) take `(address, ...args, next)` where `address` is the **unit ID**. High-level methods (`readHoldingRegisters`, etc.) use `setID()` + `.data` arguments.
- No `setCoil` alias — use `writeCoil`.
- Enron mode: set `options.enron: true` in connect options for 32-bit register addressing (FC3/4/6 use uint32).
- The `worker/` module provides `send()` and `poll()` for concurrent/batched requests — see `worker/README.md`.
