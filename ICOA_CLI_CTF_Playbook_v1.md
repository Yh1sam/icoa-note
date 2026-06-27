# ICOA CLI & CTF Playbook v1

> 目的：這不是 Linux manual。這是給 ICOA / PicoCTF / CTF 實戰用的 **80/20 指令手冊**。  
> 讀法：看到題目先按「題型 → 工具鏈 → 常用命令」走，不要死背全部參數。

---

## 0. 核心原則

### 0.1 先分類，再解題

看到檔案或服務時，第一步不是亂試工具，而是先分類：

| 你看到 | 第一反應 | 第一批工具 |
|---|---|---|
| `.jpg` / `.png` | 圖片 / Stego / metadata | `file`, `strings`, `exiftool`, `binwalk`, `pngcheck` |
| `.pdf` | PDF metadata / embedded objects | `file`, `strings`, `grep`, `exiftool`, `binwalk`, `pdftotext` |
| `.pcap` | 網路封包 | `tshark`, `tcpdump`, `strings` |
| ELF binary | Pwn / RE | `file`, `checksec`, `strings`, `objdump`, `nm`, `gdb` |
| 一大段 `010101` | binary bits | `fold`, `xxd`, Python bytes |
| 一大段 `aGV...==` | Base64 | `base64 -d` |
| `n, e, ciphertext` | RSA | `sympy`, `gmpy2`, `pycryptodome`, `openssl` |
| `g, p, A, B, shared` | Diffie-Hellman | Python `pow()`, `sympy` |
| `curl`, `/login`, cookie | Web / session | `curl`, `grep`, `jq`, `sqlite3` |
| `nc host port` | 互動服務 | `nc`, `pwntools`, `socat` |

### 0.2 萬能第一步

```bash
ls -lah
file *
```

如果是單一檔案：

```bash
file target
strings target | head
strings target | grep -i 'pico\|flag\|ctf\|password\|secret'
```

### 0.3 CTF 常見 pipeline 思維

```text
資料來源 → 清理 → 轉換 → 搜尋 → 驗證
```

例子：

```bash
cat logs.txt | tr -d '\n ' | base64 -d > out
file out
strings out | grep -i pico
```

### 0.4 不要把工具用錯

| 錯誤 | 原因 | 正確 |
|---|---|---|
| `objdump -d file.pdf` | `objdump` 是 binary/ELF 工具，不是 PDF 工具 | `strings`, `exiftool`, `binwalk` |
| `cat "base64string"` | `cat` 讀檔案，不是處理字串 | `echo "..." \| base64 -d` |
| `echo 123 | xxd -p` | 會把字串 `123` 變成 ASCII hex `313233` | `printf '%x\n' 123` |
| `python3 'print(...)'` | Python 以為那是檔名 | `python3 -c 'print(...)'` |
| `grep -o pico` | 只輸出 `pico` 這個匹配片段 | `grep -io 'picoCTF{[^}]*}' file` |
| 在 `nc` 裡輸入 `pip install` | `nc` 不是 shell，只是連遠端服務 | 先 `Ctrl+C` 回 shell |

---

## 1. 題型速查 Playbook

### 1.1 Unknown file

```bash
file target
ls -lh target
strings target | head -50
strings target | grep -i 'pico\|flag\|secret\|password'
binwalk target
xxd target | head
```

判斷：

- `PK` → zip/docx/jar/apk
- `%PDF` → PDF
- `JFIF` / `FF D8 FF` → JPEG
- `PNG` / `89 50 4E 47` → PNG
- `ELF` → Linux executable
- `SQLite` → database
- `pcap` → packet capture

### 1.2 PDF

```bash
file doc.pdf
strings doc.pdf | grep -i 'pico\|flag\|author\|title\|base64'
exiftool doc.pdf
binwalk doc.pdf
pdftotext doc.pdf -
```

常見思路：

- `/Author (...)`、`/Title (...)` 可能藏 Base64
- PDF 裡可能有 embedded file
- `strings` 常常比真正打開 PDF 更快

### 1.3 PNG / JPEG

```bash
file img.png
exiftool img.png
strings img.png | grep -i 'pico\|flag\|secret\|password'
binwalk img.png
pngcheck img.png
xxd img.png | head
```

