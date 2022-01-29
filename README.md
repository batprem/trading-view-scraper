# Set up functions


```python
import websocket
from websocket import create_connection

import json
import random 
import string
import re


from datetime import datetime
from time import sleep
from typing import List


class SESSION_ENUM:
    """
    Description:
        Session constants
    """
    WEBSOCKET: bool = False
    CHART: bool = True

def generate_sesssion(session) -> str:
    """
    Description:
        Get {session prefix}_{random string}
    params:
        session (bool)
            True: for websocket
            False: for chart
    returns:
        (str) web socket session
    """
    string_length = 12
    letters = string.ascii_lowercase
    random_string = "".join(
        random.choice(letters) for i in range(string_length)
    )
    prefix = "qs" if session else "cs"
    return f"qs_{random_string}"


def prepend_header(sentences: str)->str:
    """
    Description:
        format data into websocket message:
    params:
        sentence
            (str) contructed message
    returns:
        (str) An added prefix message
    example:
        ~m~54~m~{"m":"set_auth_token","p":["unauthorized_user_token"]}
    """
    return f"~m~{len(sentences)}~m~{sentences}"


def construct_message(function_name: str, parameters: List[str]) -> str:
    """
    params:
        function_name
            (str) Function to summit into websocket
        parameters:
            List[str]: list paramaters to input into the function
    returns:
        (str) a message as a JSON format join without space
    example:
        {"m":"set_auth_token","p":["unauthorized_user_token"]}
    """
    return json.dumps(
        {"m": function_name, "p": parameters}, separators=(",", ":")
    )

def create_message(function_name: str, parameters: List[str]) -> str:
    """
    Description:
        Integration of a created message function
    params:
        function_name:
            (str) Function to summit into websocket
        parameters:
            List[str]: list paramaters to input into the function
    returns:
        (str) message as websocket message format
    example:
        ~m~54~m~{"m":"set_auth_token","p":["unauthorized_user_token"]}
    """
    output = prepend_header(construct_message(function_name, parameters))
    return prepend_header(construct_message(function_name, parameters))
    
    
def send_message(
    ws: websocket._core.WebSocket,
    func: str, args: List[str]
) -> None:
    """
    Description:
        Send formatted message
    params:
        ws:
            (websocket._core.WebSocket) web socket sesssoin
        func:
            (str) Function to summit into websocket
        args:
            List[str]: list paramaters to input into the function
    """

    ws.send(create_message(func, args))

```

# Config


```python
headers = json.dumps({
    "Origin": "https://data.tradingview.com",
    "user-agent": "<prem.chotepanit@gmail.com> scrape for education purpose"
})

symbol = "BINANCE:BTCUSDT"
# time_frame_in_minute = str(1 * 60)
time_frame_in_minute = "1H"
nubmer_of_bars = 5

resolve_symbol = json.dumps({"symbol": symbol, "adjustment": "splits"})
```

# Send messages


```python
# Set up session

ws = create_connection(
    "wss://data.tradingview.com/socket.io/websocket", headers=headers
)

websocket_session = generate_sesssion(SESSION_ENUM.WEBSOCKET)
print(f"Web socket session generated: {websocket_session}")

chart_session = generate_sesssion(SESSION_ENUM.CHART)
print(f"Chart session generated: {chart_session}")


send_message(ws, "set_auth_token", ["unauthorized_user_token"])
send_message(ws, "chart_create_session", [chart_session, ""])
send_message(ws, "quote_create_session", [websocket_session])

send_message(
    ws, "quote_add_symbols", [websocket_session, symbol, {"flags": ["force_permission"]}]
)

send_message(ws, "resolve_symbol", [chart_session, "symbol_1", f"={resolve_symbol}"])
send_message(
    ws,
    "create_series",
    [chart_session, "btc_1", "btc_1", "symbol_1", time_frame_in_minute, nubmer_of_bars]
)

```

    Web socket session generated: qs_myldepzcukgh
    Chart session generated: qs_hpdlntzkbgpo


# Receive response


