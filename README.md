from web3 import Web3
from eth_account import Account
from solana.rpc.api import Client as SolanaClient
from solders.keypair import Keypair
from concurrent.futures import ThreadPoolExecutor
from colorama import Fore, init
import threading
import base58
import re
import os
import time

init(autoreset=True)

# =========================================================
# CONFIG
# =========================================================

THREADS = 30
INPUT_FILE = "crypto_results.txt"
RESULT_FILE = "result_balance.txt"

SOLANA_RPCS = [
    "https://rpc.ankr.com/solana",
    "https://api.mainnet-beta.solana.com",
    "https://solana-mainnet.rpc.extrnode.com"
]

# =========================================================
# AUTO CREATE FILE
# =========================================================

if not os.path.exists(INPUT_FILE):
    with open(INPUT_FILE, "w", encoding="utf-8") as f:
        f.write("# private keys here\n")
    print(Fore.YELLOW + f"[INFO] Created {INPUT_FILE}")
    exit()

# =========================================================
# RPC CONNECT (EVM)
# =========================================================

def connect_rpc(rpcs):
    for rpc in rpcs:
        try:
            w3 = Web3(Web3.HTTPProvider(rpc, request_kwargs={"timeout": 10}))
            if w3.is_connected():
                return w3
        except:
            continue
    return None

# =========================================================
# CHAINS
# =========================================================

CHAINS = {
    "ETH": ["https://ethereum-rpc.publicnode.com"],
    "BSC": ["https://bsc-rpc.publicnode.com"],
    "POLYGON": [
        "https://polygon-bor-rpc.publicnode.com",
        "https://polygon-rpc.com",
        "https://rpc.ankr.com/polygon"
    ],
    "ARBITRUM": ["https://arb1.arbitrum.io/rpc"],
    "BASE": ["https://mainnet.base.org"],
    "OPTIMISM": ["https://mainnet.optimism.io"],
    "AVAX": ["https://api.avax.network/ext/bc/C/rpc"],
    "LINEA": ["https://rpc.linea.build"]
}

SYMBOLS = {
    "ETH": "ETH",
    "BSC": "BNB",
    "POLYGON": "MATIC",
    "ARBITRUM": "ETH",
    "BASE": "ETH",
    "OPTIMISM": "ETH",
    "AVAX": "AVAX",
    "LINEA": "ETH"
}

# =========================================================
# LOAD EVM RPC
# =========================================================

web3_instances = {}

for chain, rpcs in CHAINS.items():
    w3 = connect_rpc(rpcs)
    if w3:
        web3_instances[chain] = w3
        print(Fore.GREEN + f"[CONNECTED] {chain}")
    else:
        print(Fore.RED + f"[FAILED] {chain}")

# =========================================================
# SOLANA CONNECT (CLEAN MODE - NO FALSE FAILED SPAM)
# =========================================================

solana = None
solana_ok = False

for rpc in SOLANA_RPCS:
    try:
        client = SolanaClient(rpc)
        client.get_health()
        solana = client
        solana_ok = True
        print(Fore.GREEN + f"[CONNECTED] SOLANA")
        break
    except:
        continue

if not solana_ok:
    print(Fore.YELLOW + "[WARNING] SOLANA NOT AVAILABLE (SKIP ONLY)")

# =========================================================
# VALIDATION
# =========================================================

HEX_64 = re.compile(r'^(0x)?[0-9a-fA-F]{64}$')
MAX_PK = 115792089237316195423570985008687907852837564279074904382605163141518161494337

lock = threading.Lock()

# =========================================================
# SAVE RESULT
# =========================================================

def save_result(address, pk, chain, balance, symbol):
    with lock:
        with open(RESULT_FILE, "a", encoding="utf-8") as f:
            f.write(
                f"ADDRESS: {address}\n"
                f"PRIVATE_KEY: {pk}\n"
                f"CHAIN: {chain}\n"
                f"BALANCE: {balance} {symbol}\n"
                f"{'='*60}\n"
            )

# =========================================================
# EVM CHECK
# =========================================================

def check_evm(private_key):
    try:
        pk = private_key.strip()

        if not HEX_64.match(pk):
            return

        if pk.startswith("0x"):
            pk = pk[2:]

        if int(pk, 16) <= 0 or int(pk, 16) >= MAX_PK:
            return

        account = Account.from_key(pk)
        address = account.address

        found = False

        for chain, w3 in web3_instances.items():
            try:
                wei = w3.eth.get_balance(address)
                bal = float(w3.from_wei(wei, "ether"))

                if bal > 0:
                    found = True
                    print(Fore.GREEN + f"[{chain}] {bal} {SYMBOLS[chain]}")
                    save_result(address, "0x"+pk, chain, bal, SYMBOLS[chain])
            except:
                continue

        if found:
            print(Fore.CYAN + f"[WALLET] {address}")

    except:
        pass

# =========================================================
# SOLANA CHECK (SAFE SKIP IF DOWN)
# =========================================================

def check_solana(private_key):
    if not solana_ok:
        return

    try:
        pk = private_key.strip()

        if HEX_64.match(pk):
            if pk.startswith("0x"):
                pk = pk[2:]

            seed = bytes.fromhex(pk)
            if len(seed) != 32:
                return

            kp = Keypair.from_seed(seed)
        else:
            decoded = base58.b58decode(pk)
            if len(decoded) != 64:
                return

            kp = Keypair.from_bytes(decoded)

        addr = str(kp.pubkey())

        try:
            res = solana.get_balance(kp.pubkey())
            sol = res.value / 1_000_000_000
        except:
            return

        if sol > 0:
            print(Fore.GREEN + f"[SOLANA] {sol} SOL -> {addr}")
            save_result(addr, private_key, "SOLANA", sol, "SOL")

    except:
        pass

# =========================================================
# PROCESS
# =========================================================

def process(pk):
    check_evm(pk)
    check_solana(pk)

# =========================================================
# LOAD FILE (SAFE UTF-8)
# =========================================================

with open(INPUT_FILE, "r", encoding="utf-8", errors="ignore") as f:
    keys = list(set(
        x.strip()
        for x in f
        if x.strip() and not x.startswith("#")
    ))

print("TOTAL KEYS:", len(keys))

start = time.time()

with ThreadPoolExecutor(max_workers=THREADS) as ex:
    ex.map(process, keys)

print("DONE IN", round(time.time() - start, 2), "s")