PNG 進階：

```bash
pngcheck -v img.png
```

JPEG 進階：

```bash
exiftool img.jpg
strings img.jpg | less
```

### 1.4 PCAP

```bash
file capture.pcap
tshark -r capture.pcap | head
tshark -r capture.pcap -q -z io,phs
strings capture.pcap | grep -i 'pico\|flag'
tshark -r capture.pcap -Y http
tshark -r capture.pcap -Y dns -T fields -e dns.qry.name
```

Follow TCP stream：

```bash
tshark -r capture.pcap -T fields -e tcp.stream | sort -n | uniq
tshark -r capture.pcap -q -z follow,tcp,ascii,0
```

### 1.5 ELF / Pwn / RE

```bash
file vuln
checksec --file=vuln
strings vuln | grep -i 'flag\|win\|password\|payload'
nm vuln | grep ' win\| main\| vuln'
objdump -d vuln | grep '<win>'
gdb ./vuln
```

常見判斷：

- `NX enabled` → shellcode 不好用，想 ROP / ret2win
- `No PIE` → 函數地址固定
- `No canary found` → buffer overflow 更容易
- `not stripped` → `win`, `main`, `vuln` 名字還在

### 1.6 Logs

```bash
grep -i 'pico\|flag' server.log
grep -io 'picoCTF{[^}]*}' server.log
sort server.log | uniq
wc -l server.log
head server.log
tail server.log
```

如果看到 flag 分段：

```bash
grep -i 'FLAGPART' server.log
```

### 1.7 Base64 / Hex / Binary

Base64：

```bash
echo 'aGVsbG8=' | base64 -d
cat data.txt | tr -d '\n ' | base64 -d > out
```

Hex：

```bash
echo '68656c6c6f' | xxd -r -p
xxd file | head
xxd -p file | head
```

Binary bits → file：

```python
with open("digits.bin") as f:
    bits = f.read().strip()

data = bytes(int(bits[i:i+8], 2) for i in range(0, len(bits), 8))

with open("recovered_file", "wb") as f:
    f.write(data)
```

### 1.8 RSA

看到：

```text
n = ...
e = ...
ciphertext = ...
```

先問：

1. `e` 是否很小？例如 `3`, `5`
2. `n` 是否能 factor？
3. ciphertext 是整段還是逐字元列表？
4. 有沒有 private key / PEM？

Weak primes：

```python
from sympy import factorint

n = ...
e = ...
c = ...

factors = factorint(n)
p, q = list(factors.keys())
phi = (p-1)*(q-1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(m.to_bytes((m.bit_length()+7)//8, "big"))
```

Small exponent without modulus wrap：

```python
# c = m^e, m^e < n
m = round(c ** (1/e))
print(chr(m))
```

### 1.9 Diffie-Hellman + XOR

看到：

```text
g, p, A, B, a, b, shared
```

模板：

```python
p = ...
A = ...
b = ...
enc = bytes.fromhex("...")

shared = pow(A, b, p)
key = shared % 256

flag = bytes([x ^ key for x in enc])
print(flag.decode())
```

### 1.10 Web / Cookie / Session

```bash
curl -i http://site/
curl -i -L http://site/
curl -i -c cookie.txt http://site/
curl -i -b cookie.txt http://site/login
curl -i -c cookie.txt -d 'username=a&password=b' http://site/login
curl -s http://site/ | grep -o 'href="[^"]*"'
```

手動帶 cookie：

```bash
curl -b 'session=VALUE' http://site/
curl -H 'Cookie: session=VALUE' http://site/
```

### 1.11 nc 互動題

盲送：

```bash
python3 -c 'print("e"*1751)' | nc host port
```

等待後再送：

```bash
{ sleep 1; python3 -c 'print("e"*1751)'; sleep 1; } | nc host port
```

pwntools：

```python
from pwn import *

io = remote("host", 1234)
io.recvuntil(b"==> ")
io.sendline(b"answer")
print(io.recvall().decode(errors="ignore"))
```

---

# 2. Core Unix

## cat

