
# Allora Node Reinstallation Script

This script is designed to reinstall the Allora Node assuming that a previous version of the node has already been installed and the `basic-coin-prediction-node` directory is located at `/root/basic-coin-prediction-node`.

## Usage

To use this script, simply paste the following command into your terminal as the `root` user and press `Enter`:

```bash
cat << 'EOF' > $HOME/allora_reinstall.sh
#!/bin/bash

if [ "$EUID" -ne 0 ]; then
  echo "Please run this script as root."
  exit 1
fi

apt install sed

cd $HOME/basic-coin-prediction-node
docker compose down -v
docker container prune

cd $HOME && rm -rf basic-coin-prediction-node

git clone https://github.com/allora-network/basic-coin-prediction-node
cd basic-coin-prediction-node

rm -rf config.json

read -p "Enter WALLET_SEED_PHRASE: " WALLET_SEED_PHRASE
echo

if [ -z "$WALLET_SEED_PHRASE" ]; then
  print_with_color $RED "WALLET_SEED_PHRASE cannot be empty."
  exit 1
fi

cat <<CONFIG_EOF > config.json
{
    "wallet": {
        "addressKeyName": "testkey",
        "addressRestoreMnemonic": "$WALLET_SEED_PHRASE",
        "alloraHomeDir": "",
        "gas": "1000000",
        "gasAdjustment": 1.0,
        "nodeRpc": "https://sentries-rpc.testnet-1.testnet.allora.network/",
        "maxRetries": 1,
        "delay": 1,
        "submitTx": false
    },
    "worker": [
        {
            "topicId": 1,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ETH"
            }
        },
        {
            "topicId": 2,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ETH"
            }
        },
        {
            "topicId": 7,
            "inferenceEntrypointName": "api-worker-reputer",
            "loopSeconds": 5,
            "parameters": {
                "InferenceEndpoint": "http://inference:8000/inference/{Token}",
                "Token": "ETH"
            }
        }
    ]
}
CONFIG_EOF

echo "config.json has been created successfully."

chmod +x init.config
./init.config

sed -i 's/intervals = \["1d"\]/intervals = \["10m", "20m", "1h", "1d"\]/' model.py

echo "model.py has been modified successfully."

docker compose up -d --build
docker ps

mkdir -p $HOME/logs
nohup docker logs -f worker > $HOME/logs/worker_allora_node.log 2>&1 &
nohup docker logs -f updater-basic-eth-pred > $HOME/logs/updater_allora_node.log 2>&1 &
nohup docker logs -f inference-basic-eth-pred > $HOME/logs/inference_allora_node.log 2>&1 &

echo "You can find logs in $HOME/logs"
EOF

chmod +x $HOME/allora_reinstall.sh
bash $HOME/allora_reinstall.sh
```

## Script Description

The script will perform the following actions:

    1. Ensure the script is run as the root user.
    2. Install sed if it's not already installed.
    3. Navigate to the `basic-coin-prediction-node` directory.
    4. Stop and remove all Docker containers associated with the node.
    5. Remove the existing `basic-coin-prediction-node` directory and clone a fresh copy from the repository.
    6. Remove any existing `config.json` file.
    7. Prompt the user to enter their `WALLET_SEED_PHRASE` and create a new `config.json` file using the provided phrase.
    8. Modify the `model.py` file to include multiple intervals.
    9. Build and start the Docker containers using `docker compose`.
    10. Start logging the output of the worker, updater, and inference services to respective log files.
    11. Notify the user that logs can be found in the `$HOME/logs` directory.

## Note
This script assumes that you have already set up your Allora node previously and that the basic-coin-prediction-node directory exists in your home directory. If this is not the case, please follow the initial setup instructions provided by Allora Network before using this script.
