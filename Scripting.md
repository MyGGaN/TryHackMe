My writeup of [Scripting](https://tryhackme.com/room/scripting). A great room for some Python fun!

# Base64 (easy)
Download the b64.txt file for this task. It contains the flag but has been base64 encoded 50 times.  
Try do this in ~~both Bash and~~ Python!

```python
#!/usr/bin/python
import base64

# Read file
with open('b64.txt') as f:
    msg = f.read()

# Decode 50 times
for _ in range(50):
    msg = base64.b64decode(msg)

print("flag =", msg.decode('utf8'))
```

# Gotta Catch em All (medium)
Start the machine and Go to: http://<machines_ip>:3010 to start...

It'll host an additional ***webserver*** on different ports (it's a fixed sequence, starting at port 1337) for 4 seconds each.  
It'll respond with: operation, number, next port.  
Starting with 0, perform all the mathematical operations you get until you reach port 9765.
Expect add, minus, divide and multiply. The number given migt be a decimal.

Note: It might take a minute or so for the sequence to restart, please have patience.

```python
#!/usr/bin/python
import socket
import time

IP = "10.10.106.101"  # Update to your target machine
port = 1337  # This is the starting port
result = 0.0

def get(port, retry = False):
    if not retry: print(f"Connecting to port {port} ...")
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        try:
            s.connect((IP, port))
        except ConnectionRefusedError:
            time.sleep(2)  # Back-off 2 seconds
            return get(port, True)  # Retry
        else:
            # Send minimal HTTP request
            s.sendall(b"GET / HTTP/1.1\r\n\r\n")
            ret = []
            while True:
                r = s.recv(1024)
                if not r: break
                ret.append(r)
            resp = b''.join(ret).decode("utf-8")
            headers, body = resp.split("\r\n\r\n", 1)
            return body

while True:
    operation, number, port = get(port).split(" ")
    number = float(number)
    port = int(port)
    old = result
    if operation == "add":
        result += number
        print(f"{old:.2f} + {number} = {result:.2f}")
    elif operation == "minus":
        result -= number
        print(f"{old:.2f} - {number} = {result:.2f}")
    elif operation == "divide":
        result /= number
        print(f"{old:.2f} / {number} = {result:.2f}")
    elif operation == "multiply":
        result *= number
        print(f"{old:.2f} * {number} = {result:.2f}")

    if port == 9765: break  # End port

print(f"## Final result: {result:.2f}")
```


# Encrypted Server Chit Chat (hard)
The VM you have to connect to has a UDP server running on port 4000. Once connected to this UDP server, send a UDP message with the payload "hello" to receive more information.

> You've connected to the super secret server, send a packet with the payload ***ready*** to receive more information

When sending "ready" you'll get the key, iv and a sha256 checksum of the correct flag.  
Now you should send "final" and "final" to receive ciphertext and tag. Decode and verify against the checksum if it's the correct flag. Continue with final/final until you're done.

```python
#!/usr/bin/python
import socket
import hashlib
from cryptography.hazmat.primitives.ciphers import (Cipher, algorithms, modes)

IP = "10.10.145.31"  # Update to your target machine

def chitchat(s, msg):
    s.sendto(msg, (IP, 4000))
    data, address = s.recvfrom(1024)
    return data

with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
    # Read hello message
    print(chitchat(s, b"hello"))

    # Send "ready" and parse the response
    msg = chitchat(s, b"ready")
    key = msg[4:28]  # The message is fixed, I just eyeballed the offsets
    iv = msg[32:44]
    checksum = msg[104:136]
    print("key =", key)
    print("iv =", iv)
    print("checksum =", checksum.hex())

    while True:
        # Get cipher and tag
        ciphertext = chitchat(s, b"final")
        tag = chitchat(s, b"final")

        # Calculate checksum, stop on match
        d = Cipher(algorithms.AES(key), modes.GCM(iv, tag)).decryptor()
        flag = d.update(ciphertext) + d.finalize()
        if hashlib.sha256(flag).digest() == checksum:
            print("flag =", flag)
            break
```