**用途**：查看檔案內容。最適合短文字檔；不要 cat binary。

**什麼時候想到它**：看到 `.txt`, `.log`, source code 時第一反應。

**最常用寫法**：

```bash
cat file.txt
cat file.txt | grep pico
cat file1 file2 > combined.txt
cat > private.pem    # 手動貼內容，Ctrl+D 結束
```

**常見坑**：
- `cat vuln` 會把 ELF binary 的 raw bytes 噴到 terminal，通常沒意義。
- binary 應該用 `file`, `strings`, `xxd`, `objdump`。

## less

**用途**：分頁查看長檔案。

```bash
less file.txt
strings binary | less
/pico    # 在 less 裡搜尋 pico
q        # 離開
```

## head

**用途**：看檔案開頭。

```bash
head file.txt
head -n 50 file.txt
head -c 100 file.bin
```

## tail

**用途**：看檔案結尾。

```bash
tail file.txt
tail -n 50 file.txt
tail -f server.log
```

## grep

**用途**：搜尋文字。CTF 使用率極高。

```bash
grep 'flag' file.txt
grep -i 'pico' file.txt
grep -r 'pico' .
grep -o 'picoCTF{[^}]*}' file.txt
grep -E 'flag|pico|secret' file.txt
```

記住：
- `-i` 忽略大小寫
- `-r` 遞迴搜尋資料夾
- `-o` 只輸出匹配片段
- `-E` 使用 extended regex

## sed

**用途**：串流文字替換與抽取。

```bash
sed 's/old/new/g' file.txt
sed -n '10,20p' file.txt
cat file | sed 's/ //g'
```

## awk

**用途**：按欄位處理文字。

```bash
awk '{print $1}' file.txt
awk -F ':' '{print $2}' file.txt
grep FLAGPART log | awk '{print $NF}'
```

## find

**用途**：尋找檔案。

```bash
find . -name 'flag.txt'
find . -type f
find . -name '*.txt'
find / -name 'flag*' 2>/dev/null
find / -perm -4000 2>/dev/null
```

## sort / uniq / wc

```bash
sort file.txt
sort -n numbers.txt
sort file.txt | uniq
sort file.txt | uniq -c
wc -l file.txt
wc -c file.bin
```

## tr

**用途**：字元替換/刪除。

