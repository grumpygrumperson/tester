#API Setup

##Prerequisites
[GitBash](https://git-scm.com/install/)<br>
[Node.js](https://nodejs.org/en)<br>
[wscat](https://github.com/websockets/wscat)

<hr>

##API Docs Authentication 
1. Follow GRVT's API Setup Guide [(Step 5 - Step 6)](https://api-docs.grvt.io/api_setup/)<br>

2. Create Shellscript using GitBash
```bash
cat > ~/grvt_api.sh << 'EOF'
GRVT_API_KEY="<insert_key_here>"
GRVT_SUB_ACCOUNT_ID="<insert_sub_account_id_here>"
GRVT_AUTH_ENDPOINT="https://edge.grvt.io/auth/api_key/login"

echo $GRVT_API_KEY
echo $GRVT_SUB_ACCOUNT_ID
echo $GRVT_AUTH_ENDPOINT

RESPONSE=$(
    curl $GRVT_AUTH_ENDPOINT \
        -H 'Content-Type: application/json' \
        -H 'Cookie: rm=true;' \
        -d '{"api_key": "'$GRVT_API_KEY'"}' \
        -s -i
)

GRVT_COOKIE=$(echo "$RESPONSE" | grep -i 'set-cookie:' | grep -o 'gravity=[^;]*')
GRVT_ACCOUNT_ID=$(echo "$RESPONSE" | grep 'x-grvt-account-id:' | awk '{print $2}' | tr -d '\r')

if [ -n "$GRVT_COOKIE" ] && [ -n "$GRVT_ACCOUNT_ID" ]; then
    echo "Authentication successful!"
    echo "Cookie: $GRVT_COOKIE"
    echo "Account ID: $GRVT_ACCOUNT_ID"
else
    echo "Authentication failed. Check your API key and endpoint."
fi
EOF
```
3. Run Shellscript
```bash
source ~/grvt_api.sh
```

<br> 
For more information, have a look at GRVT's [docs](https://api-docs.grvt.io/) or [contact support](https://set-global.com/contact)  
<hr>

##(Optional) Install GRVT's Python SDK
GRVT has a [Python SDK](https://github.com/gravity-technologies/grvt-pysdk) available for usage.
```py
pip install grvt-pysdk
```

If you are using GRVT's SDK, you will need to create another Shellscript containing your keys
```bash
cat > ~/grvt_env.sh << 'EOF'
export GRVT_PRIVATE_KEY="your_production_private_key"
export GRVT_API_KEY="your_production_api_key_here"
export GRVT_TRADING_ACCOUNT_ID="your_production_trading_account_id_here"
export GRVT_ENV="prod"
export GRVT_AUTH_ENDPOINT="https://edge.grvt.io/auth/api_key/login"
export GRVT_END_POINT_VERSION="v1"
export GRVT_WS_STREAM_VERSION="v1"
EOF
```

??? Info "Example of a Create Order Script using GRVT's Python SDK"
    ```py title="grvt_create_order.py" linenums="1"
    import logging
    import os
    import random
    import time
    from pysdk import grvt_raw_types
    from pysdk.grvt_raw_base import GrvtApiConfig
    from pysdk.grvt_raw_env import GrvtEnv
    from pysdk.grvt_raw_signing import sign_order
    from pysdk.grvt_raw_sync import GrvtRawSync

    def get_config() -> GrvtApiConfig:
        logging.basicConfig()
        logger = logging.getLogger()
        logger.setLevel(logging.INFO)  # Changed from DEBUG to INFO for production
        conf = GrvtApiConfig(
            env=GrvtEnv(os.getenv("GRVT_ENV", "prod")),
            trading_account_id=os.getenv("GRVT_TRADING_ACCOUNT_ID"),
            private_key=os.getenv("GRVT_PRIVATE_KEY"),
            api_key=os.getenv("GRVT_API_KEY"),
            logger=logger,
        )
        return conf

    def get_order(api, instruments):
        if (
            api.config.trading_account_id is None
            or api.config.private_key is None
            or api.config.api_key is None
        ):
            print("Missing credentials - check your grvt_env.sh")
            return None
        order = grvt_raw_types.Order(
            sub_account_id=str(api.config.trading_account_id),
            time_in_force=grvt_raw_types.TimeInForce.GOOD_TILL_TIME,
            legs=[
                grvt_raw_types.OrderLeg(
                    instrument="BTC_USDT_Perp",
                    size="0.001",          # Change this to your desired size
                    limit_price="50000",   # Change this to your desired price
                    is_buying_asset=True,  # Change to False to sell
                )
            ],
            signature=grvt_raw_types.Signature(
                signer="",
                r="",
                s="",
                v=0,
                expiration=str(time.time_ns() + 20 * 24 * 60 * 60 * 1_000_000_000),
                nonce=random.randint(0, 2**32 - 1),
            ),
            metadata=grvt_raw_types.OrderMetadata(
                client_order_id=str(random.randint(0, 2**32 - 1)),
            ),
        )
        return sign_order(order, api.config, api.account, instruments)

    def create_order():
        print("Connecting to GRVT production...")
        api = GrvtRawSync(config=get_config())

        print("Fetching instruments...")
        inst_resp = api.get_all_instruments_v1(
            grvt_raw_types.ApiGetAllInstrumentsRequest(is_active=True)
        )
        if isinstance(inst_resp, Exception):
            raise ValueError(f"Error fetching instruments: {inst_resp}")

        instruments = {inst.instrument: inst for inst in inst_resp.result}
        print(f"Fetched {len(instruments)} instruments")

        print("Creating and signing order...")
        order = get_order(api, instruments)
        if order is None:
            return

        print("Submitting order...")
        resp = api.create_order_v1(grvt_raw_types.ApiCreateOrderRequest(order=order))
        if isinstance(resp, Exception):
            raise ValueError(f"Error creating order: {resp}")

        if hasattr(resp, 'code') and resp.code != 0:
            print(f"Order failed: {resp.message}")
        else:
            print("Order created successfully!")
            print(resp)

    create_order()
    ```
<br>
For more information on APIs and Websockets, have a look at GRVT's [docs](https://api-docs.grvt.io/) 