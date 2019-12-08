# pyAPDUFuzzer
A fuzzer for APDU-based smartcard interfaces

## Prerequisites
If you want to install `pyAPDUFuzzer`, you will need following dependencies:
- GCC
- Python 3
- Python 3 devel
- pip3
- SWIG
- PCSC lite
- PCSC lite devel
- American fuzzy lop

You will also need modified version of `python-afl`:
``` shell
pip3 install git+https://github.com/ph4r05/python-afl
```

### Installation script
If you are using `Fedora`, `Ubuntu` or `macOS` you can use our script to
install all required dependencies. Just run following line in your terminal:
``` shell
curl -fsSL https://github.com/petrs/pyAPDUFuzzer/raw/master/install-dependencies.sh | sh
```

## Installation
Finally, install latest version of `pyAPDUFuzzer` from GitHub:
``` shell
pip3 install git+https://github.com/petrs/pyAPDUFuzzer
```

Or use version from PyPI:
``` shell
pip3 install apdu-fuzzer
```

### Local development
For local development, clone this repository to your current working directory:
``` shell
git clone https://github.com/petrs/pyAPDUFuzzer.git
```

Then install `pyAPDUFuzzer` in **editable** mode:
``` shell
cd pyAPDUFuzzer && pip3 install -e . && cd ..
```

## Usage

### AFL fuzzing

```
AFL <-> Client <-> Server <-> Card

+----------------------------------+
|  AFL                             |
|  | |                             |                                   +-------------------+
|  | |          +----------------+ |         +------------------+      |                   |
|  | |   stdin  |                | |  socket |                  |      | +---+             |
|  | +----------|     Client     |------------      Server      -------- |-|-|    Card     |
|  |            |                | |         |                  |      | +---+             |
|  | +------+   +--------|-------+ |         +------------------+      |                   |
|  +-| SHM  |------------+         |                                   +-------------------+
|    +------+                      |
|                                  |
+----------------------------------+
```

(ascii by https://textik.com/)

Notes:

- Server is started first, connects to the card and listens on the socket for raw data to send to the card.
Does not process input data in any way.

- Server stores raw responses from the card to the data files.

- Server is able to reconnect to the card if something goes wrong.

- Client is started by AFL. AFL sends input data via STDIN, forking the client with each new fuzz input.
PCSC does not like forking with AFL this server/client architecture was required.

- Client is forked by the AFL after python is initialized. Socket can be opened either before fork or after the fork.
After fork is safer as each fuzz input has a new connection but a bit slower. Opening socket before fork also works
but special care needs to be done on broken pipe exception - reconnect logic is needed. This is not implemented now.

- Client post-processes input data generated by the AFL, e.g., generates length fields, can do TLV, etc.


Communication between server/client:

- Client sends `[0, buffer]`. Buffer is raw data structure to be sent to the card. `0` is the type / status

- Server responds with: `status 1B | SW1 1B | SW2 1B | timing 2B | data 0-NB`

```
+----+----+----+--------+------------------------+
|    |    |    |        |                        |
| 0  | SW | SW | timing |     response data      |
|    |  1 |  2 |        |                        |
+----+----+----+--------+------------------------+
```

Client then takes response from the socket, and uses modified [python-afl-ph4] to add trace to the shared memory segment
that is later analyzed by AFL to determine whether this fuzz input lead to different execution trace than the previous one.

Currently the trace bitmap is done in the following way:

```python
afl.trace_offset(hashxx(bytes([sw1, sw2])))
afl.trace_offset(hashxx(timing))
afl.trace_offset(hashxx(bytes(data)))
```

Fowler-Noll-Vo (FNV) hash function used in `afl.trace_buff` is not very good with respect to the zero buffers. The timing
was usually not affecting the bitmap so we switched to very fast hash function `hashxx` for the offset computation.

#### FNV collisions

FNV is inappropriate for this use case as it returns the same hash for all buffers of the following format:
`p | 0 | x` where `p` is a fixed prefix, `x` is random suffix.

```python
afl.hash32(bytes([0,1]))
afl.hash32(bytes([0,2]))
afl.hash32(bytes([0,255]))
afl.hash32(bytes([0,255,255,255]))
# 2166136261
```

#### Running

Start server sitting on the card:

```
python main_afl.py --server
```


Testing if the client works:

```
echo -n '0000' | ../venv/bin/python main_afl.py --client --output ydat.json --log ylog.txt
cat ylog.txt
```

AFL with forking & TCP communication with the server:

```
../venv/bin/py-afl-fuzz -m 500 -t 5000 -o result/ -i inputs/ -- ../venv/bin/python main_afl.py --client --output ydat.json --log ylog.txt
```

### Example usage with templates

Payload recovery for fixed command. Command header is fixed, payload is produced by AFL. Uses templating

```bash
PYTHON_AFL_PERSISTENT=1 ../venv/bin/py-afl-fuzz -m 200 -t 3000 -o result/ -i - -- \
    ../venv/bin/apdu-afl-fuzz --client --output ydat3.json --log yres3.json \
    --mask 00000000 --tpl 0be00100 --payload-len-b $((0x0c)) --payload-len-s $((0x1f))
```

- Uses fixed APDU prefix `0be00100` as mask is zero on those bytes.
- Generates payload of lengths `0x0c - 0x1f`.