```bash
echo 'a_b_c' | tr '_' '-'
cat data.txt | tr -d '\n '
echo 'abc' | tr 'a-z' 'A-Z'
echo 'cvpbPGS' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

## diff / patch

```bash
diff file1 file2
diff -u old new > patch.diff
patch < fix.patch
patch -p1 < fix.patch
```

## chmod / chown / ln / cp / mv / mkdir / rm

```bash
chmod +x script.sh
chmod 600 private.key
sudo chown user:user file
ln -s target linkname
cp a b
cp -r dir dir2
mv out out.png
mkdir -p extracted/files
rm file
rm -r folder
```

---

# 3. Encoding / Data Processing

## base64

```bash
echo 'aGVsbG8=' | base64 -d
echo -n 'hello' | base64
cat data.txt | tr -d '\n ' | base64 -d > out
```

## xxd

```bash
xxd file | head
xxd -p file | head
echo '68656c6c6f' | xxd -r -p
xxd -r dump.hex out.bin
```

`xxd` 是 bytes ↔ hex，不是十進位 ↔ 十六進位。十進位轉 hex 用：

```bash
printf '0x%x\n' 255
```

## hexdump / od

```bash
hexdump -C file | head
hexdump -v -e '1/1 "%02x"' file
od -An -tx1 file | head
od -An -td1 file | head
od -c file | head
```

## jq

```bash
cat data.json | jq .
cat data.json | jq '.token'
curl -s api | jq '.data[] | .id'
```

## sqlite3

```bash
sqlite3 db.sqlite '.tables'
sqlite3 db.sqlite 'select * from users;'
sqlite3 db.sqlite '.schema users'
```

---

# 4. Archive / Compression

## unzip / zip

```bash
unzip file.zip
unzip -l file.zip
unzip -P password file.zip
cat part_* > full.zip && unzip full.zip
zip out.zip file.txt
zip -r out.zip folder
```

## tar / gzip / bzip2 / xz

```bash
tar -tf file.tar
tar -xf file.tar
tar -xzf file.tar.gz
tar -xJf file.tar.xz
gzip -d file.gz
bzip2 -d file.bz2
xz -d file.xz
```

---

# 5. File / Forensics

## file

```bash
file target
file *
file recovered_file
```

任何不明檔案第一步都是 `file`。

## strings

```bash
strings file
strings file | grep -i pico
strings file | less
strings -n 8 file
```

適用：binary、PDF、PNG、JPEG、pcap、log、firmware。

## binwalk

```bash
binwalk image.png
binwalk -e firmware.bin
binwalk --dd='.*' file
```

用途：掃描檔案內部是否嵌了 zip/png/pdf/gzip 等。

## exiftool

```bash
exiftool image.jpg
exiftool file.pdf
exiftool -Comment image.jpg
exiftool -a -u -g1 file
```

用途：metadata、comment、author、GPS、timestamp。

## pngcheck

```bash
pngcheck image.png
pngcheck -v image.png
```

用途：PNG chunk、corruption、hidden chunk。

## pdftotext

你貼的環境顯示 `pdftotext` 缺失，但如果正式環境或本地有它：

```bash
pdftotext file.pdf -
pdftotext file.pdf out.txt
pdftotext -layout file.pdf - | less
```

如果沒有：

```bash
strings file.pdf | less
strings file.pdf | grep -i 'pico\|flag\|author\|title'
exiftool file.pdf
```

## sleuthkit

```bash
mmls disk.img
fls -r disk.img
icat disk.img <inode> > recovered
fsstat disk.img
```

用途：磁碟映像、刪除檔恢復。

## volatility3

```bash
vol -f mem.raw windows.info
vol -f mem.raw windows.pslist
vol -f mem.raw windows.cmdline
vol -f mem.raw windows.filescan
```

用途：memory dump forensic。

---

# 6. Networking / PCAP / Remote

## curl

```bash
curl -i http://site/
curl -s http://site/
curl -L http://site/
curl -c cookie.txt -b cookie.txt http://site/
curl -d 'username=a&password=b' http://site/login
curl -H 'Cookie: session=abc' http://site/
```

重要參數：
- `-i` 顯示 header
- `-s` silent
- `-L` follow redirect
- `-c` save cookie
- `-b` use cookie
- `-d` POST data
- `-H` custom header

## wget

```bash
wget URL
wget -O file URL
wget -r URL
```

## nc

```bash
nc host port
nc -vz host port
printf 'hello\n' | nc host port
nc -l 4444
```

## socat

```bash
socat - TCP:host:port
socat TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/sh
socat file:`tty`,raw,echo=0 tcp:host:port
```

## nmap

```bash
nmap host
nmap -sV host
nmap -p- host
nmap -p 80,443 -A host
```

## ssh

```bash
ssh user@host
ssh -p 2222 user@host
scp -P 2222 file user@host:/tmp/
```

## dig / whois / ping / traceroute

```bash
dig example.com
dig TXT example.com
dig @8.8.8.8 example.com
whois example.com
ping -c 4 host
traceroute host
```

## tcpdump

```bash
tcpdump -i any
tcpdump -r capture.pcap
tcpdump -nn -r capture.pcap
tcpdump -A -r capture.pcap 'tcp port 80'
```

## tshark

```bash
tshark -r capture.pcap | head
tshark -r capture.pcap -q -z io,phs
tshark -r capture.pcap -Y http
tshark -r capture.pcap -Y 'http.request' -T fields -e http.host -e http.request.uri
tshark -r capture.pcap -Y dns -T fields -e dns.qry.name
tshark -r capture.pcap -T fields -e tcp.stream | sort -n | uniq
tshark -r capture.pcap -q -z follow,tcp,ascii,0
```

## scapy

```python
from scapy.all import *
pkts = rdpcap('capture.pcap')
pkts.summary()
for p in pkts:
    print(p.summary())
```

## paramiko / pyserial

```python
# paramiko: Python SSH automation
import paramiko