```python
a = ""

while True:
    try:
        results: str = ws.recv()

        pattern = re.compile("~m~\d+~m~~h~\d+$")

        if pattern.match(results):
            # Send heart beat to keep connection alive
            ws.recv()
            ws.send(results)

        for r in results.split("~m~"):
            try:
                r = json.loads(r)
                if not isinstance(r, dict):
                     continue
                message = r.get("m")
                if message == "timescale_update" or message == "du":
                    print(r)
            except json.JSONDecodeError:
                pass
    except KeyboardInterrupt:
        print("End")
        break
            
```

    {'m': 'timescale_update', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'node': 'hkg1-charts-20-study-engine-6@hkg1-compute-20', 's': [{'i': 0, 'v': [1643468400.0, 37620.04, 37735.67, 37268.44, 37566.82, 1881.672089999958]}, {'i': 1, 'v': [1643472000.0, 37567.98, 37603.57, 37317.37, 37538.19, 1102.7421800000066]}, {'i': 2, 'v': [1643475600.0, 37538.19, 37720.77, 37532.15, 37646.31, 784.4718399999997]}, {'i': 3, 'v': [1643479200.0, 37646.31, 37904.19, 37556.56, 37806.23, 772.066929999998]}, {'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37898.79, 195.86297000000047]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}, {'index': 0, 'zoffset': 0, 'changes': [1643468400.0, 1643472000.0, 1643475600.0, 1643479200.0, 1643482800.0], 'marks': [[10, 1643450400, 0], [30, 1643454000, 1], [33, 1643457600, 2], [30, 1643461200, 3], [30, 1643464800, 4]], 'index_diff': []}], 't': 1643483868, 't_ms': 1643483868109}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37898.88, 195.86836000000048]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37898.87, 195.88169000000048]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37898.87, 196.0062300000005]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37903.16, 196.10032000000052]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37899.25, 196.25233000000054]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.37, 196.25988000000055]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.37, 196.5769100000005]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.37, 196.6196900000005]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.35, 197.20781000000053]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.94, 197.3319400000006]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.94, 197.39360000000062]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.92, 197.40232000000063]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.92, 197.56556000000063]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.93, 197.6782900000006]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.92, 197.68998000000062]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37896.34, 198.5239000000006]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37898.13, 198.5733900000006]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.16, 198.84411000000057]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.17, 198.8840500000006]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.16, 198.90566000000058]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.16, 198.95024000000058]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.17, 198.95875000000058]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.16, 199.0243100000006]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.17, 199.04877000000062]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.16, 199.0824300000006]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.16, 199.12424000000064]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.17, 199.15238000000065]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.12, 199.22894000000062]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.13, 199.22994000000062]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37900.73, 199.52702000000065]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37899.01, 199.58028000000064]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37899.07, 199.59989000000064]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37913.1, 200.9164000000007]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37911.27, 200.9857500000007]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37911.23, 201.0370500000007]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37911.23, 201.06474000000074]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37911.23, 201.23727000000073]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37911.23, 201.2625900000007]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37904.18, 202.7842600000007]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37902.44, 203.09048000000072]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.11, 37781.49, 37902.44, 203.0926600000007]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37913.3, 37781.49, 37913.3, 203.56206000000074]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.06, 37781.49, 37914.06, 203.92409000000075]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.07, 37781.49, 37914.07, 204.01650000000075]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.08, 37781.49, 37914.08, 204.22305000000074]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.1, 37781.49, 37914.1, 204.22965000000073]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.13, 37781.49, 37914.13, 204.23119000000074]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.18, 37781.49, 37914.18, 204.31119000000075]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.25, 37781.49, 37914.25, 204.32457000000076]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.4, 37781.49, 37914.4, 204.32644000000076]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.54, 37781.49, 37914.54, 204.32673000000077]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37914.79, 37781.49, 37914.79, 204.32730000000078]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37915.56, 204.44438000000076]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37910.43, 204.80665000000084]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37908.53, 204.84421000000086]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37911.75, 205.91053000000088]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37910.44, 205.94516000000084]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37911.26, 206.21579000000082]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37911.26, 206.2760800000008]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37906.02, 206.3574800000008]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    {'m': 'du', 'p': ['qs_hpdlntzkbgpo', {'btc_1': {'s': [{'i': 4, 'v': [1643482800.0, 37806.24, 37915.56, 37781.49, 37913.24, 207.3805700000008]}], 'ns': {'d': '', 'indexes': []}, 't': 'btc_1', 'lbs': {'bar_close_time': 1643486400}}}]}
    End


# Reference

* https://github.com/rushic24/tradingview-scraper
