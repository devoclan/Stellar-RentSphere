# 🏘️ Rentsphere

> A decentralized "Lending Library" platform where neighbors can rent out tools and equipment, secured by Soroban smart contracts on Stellar blockchain.

[![Stellar](https://img.shields.io/badge/Stellar-Soroban-blue)](https://stellar.org)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

## Overview

Rentsphere enables community members to share tools, equipment, and other items through a peer-to-peer rental marketplace. Smart contracts on Stellar's Soroban platform handle security deposits, rental agreements, and dispute resolution automatically, creating a trustless lending ecosystem.

## Key Features

- Smart Contract Security Deposits**: Automated deposit handling via Soroban
- Item Listings**: Create and browse available tools and equipment
- Rental Management**: Track rental periods, returns, and extensions
- Reputation System**: Build trust through ratings and reviews
- Automated Payments**: Secure payment processing with XLM or custom tokens
- Dispute Resolution**: On-chain arbitration for rental conflicts
- Mobile-First Design**: Responsive interface for all devices

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Frontend Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   React UI   │  │  Wallet SDK  │  │  IPFS Client │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Backend Services                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   API Layer  │  │  Indexer     │  │  Notifier    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Stellar Blockchain Layer                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Soroban Smart Contracts                  │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐     │   │
│  │  │  Rental    │  │  Deposit   │  │  Dispute   │     │   │
│  │  │  Contract  │  │  Escrow    │  │  Resolution│     │   │
│  │  └────────────┘  └────────────┘  └────────────┘     │   │
│  └──────────────────────────────────────────────────────┘   │
│                    Stellar Network (Testnet/Mainnet)         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Storage Layer                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │     IPFS     │  │   Database   │  │    Cache     │      │
│  │  (Metadata)  │  │  (Off-chain) │  │    (Redis)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack

### Frontend
- **React** + **TypeScript**: UI framework
- **Stellar SDK**: Blockchain interaction
- **Freighter Wallet**: User authentication
- **TailwindCSS**: Styling
- **React Query**: State management

### Backend
- **Node.js** + **Express**: API server
- **PostgreSQL**: Off-chain data storage
- **Redis**: Caching layer
- **IPFS**: Decentralized file storage

### Blockchain
- **Soroban**: Smart contract platform
- **Rust**: Contract development language
- **Stellar Network**: Blockchain infrastructure

## Project Structure

```
rentsphere/
├── contracts/                 # Soroban smart contracts
│   ├── rental/               # Main rental contract
│   │   ├── src/
│   │   │   ├── lib.rs       # Contract entry point
│   │   │   ├── types.rs     # Data structures
│   │   │   └── storage.rs   # Storage helpers
│   │   └── Cargo.toml
│   ├── deposit/              # Deposit escrow contract
│   └── dispute/              # Dispute resolution contract
│
├── frontend/                 # React application
│   ├── src/
│   │   ├── components/      # UI components
│   │   ├── hooks/           # Custom React hooks
│   │   ├── services/        # API & blockchain services
│   │   ├── pages/           # Route pages
│   │   └── utils/           # Helper functions
│   └── package.json
│
├── backend/                  # Node.js API server
│   ├── src/
│   │   ├── routes/          # API endpoints
│   │   ├── services/        # Business logic
│   │   ├── models/          # Database models
│   │   └── middleware/      # Express middleware
│   └── package.json
│
├── indexer/                  # Blockchain event indexer
│   └── src/
│       └── index.ts
│
└── docs/                     # Documentation
    ├── API.md
    └── CONTRACTS.md
```

## Smart Contract Code Snippet

Here's a simplified example of the Rental Contract in Rust:

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, Address, Env, Symbol, Vec};

#[contracttype]
#[derive(Clone)]
pub struct RentalItem {
    pub id: u64,
    pub owner: Address,
    pub name: Symbol,
    pub daily_rate: i128,
    pub deposit_amount: i128,
    pub is_available: bool,
}

#[contracttype]
#[derive(Clone)]
pub struct RentalAgreement {
    pub item_id: u64,
    pub renter: Address,
    pub start_time: u64,
    pub end_time: u64,
    pub deposit_paid: i128,
    pub is_active: bool,
}

#[contract]
pub struct RentalContract;

#[contractimpl]
impl RentalContract {
    /// List a new item for rent
    pub fn list_item(
        env: Env,
        owner: Address,
        name: Symbol,
        daily_rate: i128,
        deposit_amount: i128,
    ) -> u64 {
        owner.require_auth();
        
        let item_id = Self::get_next_item_id(&env);
        
        let item = RentalItem {
            id: item_id,
            owner: owner.clone(),
            name,
            daily_rate,
            deposit_amount,
            is_available: true,
        };
        
        env.storage().persistent().set(&item_id, &item);
        
        env.events().publish(
            (Symbol::new(&env, "item_listed"), owner),
            item_id,
        );
        
        item_id
    }
    
    /// Create a rental agreement with deposit
    pub fn rent_item(
        env: Env,
        item_id: u64,
        renter: Address,
        duration_days: u64,
    ) -> Result<(), RentalError> {
        renter.require_auth();
        
        let mut item: RentalItem = env.storage()
            .persistent()
            .get(&item_id)
            .ok_or(RentalError::ItemNotFound)?;
        
        if !item.is_available {
            return Err(RentalError::ItemNotAvailable);
        }
        
        // Transfer deposit to escrow
        let deposit_contract = env.current_contract_address();
        Self::transfer_deposit(&env, &renter, &deposit_contract, item.deposit_amount)?;
        
        let start_time = env.ledger().timestamp();
        let end_time = start_time + (duration_days * 86400); // seconds in a day
        
        let agreement = RentalAgreement {
            item_id,
            renter: renter.clone(),
            start_time,
            end_time,
            deposit_paid: item.deposit_amount,
            is_active: true,
        };
        
        // Mark item as unavailable
        item.is_available = false;
        env.storage().persistent().set(&item_id, &item);
        
        // Store rental agreement
        let agreement_key = (Symbol::new(&env, "rental"), item_id);
        env.storage().persistent().set(&agreement_key, &agreement);
        
        env.events().publish(
            (Symbol::new(&env, "rental_started"), renter),
            (item_id, duration_days),
        );
        
        Ok(())
    }
    
    /// Return item and release deposit
    pub fn return_item(
        env: Env,
        item_id: u64,
        condition_ok: bool,
    ) -> Result<(), RentalError> {
        let agreement_key = (Symbol::new(&env, "rental"), item_id);
        let mut agreement: RentalAgreement = env.storage()
            .persistent()
            .get(&agreement_key)
            .ok_or(RentalError::AgreementNotFound)?;
        
        agreement.renter.require_auth();
        
        if !agreement.is_active {
            return Err(RentalError::AgreementNotActive);
        }
        
        let mut item: RentalItem = env.storage()
            .persistent()
            .get(&item_id)
            .ok_or(RentalError::ItemNotFound)?;
        
        // Calculate refund amount based on condition
        let refund_amount = if condition_ok {
            agreement.deposit_paid
        } else {
            // Partial refund if item damaged
            agreement.deposit_paid / 2
        };
        
        // Transfer deposit back to renter
        let contract_addr = env.current_contract_address();
        Self::transfer_deposit(&env, &contract_addr, &agreement.renter, refund_amount)?;
        
        // Transfer remaining to owner if damaged
        if !condition_ok {
            let owner_amount = agreement.deposit_paid - refund_amount;
            Self::transfer_deposit(&env, &contract_addr, &item.owner, owner_amount)?;
        }
        
        // Mark item as available again
        item.is_available = true;
        env.storage().persistent().set(&item_id, &item);
        
        // Close agreement
        agreement.is_active = false;
        env.storage().persistent().set(&agreement_key, &agreement);
        
        env.events().publish(
            (Symbol::new(&env, "rental_completed"), agreement.renter),
            (item_id, condition_ok),
        );
        
        Ok(())
    }
    
    // Helper functions
    fn get_next_item_id(env: &Env) -> u64 {
        let key = Symbol::new(env, "next_id");
        let id: u64 = env.storage().persistent().get(&key).unwrap_or(1);
        env.storage().persistent().set(&key, &(id + 1));
        id
    }
    
    fn transfer_deposit(
        env: &Env,
        from: &Address,
        to: &Address,
        amount: i128,
    ) -> Result<(), RentalError> {
        // Implement token transfer logic
        // This would use Stellar Asset Contract (SAC)
        Ok(())
    }
}

#[contracttype]
pub enum RentalError {
    ItemNotFound = 1,
    ItemNotAvailable = 2,
    AgreementNotFound = 3,
    AgreementNotActive = 4,
    InsufficientDeposit = 5,
}
```

## Getting Started

### Prerequisites

- Node.js v18+
- Rust 1.70+
- Stellar CLI (`stellar-cli`)
- Docker (optional, for local development)

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/rentsphere.git
   cd rentsphere
   ```

2. **Install dependencies**
   ```bash
   # Install frontend dependencies
   cd frontend
   npm install
   
   # Install backend dependencies
   cd ../backend
   npm install
   ```

3. **Build smart contracts**
   ```bash
   cd ../contracts/rental
   cargo build --target wasm32-unknown-unknown --release
   
   # Optimize the contract
   stellar contract optimize \
     --wasm target/wasm32-unknown-unknown/release/rental_contract.wasm
   ```

4. **Deploy contracts to testnet**
   ```bash
   stellar contract deploy \
     --wasm target/wasm32-unknown-unknown/release/rental_contract.wasm \
     --network testnet
   ```

5. **Configure environment variables**
   ```bash
   # Frontend (.env)
   REACT_APP_CONTRACT_ID=<your_contract_id>
   REACT_APP_NETWORK=testnet
   
   # Backend (.env)
   DATABASE_URL=postgresql://user:pass@localhost:5432/rentsphere
   STELLAR_NETWORK=testnet
   CONTRACT_ID=<your_contract_id>
   ```

6. **Start the development servers**
   ```bash
   # Terminal 1: Backend
   cd backend
   npm run dev
   
   # Terminal 2: Frontend
   cd frontend
   npm start
   ```

## Usage Example

### Listing an Item

```typescript
import { SorobanRpc, Contract } from '@stellar/stellar-sdk';

const listItem = async (
  name: string,
  dailyRate: number,
  depositAmount: number
) => {
  const contract = new Contract(CONTRACT_ID);
  
  const tx = await contract.call(
    'list_item',
    {
      owner: userAddress,
      name: name,
      daily_rate: dailyRate * 10_000_000, // Convert to stroops
      deposit_amount: depositAmount * 10_000_000,
    }
  );
  
  const result = await tx.send();
  return result.itemId;
};
```

### Renting an Item

```typescript
const rentItem = async (
  itemId: number,
  durationDays: number
) => {
  const contract = new Contract(CONTRACT_ID);
  
  const tx = await contract.call(
    'rent_item',
    {
      item_id: itemId,
      renter: userAddress,
      duration_days: durationDays,
    }
  );
  
  await tx.send();
};
```

## Testing

```bash
# Run contract tests
cd contracts/rental
cargo test

# Run frontend tests
cd frontend
npm test

# Run backend tests
cd backend
npm test

# Run integration tests
npm run test:integration
```

##  Roadmap

- [x] Core rental contract
- [x] Deposit escrow system
- [ ] Dispute resolution mechanism
- [ ] Mobile app (React Native)
- [ ] Multi-token support (USDC, custom tokens)
- [ ] Insurance integration
- [ ] Community governance (DAO)
- [ ] Cross-chain bridge support

## Contributing

Contributions are welcome! Please read our [Contributing Guide](CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [Stellar Development Foundation](https://stellar.org)
- [Soroban Documentation](https://soroban.stellar.org)
- The amazing blockchain community


Built with love using Stellar Soroban