# pyserial: serial challenge
import serial
# serial.Serial('/dev/ttyUSB0', 9600)
```

---

# 7. Web / API / Database

## sqlmap

```bash
sqlmap -u 'http://site/item?id=1'
sqlmap -u URL --dbs
sqlmap -u URL -D db --tables
sqlmap -u URL -D db -T users --dump
```

只用於授權 CTF target，不要掃外部真站。

## requests

```python
import requests
r = requests.get(url)
r = requests.post(url, data={'username':'a','password':'b'})
r = requests.get(url, cookies={'session':'abc'})
print(r.text)
```

## beautifulsoup4

```python
from bs4 import BeautifulSoup
soup = BeautifulSoup(html, 'html.parser')
for a in soup.find_all('a'):
    print(a.get('href'))
```

## flask

```python
from flask import Flask, request
app = Flask(__name__)

@app.route('/')
def index():
    return 'ok'

app.run(host='0.0.0.0', port=8000)
```

用途：本地 mock/webhook；正式賽是否需要看題目。

---

# 8. Crypto / Password

## openssl

```bash
openssl pkeyutl -decrypt -inkey private.pem -in flag.enc
openssl rsa -in private.pem -text -noout
openssl dgst -sha256 file
openssl enc -d -aes-256-cbc -in enc.bin -out dec.bin -k password
openssl x509 -in cert.pem -text -noout
```

`openssl pkeyutl -decrypt -inkey private.pem -in flag.enc` 人話：  
用 `private.pem` 這把私鑰，解開 `flag.enc` 這個加密檔。

## gpg

```bash
gpg --decrypt file.gpg
gpg --import key.asc
gpg --list-keys
```

## john

```bash
john hash.txt
john --wordlist=rockyou.txt hash.txt
john --show hash.txt
zip2john file.zip > hash.txt
```

## hashcat

```bash
hashcat -m 0 hash.txt wordlist.txt
hashcat --show -m 0 hash.txt
hashcat --help | grep -i sha
```

## sympy

```python
from sympy import factorint, mod_inverse
factorint(n)
d = mod_inverse(e, phi)
```

## gmpy2

```python
import gmpy2
root, exact = gmpy2.iroot(c, 3)
d = gmpy2.invert(e, phi)
```

## pycryptodome

```python
from Crypto.Util.number import long_to_bytes, bytes_to_long
from Crypto.Cipher import AES
from Crypto.PublicKey import RSA
```

## cryptography

```python
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import padding

private_key = serialization.load_pem_private_key(data, password=None)
plaintext = private_key.decrypt(ciphertext, padding.PKCS1v15())
```

---

# 9. Binary / Pwn / RE

## checksec

```bash
checksec --file=vuln
pwn checksec vuln
```

看：Canary / NX / PIE / RELRO。

## objdump

```bash
objdump -d vuln | less
objdump -d vuln | grep '<win>'
objdump -t vuln | grep win
objdump -s -j .rodata vuln
```

## nm

```bash
nm vuln
nm vuln | grep ' win\| main\| vuln'
nm -D vuln
```

## gdb

```bash
gdb ./vuln
break main
run
disassemble main
x/20gx $rsp
info registers
continue
```

## pwndbg / GEF

```bash
gdb ./vuln
checksec
cyclic 100
cyclic -l 0x6161616c
telescope $rsp
```

## pwntools

```python
from pwn import *

io = remote('host', 1234)
io.recvuntil(b'> ')
io.sendline(b'payload')
io.interactive()

payload = b'A'*40 + p64(0x401176)
```

## ROPgadget / ropper

```bash
ROPgadget --binary vuln
ROPgadget --binary vuln | grep 'pop rdi'
ROPgadget --binary vuln --only 'ret'
ropper --file vuln
ropper --file vuln --search 'pop rdi; ret'
```

## radare2 / rabin2

```bash
r2 -A vuln
aaa
afl
pdf @ main
iz
q

rabin2 -I vuln
rabin2 -s vuln
rabin2 -z vuln
```

## upx

```bash
upx -d packed_binary
upx -l binary
```

## angr / z3 / capstone / pefile / uncompyle6

```python
# z3
from z3 import *
x = BitVec('x', 32)
s = Solver()
s.add(x + 1 == 5)
print(s.check(), s.model())

