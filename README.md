# Autometa 🚀

**No-Code Blockchain Workflow Automation on Polkadot**

Autometa is a comprehensive automation platform that enables users to create, schedule, and execute blockchain workflows without writing code. Built on **Moonbeam** (a Polkadot parachain), it combines smart contracts, a Python backend/worker system, and a modern web interface to automate on-chain actions based on time triggers, price conditions, or wallet events.

---

## 🌟 Overview

Autometa allows users to:
- **Create workflows** with triggers (time-based, price-based, or event-based) and actions (token transfers, contract calls)
- **Schedule automated execution** of workflows at specified intervals
- **Monitor execution** through a comprehensive dashboard with real-time transaction tracking
- **Manage gas budgets** via an escrow system that handles execution costs

The platform leverages the **Polkadot ecosystem** through Moonbeam's EVM compatibility layer, enabling Ethereum-compatible smart contracts to run on a Polkadot parachain while benefiting from Polkadot's security and interoperability.

---

## 🏗️ Architecture

### System Components

```
┌─────────────────┐
│   Frontend      │  Next.js + Wagmi + Viem
│  (User Portal)  │  - Create/manage workflows
└────────┬────────┘  - View dashboards & transactions
         │           - Connect wallet (MetaMask)
         │
         ▼
┌─────────────────┐
│   Backend API   │  FastAPI + Web3.py
│  (REST Service) │  - Workflow CRUD operations
└────────┬────────┘  - Escrow balance queries
         │           - Price feed integration
         │           - Event log querying
         │
         ▼
┌─────────────────────────────────────────────┐
│          Moonbase Alpha (Polkadot)          │
│                                             │
│  ┌───────────────┐  ┌──────────────┐      │
│  │ WorkflowRegistry│  │ FeeEscrow   │      │
│  │  - Stores workflows│ - Gas deposits│      │
│  │  - Owner mappings │ - Gas charging│      │
│  └───────┬───────┘  └──────┬───────┘      │
│          │                  │              │
│          └──────────┬───────┘              │
│                     │                      │
│          ┌──────────▼──────────┐           │
│          │  ActionExecutor     │           │
│          │  - Execute workflows│           │
│          │  - Handle gas fees  │           │
│          └─────────────────────┘           │
└─────────────────────────────────────────────┘
         ▲
         │
┌────────┴────────┐
│  Worker System  │  Python + Redis + Web3.py
│                 │
│  ┌──────────┐  │  - Scheduler: Monitors workflows
│  │Scheduler │  │    every 10s, enqueues ready jobs
│  └─────┬────┘  │
│        │       │  - Worker: Executes workflows from
│        ▼       │    Redis queue, calls contracts
│  ┌──────────┐  │
│  │ Worker   │  │  - Single execution attempt
│  └──────────┘  │  - No infinite retries
└─────────────────┘
```

### Smart Contract Architecture

**Three core contracts deployed on Moonbase Alpha:**

1. **WorkflowRegistry.sol**
   - Stores workflow definitions (trigger, action, schedule, owner)
   - Tracks workflow state (active/paused, next run time)
   - Emits events for workflow lifecycle (created, updated, executed)

2. **FeeEscrow.sol**
   - Manages gas deposits for users
   - Allows worker to charge gas from user balances
   - Provides deposit/withdrawal functionality

3. **ActionExecutor.sol**
   - Executes workflow actions on-chain (token transfers, contract calls)
   - Updates workflow `nextRun` timestamp after execution
   - Charges gas fees from escrow (only for successful executions in updated version)
   - Restricted to authorized worker addresses via `WORKER_ROLE`

---

## 🔗 Polkadot Integration

### Why Moonbeam?

**Moonbeam** is an Ethereum-compatible smart contract parachain on **Polkadot**. By building on Moonbeam, Autometa gains:

