# Prompt History

## conversation1

### prompt1
とあるコントラクトが、過去にどのメソッドがどこから叩かれたかを網羅的に知りたいです。期間としては直近1週間くらいで大丈夫です。
というのも、Etherscanなどからでは微妙に分かりにくいからです。最終的な出力は、csv形式で少なくとも以下の内容は欲しいです。

- txHash: その関数呼び出しのトランザクションハッシュ
- method: 叩かれたメソッド名
- caller: 呼び出したEOA/contract address
- isInternal: internal txとして呼ばれたのか, EOAから直接呼ばれたのか

Pythonで`main.py`という1ファイルのスクリプトを書いていただけますか？

### reply1

`main.py` を作成しました。このスクリプトは以下の機能を持ちます：

* 過去1週間の対象コントラクトへの外部トランザクション（EOA または他のコントラクトからの直接呼び出し）を取得
* 同期間の内部トランザクション（他コントラクト実行中の呼び出し）を取得
* 各呼び出しのトランザクションハッシュ、メソッド名、呼び出し元アドレス、`isInternal` フラグを CSV (`output.csv`) に出力

必要な Python パッケージ：

```bash
pip install web3 requests
```

実行例：

```bash
python main.py --address 0xYourContractAddress --apikey YourEtherscanApiKey --network mainnet
```

ご不明点や機能追加のご要望があればお知らせください。

```python
#!/usr/bin/env python3
"""
main.py: Fetch all external and internal method calls made to a specified Ethereum contract over the past week and output the results as CSV.

Usage:
    python main.py --address <CONTRACT_ADDRESS> --apikey <ETHERSCAN_API_KEY> [--network <NETWORK>]

Example:
    python main.py --address 0x1234...abcd --apikey YourEtherscanKey --network mainnet
"""
import time
import requests
import json
import csv
import argparse
from web3 import Web3


def get_block_by_time(base_url, params):
    """Fetch block number closest to a given timestamp via Etherscan API"""
    r = requests.get(base_url, params=params)
    r.raise_for_status()
    data = r.json()
    if data.get('status') != '1':
        raise RuntimeError(f"Error fetching block number: {data.get('message')} / {data.get('result')}")
    return int(data['result'])


def main():
    parser = argparse.ArgumentParser(description="Fetch contract method calls from the past week.")
    parser.add_argument(
        '--address', required=True,
        help="Target contract address (in checksum or non-checksum form)"
    )
    parser.add_argument(
        '--apikey', required=True,
        help="Your Etherscan API key"
    )
    parser.add_argument(
        '--network', default="mainnet",
        choices=["mainnet", "ropsten", "rinkeby", "kovan", "goerli"],
        help="Ethereum network to query (default: mainnet)"
    )
    args = parser.parse_args()

    address = Web3.toChecksumAddress(args.address)
    api_key = args.apikey
    network = args.network.lower()

    # Determine base API URL
    if network == 'mainnet':
        base_url = 'https://api.etherscan.io/api'
    else:
        base_url = f'https://api-{network}.etherscan.io/api'

    # Compute timestamps for the past week
    now_ts = int(time.time())
    one_week_ago_ts = now_ts - 7 * 24 * 60 * 60

    # Find the block range for those timestamps
    start_block = get_block_by_time(
        base_url,
        {
            'module': 'block',
            'action': 'getblocknobytime',
            'timestamp': one_week_ago_ts,
            'closest': 'before',
            'apikey': api_key,
        }
    )
    end_block = get_block_by_time(
        base_url,
        {
            'module': 'block',
            'action': 'getblocknobytime',
            'timestamp': now_ts,
            'closest': 'before',
            'apikey': api_key,
        }
    )
    print(f"Scanning blocks {start_block} to {end_block} for contract {address}...")

    # Fetch contract ABI from Etherscan
    resp = requests.get(
        base_url,
        params={
            'module': 'contract',
            'action': 'getabi',
            'address': address,
            'apikey': api_key,
        }
    )
    resp.raise_for_status()
    abi_data = resp.json()
    if abi_data.get('status') != '1':
        raise RuntimeError(f"Failed to fetch ABI: {abi_data.get('message')} / {abi_data.get('result')}")

    abi = json.loads(abi_data['result'])
    w3 = Web3()
    contract = w3.eth.contract(address=address, abi=abi)

    # Fetch external transactions
    ext_txs_resp = requests.get(
        base_url,
        params={
            'module': 'account',
            'action': 'txlist',
            'address': address,
            'startblock': start_block,
            'endblock': end_block,
            'sort': 'asc',
            'apikey': api_key,
        }
    )
    ext_txs_resp.raise_for_status()
    ext_txs = ext_txs_resp.json().get('result', [])

    # Fetch internal transactions
    int_txs_resp = requests.get(
        base_url,
        params={
            'module': 'account',
            'action': 'txlistinternal',
            'address': address,
            'startblock': start_block,
            'endblock': end_block,
            'sort': 'asc',
            'apikey': api_key,
        }
    )
    int_txs_resp.raise_for_status()
    int_txs = int_txs_resp.json().get('result', [])

    # Prepare CSV output
    output_file = 'output.csv'
    with open(output_file, 'w', newline='') as csvfile:
        fieldnames = ['txHash', 'method', 'caller', 'isInternal']
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()

        # External calls: direct from EOA or other contracts
        for tx in ext_txs:
            method = 'UNKNOWN'
            try:
                fn_obj, _ = contract.decode_function_input(tx.get('input', ''))
                method = fn_obj.fn_name
            except Exception:
                pass
            writer.writerow({
                'txHash': tx['hash'],
                'method': method,
                'caller': tx['from'],
                'isInternal': False,
            })

        # Internal calls: executed inside other contract contexts
        for tx in int_txs:
            method = 'UNKNOWN'
            try:
                fn_obj, _ = contract.decode_function_input(tx.get('input', ''))
                method = fn_obj.fn_name
            except Exception:
                pass
            writer.writerow({
                'txHash': tx['hash'],
                'method': method,
                'caller': tx['from'],
                'isInternal': True,
            })

    print(f"Done. Results saved to {output_file}.")


if __name__ == '__main__':
    main()
```

## conversation2