# capstone
from capstone import *

# angr
import angr
proj = angr.Project('./vuln', auto_load_libs=False)
```

```bash
uncompyle6 file.pyc
uncompyle6 -o out file.pyc
```

---

# 10. Build / Runtime / Editors

## gcc / g++

```bash
gcc vuln.c -o vuln
gcc vuln.c -o vuln -fno-stack-protector -no-pie
gcc -g vuln.c -o vuln_dbg
g++ -std=c++17 main.cpp -o main
```

## make / cmake / nasm / as / ld / pkg-config

```bash
make
make clean
make -j4
cmake -S . -B build
cmake --build build
nasm -f elf64 shell.asm -o shell.o
ld shell.o -o shell
pkg-config --cflags --libs openssl
```

## node

```bash
node solve.js
node -e 'console.log(Buffer.from("aGVsbG8=","base64").toString())'
```

不要依賴比賽中 `npm install`，除非規則明確允許。

## python3 / pip3 / ipython

```bash
python3 solve.py
python3 -c 'print(hex(255))'
python3 -m http.server 8000
pip3 list
pip3 show pwntools
ipython
```

## vim / nano

```bash
vim solve.py
# i insert, Esc, :wq save quit, :q! quit no save

nano solve.py
# Ctrl+O save, Ctrl+X exit
```

## tmux / screen

```bash
tmux
# Ctrl+b c new window
# Ctrl+b n next
# Ctrl+b d detach
tmux attach

screen
# Ctrl+a c new window
# Ctrl+a d detach
screen -r
```

---

# 11. Git / Docker

## git

```bash
git status
git log --all --oneline
git show HEAD
git branch -a
git reflog
git remote -v
git commit --amend --author="root <root@picoctf>" --no-edit
git push --force
```

Git CTF 思路：

```text
history? branch? tag? reflog? author metadata? remote hook?
```

## docker

```bash
docker ps
docker images
docker run --rm -it image sh
docker build -t test .
```

正式賽可用性要看 sandbox 權限。

---

# 12. Python Data / Image / Forensics Libraries

## Pillow

```python
from PIL import Image
img = Image.open('image.png')
print(img.size, img.mode)
print(img.getpixel((0,0)))
img.save('out.png')
```

LSB 基礎：

```python
from PIL import Image
img = Image.open('image.png').convert('RGB')
bits = []
for r,g,b in img.getdata():
    bits.append(str(r & 1))
print(''.join(bits[:200]))
```

## numpy

```python
import numpy as np
from PIL import Image
arr = np.array(Image.open('img.png'))
print(arr.shape)
```

## python-magic

```python
import magic
print(magic.from_file('file'))
```

## yara-python

```python
import yara
rules = yara.compile(source='rule dummy { condition: true }')
print(rules.match('file'))
```

---

# 13. Copy-Paste One-liners

## 找 flag

```bash
grep -RIn 'picoCTF\|ICOA\|flag' .
grep -Rio 'picoCTF{[^}]*}' .
strings file | grep -io 'picoCTF{[^}]*}'
```

## Base64 decode

```bash
echo 'aGVsbG8=' | base64 -d
cat data.txt | tr -d '\n ' | base64 -d > out
```

## Hex decode

```bash
echo '68656c6c6f' | xxd -r -p
```

## Decimal ↔ Hex

```bash
printf '0x%x\n' 106950354727591
printf '%d\n' 0x614551e6f2a7
```

## Binary bits → file

```bash
python3 -c 's=open("digits.bin").read().strip(); data=bytes(int(s[i:i+8],2) for i in range(0,len(s),8)); open("recovered_file","wb").write(data)'
file recovered_file
```

## XOR single-byte brute force

```python
data = bytes.fromhex("...")
for k in range(256):
    out = bytes([b ^ k for b in data])
    if b"pico" in out.lower() or b"ctf" in out.lower():
        print(k, out)
```

## Diffie-Hellman leaked secret + XOR

```python
p = ...
A = ...
b = ...
enc = bytes.fromhex("...")