1. **EVM Compatibility**: Deploy Solidity contracts without modifications
2. **Polkadot Security**: Inherit security from Polkadot's shared security model
3. **Cross-Chain Potential**: Future integration with other Polkadot parachains via XCM (Cross-Consensus Messaging)
4. **Low Fees**: Benefit from Polkadot's efficient consensus and Moonbeam's gas optimization
5. **Developer Tooling**: Use familiar Ethereum tools (Hardhat, Web3.py, Ethers.js, MetaMask)

### Moonbase Alpha Testnet

The project is configured for **Moonbase Alpha**, Moonbeam's testnet:

- **Chain ID**: 1287
- **RPC**: `https://rpc.api.moonbase.moonbeam.network`
- **Native Token**: DEV (test token)
- **Explorer**: [Moonbase Moonscan](https://moonbase.moonscan.io/)
- **Faucet**: [Get testnet DEV](https://faucet.moonbeam.network/)

### Polkadot-Specific Features Used

- **POA Middleware**: Web3.py configured with Proof-of-Authority middleware for Moonbeam compatibility
- **EIP-1559 Transactions**: Worker uses EIP-1559 format (maxFeePerGas, maxPriorityFeePerGas) adapted for Moonbeam
- **Token Mappings**: Price adapters support DOT → Polkadot and GLMR → Moonbeam asset mappings
- **Moonbeam Libraries**: Frontend uses `viem/chains` with `moonbaseAlpha` chain configuration

---

## 🔄 Complete Workflow Execution Flow

### 1. Workflow Creation

**User Journey:**
```
User → Frontend → Backend API → WorkflowRegistry Contract
```

1. User connects wallet (MetaMask) to frontend
2. User creates workflow via UI:
   - **Trigger**: Time-based (interval in seconds), price condition, or wallet event
   - **Action**: Token transfer, native currency transfer, or contract call
   - **Schedule**: Next run time and recurrence interval
   - **Gas Budget**: Amount (in DEV) allocated for execution
3. Frontend calls backend API `/api/workflow/create`
4. Backend validates workflow and calls `WorkflowRegistry.createWorkflow()`
5. Contract stores workflow on-chain with unique `workflowId`
6. Event `WorkflowCreated` emitted
7. User deposits gas to `FeeEscrow` contract for execution costs

### 2. Workflow Scheduling & Execution

**Automated Execution Flow:**
```
Scheduler → WorkflowRegistry (read) → Redis Queue → Worker → ActionExecutor Contract
```

**Every 10 seconds, the Scheduler:**
1. Queries `WorkflowRegistry` for all active workflows
2. Checks `nextRun` timestamp for each workflow
3. If `currentTime >= nextRun`, enqueues workflow to Redis queue
4. Workflow job includes: `workflowId`, `owner`, `actionData`, `gasBudget`, `interval`

**Worker Process:**
1. Dequeues job from Redis
2. **Balance Check**: Queries `FeeEscrow` to verify user has sufficient balance
   - If insufficient → logs warning, skips execution (waits for next scheduled time)
3. Prepares transaction data:
   - Decodes `actionData` from workflow
   - Calculates `newNextRun = currentTime + interval`
4. Calls `ActionExecutor.executeWorkflow()`:
   ```solidity
   function executeWorkflow(
       uint256 workflowId,
       bytes calldata actionData,
       uint256 newNextRun,
       address userToCharge,
       uint256 gasToCharge
   )
   ```
5. **ActionExecutor Logic**:
   - Executes the action (transfer, contract call, etc.)
   - **If successful**: Charges gas from user's escrow balance
   - **If failed**: No gas charged (in updated version), logs error
   - Updates `workflow.nextRun` in `WorkflowRegistry`
   - Emits `WorkflowExecuted` event
6. Worker logs result (success/failure) with transaction hash
7. **No retries**: Failed workflows wait for next scheduled execution time

### 3. Monitoring & Management

**User Dashboard:**
- View all workflows with status badges (Active, Paused, Pending)
- See execution counts, creation timestamps, next run times
- Monitor escrow balance
- View transaction history with Moonscan links
- Pause/resume/delete workflows
- Deposit/withdraw gas funds

**Transaction Tracking:**
- Each execution emits blockchain events
- Frontend queries events via backend API
- Displays: workflow ID, timestamp, gas used, success/failure status
- Links to Moonscan for full transaction details

---

## 📁 Project Structure

```
autometa/
├── contracts/                 # Solidity smart contracts
│   ├── contracts/
│   │   ├── WorkflowRegistry.sol    # Workflow storage & management
│   │   ├── ActionExecutor.sol      # Execute workflows on-chain
│   │   ├── FeeEscrow.sol           # Gas deposit & charging
│   │   └── MockERC20.sol           # Test token for development
│   ├── scripts/               # Hardhat deployment & helper scripts
│   ├── test/                  # Contract unit tests
│   └── hardhat.config.js      # Hardhat config (Moonbase Alpha)
│
├── backend/                   # FastAPI REST service
│   ├── src/
│   │   ├── api/               # API route handlers
│   │   │   ├── workflow.py    # Workflow CRUD + metadata
│   │   │   ├── escrow.py      # Balance queries
│   │   │   ├── price.py       # Price feed integration
│   │   │   └── utils.py       # Action templates
│   │   ├── services/          # Business logic
│   │   │   ├── registry_service.py
│   │   │   ├── escrow_service.py
│   │   │   └── price_service.py
│   │   ├── utils/             # Web3 provider, config, logging
│   │   └── main.py            # FastAPI app entry point
│   ├── abi/                   # Contract ABIs (synced from contracts/)
│   └── requirements.txt       # Python dependencies
│
├── worker/                    # Python scheduler + worker
│   ├── src/
│   │   ├── scheduler/         # Workflow monitoring & enqueueing
│   │   ├── executors/         # Job execution logic
│   │   │   ├── job_worker.py  # Main workflow executor
│   │   │   └── evm_executor.py # EVM transaction handling
│   │   ├── triggers/          # Trigger evaluation (time, price)
│   │   ├── adapters/          # Price feeds (CoinGecko)
│   │   └── main.py            # Worker entry point
│   ├── tools/                 # Utility scripts
│   └── requirements.txt       # Python dependencies
│
├── frontend/                  # Next.js web application
│   ├── src/
│   │   ├── app/               # Next.js app router pages
│   │   │   ├── dashboard/     # Workflow dashboard
│   │   │   ├── create/        # Workflow creation wizard
│   │   │   └── layout.tsx     # Root layout
│   │   ├── components/        # React components
│   │   │   ├── dashboard/     # Dashboard cards & tables
│   │   │   ├── wallet/        # Wallet connection UI
│   │   │   └── footer.tsx     # Footer with Moonbeam links
│   │   ├── config/            # Contract addresses & ABIs
│   │   └── lib/               # Utilities
│   ├── public/                # Static assets
│   └── package.json           # Node dependencies
│
└── README.md                  # This file
```

---

## 🚀 Getting Started

### Prerequisites

- **Node.js** 18+ (for frontend)
- **Python** 3.10+ (for backend & worker)
- **Redis** (for worker queue)
- **MetaMask** or compatible wallet
- **DEV tokens** from [Moonbeam Faucet](https://faucet.moonbeam.network/)

### 1. Smart Contracts Setup

```bash
cd contracts

# Install dependencies
npm install

# Compile contracts
npx hardhat compile

# Deploy to Moonbase Alpha (requires DEPLOYER_PRIVATE_KEY in .env)
npx hardhat run scripts/deploy.js --network moonbaseAlpha

# Grant WORKER_ROLE to worker address
npx hardhat run scripts/grant-worker-role.js --network moonbaseAlpha
```

**Note the deployed contract addresses** - you'll need them for backend/worker/frontend configuration.

### 2. Backend Setup

```bash
cd backend

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment (.env file)
cat > .env << EOF
MOONBASE_RPC=https://rpc.api.moonbase.moonbeam.network
CHAIN_ID=1287
WORKFLOW_REGISTRY=<your_deployed_address>
ACTION_EXECUTOR=<your_deployed_address>
FEE_ESCROW=<your_deployed_address>
BACKEND_API_URL=http://localhost:8000/api
EOF

# Start backend server
uvicorn src.main:app --reload --port 8000
```

Backend will be available at `http://localhost:8000`

### 3. Worker Setup

```bash
cd worker

# Create virtual environment (or reuse backend's)
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Configure environment (.env file)
cat > .env << EOF
MOONBASE_RPC=https://rpc.api.moonbase.moonbeam.network
CHAIN_ID=1287
WORKFLOW_REGISTRY=<your_deployed_address>
ACTION_EXECUTOR=<your_deployed_address>
FEE_ESCROW=<your_deployed_address>
BACKEND_API_URL=http://localhost:8000/api
REDIS_URL=redis://localhost:6379/0
WORKER_PRIVATE_KEY=<your_worker_private_key>
EOF

# Start Redis (in separate terminal)
redis-server

# Start scheduler
nohup python -m src.main scheduler >> scheduler.log 2>&1 &

# Start worker
nohup python -m src.main worker >> worker.log 2>&1 &

# Monitor logs
tail -f worker.log
```

### 4. Frontend Setup

```bash
cd frontend

# Install dependencies
npm install

# Configure environment (.env.local file)
cat > .env.local << EOF
NEXT_PUBLIC_WORKFLOW_REGISTRY=<your_deployed_address>
NEXT_PUBLIC_ACTION_EXECUTOR=<your_deployed_address>
NEXT_PUBLIC_FEE_ESCROW=<your_deployed_address>
NEXT_PUBLIC_CHAIN_ID=1287
NEXT_PUBLIC_CHAIN_NAME=Moonbase Alpha
NEXT_PUBLIC_RPC_URL=https://rpc.api.moonbase.moonbeam.network
NEXT_PUBLIC_BACKEND_API=http://localhost:8000/api
EOF

# Start development server
npm run dev
```

Frontend will be available at `http://localhost:3000`

---

## 💡 Usage Guide

### Creating Your First Workflow

1. **Get Testnet Tokens**
   - Visit [Moonbeam Faucet](https://faucet.moonbeam.network/)
   - Request DEV tokens for your wallet address

2. **Connect Wallet**
   - Open frontend at `http://localhost:3000`
   - Click "Connect Wallet" and approve MetaMask connection
   - Ensure you're on Moonbase Alpha network (Chain ID 1287)

3. **Deposit Gas to Escrow**
   - Navigate to Dashboard → Escrow
   - Deposit DEV tokens (e.g., 1.0 DEV) to cover execution costs
   - Confirm transaction in MetaMask

4. **Create Workflow**
   - Click "Create Workflow"
   - **Configure Trigger**:
     - Time-based: Set interval (e.g., 60 seconds)
     - Price-based: Set token and threshold
   - **Configure Action**:
     - Transfer: Specify recipient and amount
     - Contract call: Provide contract address and calldata
   - **Set Schedule**:
     - Next run: When first execution should occur
     - Interval: How often to repeat (0 for one-time)
   - **Set Gas Budget**: Amount allocated per execution (e.g., 0.1 DEV)
   - Submit transaction and confirm in MetaMask

5. **Monitor Execution**
   - View workflow in dashboard
   - Check status badges (Pending → Active → Executed)
   - See execution count increment
   - Click transaction links to view on Moonscan

### Managing Workflows

- **Pause**: Temporarily disable execution (keeps schedule)
- **Resume**: Re-enable paused workflow
- **Delete**: Permanently remove workflow (cannot undo)
- **View Logs**: See execution history and gas consumption

---

## 🔧 Advanced Configuration

### Environment Variables Reference

**Backend/Worker (.env):**
```bash
# Moonbase Alpha RPC
MOONBASE_RPC=https://rpc.api.moonbase.moonbeam.network
CHAIN_ID=1287

# Contract addresses (from deployment)
WORKFLOW_REGISTRY=0x...
ACTION_EXECUTOR=0x...
FEE_ESCROW=0x...

# API configuration
BACKEND_API_URL=http://localhost:8000/api

# Redis queue (worker only)
REDIS_URL=redis://localhost:6379/0

# Worker private key (worker only)
WORKER_PRIVATE_KEY=0x...

# Price feed API keys (optional)
COINGECKO_API_KEY=your_api_key
```

**Frontend (.env.local):**
```bash
# Contract addresses
NEXT_PUBLIC_WORKFLOW_REGISTRY=0x...
NEXT_PUBLIC_ACTION_EXECUTOR=0x...
NEXT_PUBLIC_FEE_ESCROW=0x...

# Network configuration
NEXT_PUBLIC_CHAIN_ID=1287
NEXT_PUBLIC_CHAIN_NAME=Moonbase Alpha
NEXT_PUBLIC_RPC_URL=https://rpc.api.moonbase.moonbeam.network

# Backend API
NEXT_PUBLIC_BACKEND_API=http://localhost:8000/api
```

### Contract Redeployment

If you need to update contracts:

```bash
cd contracts

# Update contract code
# Then redeploy
npx hardhat run scripts/deploy.js --network moonbaseAlpha

# Grant roles to worker
npx hardhat run scripts/grant-worker-role.js --network moonbaseAlpha

# Update contract addresses in all .env files
# Restart backend, worker, and frontend
```

---

## 🎯 Key Features

### ✅ No Infinite Retries
- Worker executes each workflow **once** per scheduled time
- Failed executions wait for next scheduled interval
- Prevents gas waste from repeated failures
- Balance checks before execution to avoid unnecessary transactions

### ✅ Gas Management
- User deposits DEV to escrow before creating workflows
- Gas charged only for successful executions
- Transparent balance tracking in dashboard
- Users can withdraw unused funds anytime

### ✅ Real-Time Monitoring
- Live workflow status updates
- Execution count tracking
- Creation timestamps and next run times
- Transaction history with blockchain explorer integration

### ✅ Flexible Workflows
- Multiple trigger types (time, price, events)
- Various actions (transfers, contract calls)
- Recurring or one-time execution
- Pause/resume without losing schedule

---

## 🛠️ Troubleshooting

### Common Issues

**1. Worker Not Executing Workflows**
```bash
# Check if scheduler is running
pgrep -af "src.main scheduler"

# Check if worker is running
pgrep -af "src.main worker"

# View worker logs
tail -f worker/worker.log

# Check Redis connection
redis-cli ping
```

**2. Insufficient Balance Errors**
```bash
# Check escrow balance via API
curl http://localhost:8000/api/escrow/balance/<your_wallet_address>

# Deposit more DEV via frontend
# Or check on Moonscan: https://moonbase.moonscan.io/address/<FEE_ESCROW_ADDRESS>
```

**3. Transaction Failures**
- Verify you're on Moonbase Alpha network (Chain ID 1287)
- Check gas budget is sufficient (minimum ~0.1 DEV per execution)
- Ensure worker has WORKER_ROLE on ActionExecutor contract
- View transaction details on Moonscan for specific error messages

**4. Frontend Not Connecting**
```bash
# Check backend is running
curl http://localhost:8000/api/utils/info

# Verify contract addresses match in .env.local
# Clear browser cache and reconnect wallet
# Check MetaMask network (should be Moonbase Alpha)
```

---

## 📊 Performance & Scalability

### Current Configuration
- **Scheduler interval**: 10 seconds (configurable)
- **Worker concurrency**: Single worker (horizontally scalable)
- **Redis queue**: Reliable job persistence
- **Gas efficiency**: Single transaction per execution (no retries)

### Scaling Recommendations
- Run multiple worker processes for higher throughput
- Use Redis Cluster for distributed queue
- Deploy backend with load balancer
- Consider L2/rollup solutions for even lower fees

---

## 🔐 Security Considerations

### Smart Contracts
- ✅ AccessControl for role-based permissions
- ✅ ReentrancyGuard on all state-changing functions
- ✅ Only authorized workers can execute workflows
- ✅ Users control their own workflows (owner checks)

### Off-Chain Components
- ⚠️ Worker private key must be securely stored
- ⚠️ Backend should use API authentication in production
- ⚠️ Redis should require authentication
- ⚠️ Use HTTPS/WSS in production deployments

### Recommended Production Hardening
- Enable API rate limiting
- Add authentication middleware (JWT/OAuth)
- Use secure Redis (password + TLS)
- Store private keys in HSM or secret manager
- Enable CORS restrictions
- Add monitoring/alerting (Grafana, Prometheus)

---

## 🌐 Polkadot Ecosystem Resources

### Moonbeam Documentation
- [Moonbeam Docs](https://docs.moonbeam.network/)
- [Moonbase Alpha Testnet](https://docs.moonbeam.network/networks/testnet/)
- [Moonbeam Developer Portal](https://moonbeam.network/developers/)

### Tools & Explorers
- [Moonbase Moonscan](https://moonbase.moonscan.io/) - Block explorer
- [Moonbeam Faucet](https://faucet.moonbeam.network/) - Get testnet DEV
- [Polkadot.js Apps](https://polkadot.js.org/apps/) - Polkadot ecosystem UI

### Community & Support
- [Moonbeam Discord](https://discord.gg/moonbeam)
- [Polkadot Forum](https://forum.polkadot.network/)
- [Moonbeam GitHub](https://github.com/moonbeam-foundation)

---

## 🤝 Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📝 License

This project is licensed under the MIT License.

---

## 🙏 Acknowledgments

- **Moonbeam Foundation** for providing EVM compatibility on Polkadot
- **Polkadot** for the interoperable multi-chain ecosystem
- **OpenZeppelin** for secure smart contract libraries
- **CoinGecko** for price feed data

---

**Built with ❤️ on the Polkadot ecosystem**

## How to run (quick)

1. Backend

- Create a Python venv and install requirements from `backend/requirements.txt`.
- Set `MOONBASE_RPC` and other env variables in `backend/.env`.
- Start FastAPI server (example):

```bash
cd backend
# activate venv
python -m uvicorn src.main:app --reload --port 8000
```

2. Worker & Scheduler

- Create a Python venv and install `worker/requirements.txt`.
- Configure `worker/.env` (ensure `MOONBASE_RPC`, `BACKEND_API_URL`, Redis URL, private keys, etc.).
- Start scheduler and worker (examples):

```bash
cd worker
# activate venv
nohup python -m src.main scheduler >> scheduler.log 2>&1 &
nohup python -m src.main worker >> worker.log 2>&1 &
```

3. Frontend

- `cd frontend` and install node deps
- Add `frontend/.env.local` values (RPC and contract addresses)
- Start Next.js dev server: `npm run dev`

## Notes & caveats

- Although running on Moonbeam (an ecosystem project inside Polkadot), this code is EVM-oriented — contracts are Solidity and interactions use web3/ethers-style patterns.
- The current setup uses Moonbase Alpha testnet for development and tests.
- Some behavior (for example gas charging on failed contract calls) may require contract redeploys to change; the worker fixes can mitigate operational issues without redeploying contracts.

## Useful links

- Moonbeam docs: https://docs.moonbeam.network/
- Moonbase Alpha info: https://docs.moonbeam.network/networks/testnet/
- Moonscan (Moonbase explorer): https://moonbase.moonscan.io/
- Moonbeam faucet: https://faucet.moonbeam.network/

