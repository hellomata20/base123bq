# base123bqimport time
from collections import defaultdict, deque
from web3 import Web3


RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()


WINDOW_BLOCKS = 10
LOW_ACTIVITY_LIMIT = 10
HIGH_ACTIVITY_LIMIT = 100

SCORE_THRESHOLD = 3

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Detecting pre-expansion wallets...\n")


    last_block = w3.eth.block_number


    # token activity history
    token_history = defaultdict(
        lambda: deque(maxlen=5)
    )


    # wallets seen during low activity phase
    early_wallets = defaultdict(set)


    # wallet score
    wallet_scores = defaultdict(int)


    while True:

        try:

            current_block = w3.eth.block_number


            if current_block >= last_block + WINDOW_BLOCKS:


                logs = w3.eth.get_logs({
                    "fromBlock": current_block - WINDOW_BLOCKS,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })


                activity = defaultdict(int)
                wallets = defaultdict(set)


                for log in logs:

                    token = log["address"]

                    from_addr = decode_address(
                        log["topics"][1]
                    )

                    to_addr = decode_address(
                        log["topics"][2]
                    )


                    activity[token] += 1


                    if from_addr != ZERO:
                        wallets[token].add(from_addr)

                    if to_addr != ZERO:
                        wallets[token].add(to_addr)



                for token, count in activity.items():

                    history = token_history[token]


                    # token was quiet before
                    if (
                        len(history) >= 2
                        and sum(history) / len(history)
                        <= LOW_ACTIVITY_LIMIT
                    ):

                        early_wallets[token].update(
                            wallets[token]
                        )


                    # token suddenly expanded
                    if count >= HIGH_ACTIVITY_LIMIT:

                        for wallet in early_wallets[token]:
                            wallet_scores[wallet] += 1



                    history.append(count)



                print(
                    f"\nBlocks "
                    f"{current_block - WINDOW_BLOCKS}"
                    f" -> {current_block}"
                )


                for wallet, score in wallet_scores.items():

                    if score >= SCORE_THRESHOLD:

                        print("🚀 Pre-Expansion Wallet")
                        print("Wallet:", wallet)
                        print("Score:", score)
                        print()



                last_block = current_block


            time.sleep(3)


        except Exception as e:

            print("Error:", e)
            time.sleep(5)



if __name__ == "__main__":
    main()
