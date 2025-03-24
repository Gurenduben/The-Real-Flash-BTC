import tkinter as tk
from tkinter import messagebox
from nbitcoin import *
import requests

def get_utxos_from_blockcypher(address, network="main"):
    """Retrieves UTXOs from BlockCypher API."""
    try:
        if network == "main":
            url = f"https://api.blockcypher.com/v1/btc/main/addrs/{address}/full?unspentOnly=true"
        elif network == "testnet":
            url = f"https://api.blockcypher.com/v1/btc/test3/addrs/{address}/full?unspentOnly=true"
        else:
            raise ValueError("Invalid network. Must be 'main' or 'testnet'.")

        response = requests.get(url)
        response.raise_for_status()  # Raise HTTPError for bad responses (4xx or 5xx)
        data = response.json()
        utxos = []
        for tx in data.get('txs', []):
            for output in tx.get('outputs', []):
                if output.get('spent_by') is None: #check if utxo is unspent.
                    utxos.append({
                        "txid": tx["hash"],
                        "vout": output["vout"],
                        "value": output["value"],
                        "script_pub_key": output["script"]
                    })
        return utxos
    except requests.exceptions.RequestException as e:
        raise Exception(f"BlockCypher API error: {e}")
    except KeyError as e:
        raise Exception(f"BlockCypher API data error: {e}")

def select_utxo(utxos, amount_sat):
    """Selects UTXOs to cover the transaction amount."""
    selected_utxos = []
    total_value = 0
    for utxo in utxos:
        selected_utxos.append(utxo)
        total_value += utxo["value"]
        if total_value >= amount_sat:
            return selected_utxos, total_value
    return None, 0 # Return None if not enough funds

def create_transaction(sender_address, recipient_address, amount_btc, network_str="main", private_key_wif=None):
    """Creates a Bitcoin transaction."""
    try:
        if network_str == "main":
            network = Network.Main
        elif network_str == "testnet":
            network = Network.TestNet
        else:
            raise ValueError("Invalid network. Must be 'main' or 'testnet'.")

        recipient = BitcoinAddress.from_base58(recipient_address, network)
        sender = BitcoinAddress.from_base58(sender_address, network)
        amount_sat = int(amount_btc * 100000000) #convert btc to satoshi.

        utxos = get_utxos_from_blockcypher(sender_address, network_str)
        selected_utxos, total_input_value = select_utxo(utxos, amount_sat)

        if selected_utxos is None:
            raise Exception("Insufficient funds.")

        transaction = Transaction()
        for utxo in selected_utxos:
            outpoint = OutPoint(uint256(utxo["txid"]), utxo["vout"])
            script_pub_key = Script.from_hex(utxo["script_pub_key"])
            coin = Coin(outpoint, TxOut(Money(utxo["value"], MoneyUnit.Satoshi), script_pub_key))
            transaction.inputs.append(TxIn(outpoint))

        transaction.outputs.append(TxOut(Money(amount_sat, MoneyUnit.Satoshi), recipient.script_pub_key))

        # Calculate change and add change output
        change_sat = total_input_value - amount_sat
        if change_sat > 0:
            transaction.outputs.append(TxOut(Money(change_sat, MoneyUnit.Satoshi), sender.script_pub_key))

        # Sign the transaction
        if private_key_wif:
            private_key = Key.from_wif(private_key_wif)
            transaction.sign(private_key, [Coin(OutPoint(uint256(utxo["txid"]), utxo["vout"]), TxOut(Money(utxo["value"], MoneyUnit.Satoshi), Script.from_hex(utxo['script_pub_key']))) for utxo in selected_utxos])

        return transaction.to_hex()

    except Exception as e:
        return f"Error: {e}"

def create_transaction_click():
    """Handles the button click event."""
    sender_address = sender_entry.get()
    recipient_address = recipient_entry.get()
    amount_str = amount_entry.get()
    private_key_wif = private_key_entry.get() #get private key from entry.
    network_select = network_variable.get()

    try:
        amount = float(amount_str)
        transaction_hex = create_transaction(sender_address, recipient_address, amount, network_select, private_key_wif)
        transaction_text.delete(1.0, tk.END)  # Clear previous text
        transaction_text.insert(tk.END, transaction_hex)

    except ValueError:
        messagebox.showerror("Error", "Invalid amount. Please enter a number.")
    except Exception as e:
        messagebox.showerror("Error", str(e))

# GUI setup
window = tk.Tk()
window.title("BTC Transaction Creator")

sender_label = tk.Label(window, text="Sender Address:")
sender_label.pack()
sender_entry = tk.Entry(window, width=40)
sender_entry.pack()

recipient_label = tk.Label(window, text="Recipient Address:")
recipient_label.pack()
recipient_entry = tk.Entry(window, width=40)
recipient_entry.pack()

amount_label = tk.Label(window, text="Amount (BTC):")
amount_label.pack()
amount_entry = tk.Entry(window, width=40)
amount_entry.pack()

private_key_label = tk.Label(window, text = "Private Key (WIF):")
private_key_label.pack()
private_key_entry = tk.Entry(window, width = 40, show="*")
private_key_entry.pack()

network_variable = tk.StringVar(window)
network_variable.set("main") #default is mainnet.
network_dropdown = tk.OptionMenu(window, network_variable, "main", "testnet")
network_dropdown.pack()

create_button = tk.Button(window, text="Create Transaction", command=create_transaction_click)
create_button.pack()

transaction_text = tk.Text(window, height=10, width=40)
transaction_text.pack()

window.mainloop()