shared = pow(A, b, p)
key = shared % 256
print(bytes([x ^ key for x in enc]).decode())
```

## RSA weak n

```python
from sympy import factorint

n = ...
e = ...
c = ...

p, q = list(factorint(n).keys())
phi = (p-1)*(q-1)
d = pow(e, -1, phi)
m = pow(c, d, n)
print(m.to_bytes((m.bit_length()+7)//8, "big"))
```

## RSA small exponent

```python
import gmpy2

c = ...
e = 5

m, exact = gmpy2.iroot(c, e)
print(m, exact)
print(int(m).to_bytes((int(m).bit_length()+7)//8, "big"))
```

## pwntools remote

```python
from pwn import *

io = remote("host", 1234)
io.recvuntil(b"> ")
io.sendline(b"payload")
io.interactive()
```

## PIE main leak → win address

```bash
objdump -d vuln | grep '<main>'
objdump -d vuln | grep '<win>'
printf '0x%x\n' $((REMOTE_MAIN - MAIN_OFFSET + WIN_OFFSET))
```

Example：

```bash
printf '0x%x\n' $((0x614551e6f33d - 0x133d + 0x12a7))
```

## pcap quick view

```bash
tshark -r capture.pcap -q -z io,phs
strings capture.pcap | grep -i 'pico\|flag'
tshark -r capture.pcap -q -z follow,tcp,ascii,0
```

## PDF metadata

```bash
strings file.pdf | grep -i 'author\|title\|pico\|flag'
exiftool file.pdf
```

## PNG/JPEG triage

```bash
file img.png
exiftool img.png
strings img.png | grep -i 'pico\|flag\|secret\|password'
binwalk img.png
pngcheck img.png
```

## Git CTF

```bash
git log --all --oneline
git show HEAD
git branch -a
git reflog
git remote -v
git commit --amend --author="root <root@picoctf>" --no-edit
```

---

# 14. 最重要的 30 個工具排序

如果時間很短，優先熟這些：

1. `file`
2. `strings`
3. `grep`
4. `cat`
5. `less`
6. `head`
7. `tail`
8. `xxd`
9. `base64`
10. `tr`
11. `sort`
12. `uniq`
13. `wc`
14. `find`
15. `curl`
16. `nc`
17. `tshark`
18. `binwalk`
19. `exiftool`
20. `pngcheck`
21. `openssl`
22. `python3`
23. `pwntools`
24. `sympy`
25. `gmpy2`
26. `objdump`
27. `nm`
28. `gdb`
29. `checksec`
30. `git`

---

# 15. 錯誤對照表

| 你遇過的錯 | 本質 | 正確理解 |
|---|---|---|
| `cat vuln` 亂碼 | binary 不是文字 | 用 `strings`, `objdump`, `file` |
| `objdump pdf` 失敗 | PDF 不是 executable | PDF 用 `strings`, `exiftool` |
| `xxd` 轉 decimal 失敗 | `xxd` 處理 bytes，不是數字進制 | decimal→hex 用 `printf '%x\n'` |
| `grep -o pico` 只出 pico | `-o` 只輸出匹配片段 | 用完整 regex |
| Python `decode()` 太早 | XOR 要對 bytes 做 | 解密完再 decode |
| zsh `pipe dquote>` | 引號沒關或用了智能引號 | 用普通英文引號 |
| `a = 1` in shell | shell 不是 Python | 進 `python3` 或寫 `.py` |
| `pip install` in nc | nc 是遠端互動，不是 shell | Ctrl+C 回本地 shell |

---

# 16. 最後的實戰原則

1. **先 `file`，再選工具。**
2. **看到文字就 `grep`；看到 binary 就 `strings`。**
3. **看到 hex/base64/binary bits，先轉 bytes。**
4. **看到 RSA 就找 `n,e,c`，問：能不能 factor？e 小不小？有沒有 key？**
5. **看到 DH 就找 `g,p,A,B,a,b`，問：哪個 secret 洩漏？**
6. **看到 pwn 就先 `checksec`，再看 `win()`、offset、canary、PIE。**
7. **看到 pcap 就 `tshark -z io,phs`，再 follow stream。**
8. **不要背全部參數；記住每類題的第一反應。**
