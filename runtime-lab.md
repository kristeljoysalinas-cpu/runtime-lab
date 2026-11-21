# Runtime Lab — Module 1: Modern JavaScript and TypeScript Runtimes and Tooling

**Activity 1.1:** Monorepo Runtime Comparison and Security Audit

## 1. Project setup

The monorepo was created with Turborepo (`pnpm dlx create-turbo@latest`, npm workspaces). Two simple apps were added under `apps/`:

```
runtime-lab/
├── apps/
│   ├── hello-node/        # runs on Node.js
│   │   ├── hello.js
│   │   └── package.json   # "start": "node hello.js"
│   ├── hello-bun/         # runs on Bun
│   │   ├── hello.js
│   │   └── package.json   # "start": "bun hello.js"
│   ├── web/               # Next.js app from the create-turbo template
│   └── docs/              # Next.js app from the create-turbo template
├── packages/              # shared ui, eslint-config, typescript-config
├── package.json           # npm workspaces + "start" script
├── turbo.json             # Turborepo task pipeline ("start" task added)
└── runtime-lab.md         # this brief
```

`apps/hello-node/hello.js`

```js
console.log("Hello from Node in Monorepo");
console.log("Runtime: Node.js " + process.version);
```

`apps/hello-bun/hello.js`

```js
console.log("Hello from Bun in Monorepo");
console.log("Runtime: Bun v" + Bun.version);
```

The second line in each file prints the runtime's version, which shows that each app really ran on its own runtime. (`Bun.version` exists only in Bun, so `hello-bun` cannot run on Node.)

## 2. How to run

```bash
npm install                      # install workspace dependencies
node apps/hello-node/hello.js    # run the Node app
bun apps/hello-bun/hello.js      # run the Bun app
npm start                        # run both through Turborepo (turbo run start)
```

## 3. Output

```
> node apps\hello-node\hello.js
Hello from Node in Monorepo
Runtime: Node.js v26.11.1

> bun apps\hello-bun\hello.js
Hello from Bun in Monorepo
Runtime: Bun v1.4.4
```

`turbo run start` ran both tasks in parallel: **2 successful, 2 total**.

Machine used: Windows 11 (Command Prompt) with Node.js v26.11.1, npm 11.20.0, Bun 1.4.4 and Turborepo 2.11.13.

### Issues encountered and how they were solved

1. **npm version mismatch.** The template's `package.json` required exactly npm 10.9.9, but this PC has npm 11.20.0, so npm stopped with `EBADDEVENGINES`. The field was changed to `"packageManager": "npm@10.9.9"`, which Turborepo still reads but npm does not enforce.
2. **Turborepo blocked by Windows.** Windows Smart App Control (Device Guard) blocked `turbo.exe`. Smart App Control was turned off in Windows Security, after which `turbo` ran normally.

## 4. Security audit

| Step | Command | Result |
|---|---|---|
| First scan | `npm audit` | **3 vulnerabilities (2 moderate, 1 critical)**: `next` 16.3.8 (critical, including remote code execution advisories) and `minimatch` via `eslint` 10.9.1 (moderate) |
| Update | `npm update` | No change: the template pins exact versions, so `npm update` stays inside those ranges |
| Fix | `npm audit fix --force` | `next` upgraded to 16.4.6 and `eslint` to 10.15.0. The stray `next` entries that npm added to `packages/ui` and `packages/eslint-config` were removed by hand. |
| Re-scan | `npm audit` | **found 0 vulnerabilities** |
| Snyk | `snyk auth` then `snyk test --all-projects` | **8 projects tested, 0 security issues** |

All the vulnerable packages came from the template's Next.js demo apps. `hello-node` and `hello-bun` have no dependencies. After the upgrade, `turbo run build` still builds the whole repo without errors.

## 5. Runtime selection brief: real-time chat application

**Recommendation: Node.js (an LTS release)**

A real-time chat app keeps thousands of WebSocket connections open and pushes small messages between them. Most of its time is spent waiting on the network, so the event-loop model of every runtime handles this well. What matters most is how reliable the runtime is, which tools are available for it, and how well it can be kept secure in production.

**Reason 1: Ecosystem.** Node.js has the most mature real-time libraries: Socket.IO (rooms, automatic reconnection, fallback when WebSockets are blocked) and `ws`. It also has proven ways to scale past one server, such as the Socket.IO Redis adapter. Hosting providers, tutorials and monitoring tools all support Node first, so problems are faster to solve.

**Reason 2: Stability (and speed).** Bun is clearly faster at starting up: in a quick test while building the project (30 runs each), it started in about **5 ms vs 26 ms** for Node, and its built-in WebSocket server is very fast. But a chat server starts once and then runs for days, so startup time matters little. Node's long-term track record under heavy, long-running load is worth more here than Bun's raw speed.

**Security reason:** Node.js LTS releases get scheduled security patches for about 30 months, so a production chat server can stay patched without major upgrades. The supply-chain tools used in this lab, `npm audit` and Snyk, fully support Node/npm projects, so vulnerable dependencies are found and fixed early. This matters for a chat app, because it handles user messages and login sessions.

**When I would pick Bun instead:** for a small or new project where raw throughput and fast development matter most. Bun's built-in `Bun.serve()` WebSocket support with pub/sub needs no extra libraries.
