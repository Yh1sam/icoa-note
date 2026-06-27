# ICOA CLI & CTF Playbook v3 — Terminal Friendly

> 目的：這不是 Linux manual。這是給 ICOA / PicoCTF / CTF 實戰用的 **80/20 指令手冊**。  
> 讀法：看到題目先按「題型 → 工具鏈 → 常用命令」走，不要死背全部參數。
> v3 改動：移除 Markdown 表格，改成 terminal-friendly list，方便用 `less`, `grep`, `sed`, `vim -R` 閱讀。

---

## 0.0 CLI 閱讀方式

這版避免使用 Markdown table，因為 terminal 裡表格容易錯位。建議這樣查：

```bash
grep -n "RSA" ICOA_CLI_CTF_Playbook_v3_terminal_friendly.md
less +1390 ICOA_CLI_CTF_Playbook_v3_terminal_friendly.md
sed -n '1390,1450p' ICOA_CLI_CTF_Playbook_v3_terminal_friendly.md
vim -R ICOA_CLI_CTF_Playbook_v3_terminal_friendly.md
```

閱讀時只記三個動作：

```text
grep -n keyword file   # 找位置
less +line file        # 跳到位置閱讀
sed -n 'a,bp' file     # 印出指定行段
```

---

## 0. 核心原則

### 0.1 先分類，再解題

看到檔案或服務時，第一步不是亂試工具，而是先分類：


> 終端友好列表： 你看到 / 第一反應 / 第一批工具

- **你看到:** `.jpg` / `.png`
  - **第一反應:** 圖片 / Stego / metadata
  - **第一批工具:** `file`, `strings`, `exiftool`, `binwalk`, `pngcheck`

- **你看到:** `.pdf`
  - **第一反應:** PDF metadata / embedded objects
  - **第一批工具:** `file`, `strings`, `grep`, `exiftool`, `binwalk`, `pdftotext`

- **你看到:** `.pcap`
  - **第一反應:** 網路封包
  - **第一批工具:** `tshark`, `tcpdump`, `strings`

- **你看到:** ELF binary
  - **第一反應:** Pwn / RE
  - **第一批工具:** `file`, `checksec`, `strings`, `objdump`, `nm`, `gdb`

- **你看到:** 一大段 `010101`
  - **第一反應:** binary bits
  - **第一批工具:** `fold`, `xxd`, Python bytes

- **你看到:** 一大段 `aGV...==`
  - **第一反應:** Base64
  - **第一批工具:** `base64 -d`

- **你看到:** `n, e, ciphertext`
  - **第一反應:** RSA
  - **第一批工具:** `sympy`, `gmpy2`, `pycryptodome`, `openssl`

- **你看到:** `g, p, A, B, shared`
  - **第一反應:** Diffie-Hellman
  - **第一批工具:** Python `pow()`, `sympy`

- **你看到:** `curl`, `/login`, cookie
  - **第一反應:** Web / session
  - **第一批工具:** `curl`, `grep`, `jq`, `sqlite3`

- **你看到:** `nc host port`
  - **第一反應:** 互動服務
  - **第一批工具:** `nc`, `pwntools`, `socat`


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


> 終端友好列表： 錯誤 / 原因 / 正確

- **錯誤:** `objdump -d file.pdf`
  - **原因:** `objdump` 是 binary/ELF 工具，不是 PDF 工具
  - **正確:** `strings`, `exiftool`, `binwalk`

- **錯誤:** `cat "base64string"`
  - **原因:** `cat` 讀檔案，不是處理字串
  - **正確:** `echo "..." | base64 -d`

- **錯誤:** `echo 123 | xxd -p`
  - **原因:** 會把字串 `123` 變成 ASCII hex `313233`
  - **正確:** `printf '%x\n' 123`

- **錯誤:** `python3 'print(...)'`
  - **原因:** Python 以為那是檔名
  - **正確:** `python3 -c 'print(...)'`

- **錯誤:** `grep -o pico`
  - **原因:** 只輸出 `pico` 這個匹配片段
  - **正確:** `grep -io 'picoCTF{[^}]*}' file`

- **錯誤:** 在 `nc` 裡輸入 `pip install`
  - **原因:** `nc` 不是 shell，只是連遠端服務
  - **正確:** 先 `Ctrl+C` 回 shell


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


> 終端友好列表： 你遇過的錯 / 本質 / 正確理解

- **你遇過的錯:** `cat vuln` 亂碼
  - **本質:** binary 不是文字
  - **正確理解:** 用 `strings`, `objdump`, `file`

- **你遇過的錯:** `objdump pdf` 失敗
  - **本質:** PDF 不是 executable
  - **正確理解:** PDF 用 `strings`, `exiftool`

- **你遇過的錯:** `xxd` 轉 decimal 失敗
  - **本質:** `xxd` 處理 bytes，不是數字進制
  - **正確理解:** decimal→hex 用 `printf '%x\n'`

- **你遇過的錯:** `grep -o pico` 只出 pico
  - **本質:** `-o` 只輸出匹配片段
  - **正確理解:** 用完整 regex

- **你遇過的錯:** Python `decode()` 太早
  - **本質:** XOR 要對 bytes 做
  - **正確理解:** 解密完再 decode

- **你遇過的錯:** zsh `pipe dquote>`
  - **本質:** 引號沒關或用了智能引號
  - **正確理解:** 用普通英文引號

- **你遇過的錯:** `a = 1` in shell
  - **本質:** shell 不是 Python
  - **正確理解:** 進 `python3` 或寫 `.py`

- **你遇過的錯:** `pip install` in nc
  - **本質:** nc 是遠端互動，不是 shell
  - **正確理解:** Ctrl+C 回本地 shell


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

---

# Appendix v2 — RSA 經典情形、檔案 Header、URL / Encoding 轉換

> 這一章是 v2 補充。目的不是把密碼學或檔案格式變成教科書，而是把 ICOA / CTF 最常見的「看到什麼 → 想到什麼 → 下什麼命令」整理成可直接使用的模板。

---

## A. RSA 經典情形 Playbook

### A.0 看到 RSA 先問的 6 個問題

拿到：

```text
n = ...
e = ...
ciphertext = ...
```

先不要急著寫解密程式，先分類：


> 終端友好列表： 現象 / 可能題型 / 第一反應

- **現象:** `e = 3` / `e = 5` 且 `ciphertext` 很小
  - **可能題型:** small exponent / no padding
  - **第一反應:** 試 `e` 次方根

- **現象:** `n` 不大或題目說 weak primes
  - **可能題型:** weak factorization
  - **第一反應:** factor `n`

- **現象:** 多個 `n`，同一個 `e`，同一個 message
  - **可能題型:** Hastad broadcast
  - **第一反應:** CRT + 開根

- **現象:** 多個 `n` 有共同因子
  - **可能題型:** shared prime
  - **第一反應:** 對所有 `n` 做 `gcd`

- **現象:** 同一個 `n`，兩個不同 `e` 加密同一個 message
  - **可能題型:** common modulus
  - **第一反應:** Extended Euclid

- **現象:** 已知 `p` / `q` / `phi` / `dp` / `dq`
  - **可能題型:** key leak
  - **第一反應:** 重建 `d`

- **現象:** `d` 異常小
  - **可能題型:** Wiener's attack
  - **第一反應:** continued fractions

- **現象:** 每個 ciphertext 都像 `17623416832, 10510100501...`
  - **可能題型:** char-by-char RSA
  - **第一反應:** 每個數開根或暴力字元


RSA 的基本式：

```text
c = m^e mod n
m = c^d mod n
n = p * q
phi = (p-1)(q-1)
d = e^{-1} mod phi
```

CTF 裡最常見的破法不是「破解 RSA 本身」，而是：

```text
padding 弱
p/q 弱
key 洩漏
e 太小
同一個 message 被錯誤重複加密
```

---

### A.1 標準 weak n：分解 n 後解密

#### 什麼時候想到？

題目說：

```text
weak primes
factor the modulus
find the factors
```

或者 `n` 明顯比正常 RSA 小。

#### 解法模板

```python
from sympy import factorint

n = 914224879713534891752745051672415462330144197757968187295993
e = 17
c = 15229380834061659422907324226703773035343

f = factorint(n)
print(f)

p, q = list(f.keys())
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)

m = pow(c, d, n)
print(m.to_bytes((m.bit_length() + 7) // 8, 'big'))
```

#### 核心理解

```text
只要 n 被分解成 p 和 q
就能算 phi
就能算 d
就能正常解密
```

---

### A.2 Small e + no padding：直接開 e 次方根

#### 什麼時候想到？

看到：

```text
e = 3
```

或：

```text
e = 5
```

且 ciphertext 很小，或者 ciphertext 是一串列表：

```text
[17623416832, 10510100501, ...]
```

#### 為什麼能破？

正常 RSA：

```text
c = m^e mod n
```

但如果：

```text
m^e < n
```

那 `mod n` 根本沒有發生效果，所以：

```text
c = m^e
```

那就直接：

```text
m = e-th_root(c)
```

#### gmpy2 寫法

```python
import gmpy2

c = 17623416832
e = 5
m, exact = gmpy2.iroot(c, e)
print(m, exact)
print(chr(int(m)))
```

#### 多個字元逐個開根

```python
import gmpy2

ciphertext = [17623416832, 10510100501, 9509900499]
e = 5

out = ''
for c in ciphertext:
    m, exact = gmpy2.iroot(c, e)
    if not exact:
        print('not exact root:', c)
    out += chr(int(m))

print(out)
```

#### 沒有 gmpy2 的簡單暴力版

如果每個明文字元是 ASCII，直接暴力 0–255：

```python
ciphertext = [17623416832, 10510100501, 9509900499]
e = 5

out = ''
for c in ciphertext:
    for m in range(256):
        if m ** e == c:
            out += chr(m)
            break
print(out)
```

這個在「逐字元 RSA」題目非常好用。

---

### A.3 Small message brute force：窮舉明文

#### 什麼時候想到？

明文空間很小，例如：

```text
m 是 0–255 字元
m 是 4 位 PIN
m 是常見單詞
```

#### 模板：暴力 ASCII 字元

```python
n = ...
e = ...
c = ...

for m in range(256):
    if pow(m, e, n) == c:
        print('found:', m, chr(m))
        break
```

#### 模板：暴力短 PIN

```python
n = ...
e = ...
c = ...

for pin in range(10000):
    msg = f'{pin:04d}'.encode()
    m = int.from_bytes(msg, 'big')
    if pow(m, e, n) == c:
        print(msg)
        break
```

---

### A.4 Fermat factorization：p 和 q 太接近

#### 什麼時候想到？

題目暗示：

```text
close primes
p and q are too close
```

或者 `factorint(n)` 很慢但 n 看起來像兩個接近質數相乘。

#### 原理一句話

如果：

```text
n = p*q
```

且 `p`、`q` 很接近，則：

```text
n = a^2 - b^2 = (a-b)(a+b)
```

#### 模板

```python
import gmpy2

n = ...
a = gmpy2.isqrt(n)
if a * a < n:
    a += 1

while True:
    b2 = a*a - n
    b = gmpy2.isqrt(b2)
    if b*b == b2:
        p = int(a - b)
        q = int(a + b)
        print(p, q)
        break
    a += 1
```

---

### A.5 Shared prime：多個 n 共用同一個 p

#### 什麼時候想到？

你拿到很多組：

```text
n1, n2, n3, ...
```

題目說：

```text
bad random
reused prime
many public keys
```

#### 核心命令 / Python

```python
from math import gcd

Ns = [n1, n2, n3]

for i in range(len(Ns)):
    for j in range(i+1, len(Ns)):
        g = gcd(Ns[i], Ns[j])
        if g != 1:
            print('shared prime:', i, j, g)
```

如果找到：

```text
g = p
```

那：

```python
p = g
q = n // p
```

再走標準 RSA 解密流程。

---

### A.6 Common modulus：同一個 n，不同 e，加密同一個 m

#### 什麼時候想到？

看到：

```text
n same
c1 = m^e1 mod n
c2 = m^e2 mod n
gcd(e1, e2) = 1
```

#### 原理一句話

如果能找到：

```text
a*e1 + b*e2 = 1
```

那：

```text
m = c1^a * c2^b mod n
```

#### 模板

```python
from sympy import gcdex

n = ...
e1 = ...
e2 = ...
c1 = ...
c2 = ...

s, t, g = gcdex(e1, e2)
assert g == 1

s = int(s)
t = int(t)

def modpow_with_negative(c, exp, n):
    if exp < 0:
        return pow(pow(c, -1, n), -exp, n)
    return pow(c, exp, n)

m = (modpow_with_negative(c1, s, n) * modpow_with_negative(c2, t, n)) % n
print(m.to_bytes((m.bit_length()+7)//8, 'big'))
```

---

### A.7 Hastad Broadcast Attack：同一明文，小 e，多個不同 n

#### 什麼時候想到？

看到：

```text
e = 3
same message encrypted to 3 recipients
n1,n2,n3 are different and coprime
```

#### 原理一句話

如果同一個 `m` 被用同一個小 `e` 加密到多個互質模數：

```text
c1 = m^e mod n1
c2 = m^e mod n2
c3 = m^e mod n3
```

用 CRT 合成：

```text
C = m^e
```

再開 e 次方根。

#### 模板

```python
from sympy.ntheory.modular import crt
import gmpy2

Ns = [n1, n2, n3]
Cs = [c1, c2, c3]
e = 3

C, N = crt(Ns, Cs)
C = int(C)

m, exact = gmpy2.iroot(C, e)
print(exact)
print(int(m).to_bytes((int(m).bit_length()+7)//8, 'big'))
```

---

### A.8 已知 phi(n)：直接算 d

#### 什麼時候想到？

題目直接給：

```text
phi
totient
Euler phi
```

那就不需要 factor n。

```python
n = ...
e = ...
phi = ...
c = ...

d = pow(e, -1, phi)
m = pow(c, d, n)
print(m.to_bytes((m.bit_length()+7)//8, 'big'))
```

---

### A.9 已知 p 或 q：直接算另一個因子

```python
n = ...
p = ...
q = n // p
phi = (p-1)*(q-1)
d = pow(e, -1, phi)
m = pow(c, d, n)
```

---

### A.10 已知 p+q：用二次方程解 p 和 q

如果知道：

```text
s = p + q
n = p*q
```

那：

```text
x^2 - s*x + n = 0
```

```python
import gmpy2

n = ...
s = ...
D = s*s - 4*n
root = gmpy2.isqrt(D)

p = (s + root) // 2
q = (s - root) // 2
print(p*q == n)
```

---

### A.11 RSA 題最短決策樹

```text
看到 n,e,c
│
├─ e 很小？
│   ├─ c 很小 / char-by-char → 開根 / 暴力 m
│   └─ 多組相同 m → Hastad / CRT
│
├─ n 看起來弱？
│   ├─ factorint(n)
│   ├─ Fermat close primes
│   └─ 多個 n → gcd shared prime
│
├─ 給了 p/q/phi/dp/dq？
│   └─ 重建 d
│
├─ 同 n 多個 e？
│   └─ common modulus
│
└─ 都沒有？
    └─ 可能不是基礎 RSA 題，先看 padding / oracle / implementation bug
```

---

## B. Key Header Structure：常見檔案格式 Header 解析

> CTF Forensics 的第一件事通常不是「打開檔案」，而是看前幾十個 bytes。副檔名可以騙人，header 通常比較誠實。

### B.0 常用查看命令

```bash
file target
xxd -g 1 -l 64 target
hexdump -C target | head
strings target | head
```

- `xxd -g 1`：每 byte 分開顯示。
- `-l 64`：只看前 64 bytes。
- 如果副檔名和 header 不一致，優先相信 header。

---

### B.1 JPEG / JFIF

常見開頭：

```text
FF D8 FF E0 00 10 4A 46 49 46 00 01 01 ...
```


> 終端友好列表： Offset / Bytes / 意義

- **Offset:** `0x00`
  - **Bytes:** `FF D8`
  - **意義:** SOI, Start of Image

- **Offset:** `0x02`
  - **Bytes:** `FF E0`
  - **意義:** APP0 Marker，JFIF 常用

- **Offset:** `0x04`
  - **Bytes:** `00 10`
  - **意義:** APP0 segment length，通常 16

- **Offset:** `0x06`
  - **Bytes:** `4A 46 49 46 00`
  - **意義:** ASCII `JFIF\0`

- **Offset:** 後續
  - **Bytes:** `01 01` / `01 02`
  - **意義:** JFIF version

- **Offset:** 後續
  - **Bytes:** `00` / `01` / `02`
  - **意義:** density units

- **Offset:** 後續
  - **Bytes:** X/Y density
  - **意義:** 解析度

- **Offset:** 後續
  - **Bytes:** thumbnail width/height
  - **意義:** `00 00` 常見，表示沒有縮圖


JPEG 常見 marker：


> 終端友好列表： Marker / 意義

- **Marker:** `FF D8`
  - **意義:** SOI 開始

- **Marker:** `FF E0`
  - **意義:** APP0 / JFIF

- **Marker:** `FF E1`
  - **意義:** APP1 / EXIF

- **Marker:** `FF FE`
  - **意義:** Comment，CTF 常藏字串

- **Marker:** `FF DB`
  - **意義:** Quantization Table

- **Marker:** `FF C0`
  - **意義:** Start of Frame

- **Marker:** `FF C4`
  - **意義:** Huffman Table

- **Marker:** `FF DA`
  - **意義:** Start of Scan，真正壓縮影像資料開始

- **Marker:** `FF D9`
  - **意義:** EOI 結束


CTF 快速檢查：

```bash
file img.jpg
exiftool img.jpg
strings img.jpg | grep -i 'pico\|flag\|password\|steghide'
binwalk img.jpg
xxd -g 1 -l 32 img.jpg
```

---

### B.2 PNG

固定 signature：

```text
89 50 4E 47 0D 0A 1A 0A
```

等於：

```text
\x89 PNG \r \n \x1A \n
```

PNG 由一連串 chunk 組成：

```text
[Length: 4 bytes][Type: 4 bytes][Data: N bytes][CRC: 4 bytes]
```

第一個重要 chunk 是 `IHDR`：


> 終端友好列表： Offset / Bytes / 意義

- **Offset:** `0x00`
  - **Bytes:** `89 50 4E 47 0D 0A 1A 0A`
  - **意義:** PNG signature

- **Offset:** `0x08`
  - **Bytes:** `00 00 00 0D`
  - **意義:** IHDR data length = 13

- **Offset:** `0x0C`
  - **Bytes:** `49 48 44 52`
  - **意義:** ASCII `IHDR`

- **Offset:** `0x10`
  - **Bytes:** 4 bytes
  - **意義:** width

- **Offset:** `0x14`
  - **Bytes:** 4 bytes
  - **意義:** height

- **Offset:** `0x18`
  - **Bytes:** 1 byte
  - **意義:** bit depth

- **Offset:** `0x19`
  - **Bytes:** 1 byte
  - **意義:** color type

- **Offset:** `0x1A`
  - **Bytes:** 1 byte
  - **意義:** compression method

- **Offset:** `0x1B`
  - **Bytes:** 1 byte
  - **意義:** filter method

- **Offset:** `0x1C`
  - **Bytes:** 1 byte
  - **意義:** interlace method

- **Offset:** `0x1D`
  - **Bytes:** 4 bytes
  - **意義:** CRC


Color type 常見值：


> 終端友好列表： Value / 意義

- **Value:** `00`
  - **意義:** Grayscale

- **Value:** `02`
  - **意義:** Truecolor RGB

- **Value:** `03`
  - **意義:** Indexed-color

- **Value:** `04`
  - **意義:** Grayscale + alpha

- **Value:** `06`
  - **意義:** Truecolor RGB + alpha


CTF 快速檢查：

```bash
file img.png
pngcheck -v img.png
exiftool img.png
strings img.png | grep -i 'pico\|flag\|secret'
binwalk img.png
```

PNG 題常見點：

```text
IHDR 寬高被改
IEND 後附加資料
tEXt/iTXt/zTXt chunk 藏文字
IDAT LSB 隱寫
```

---

### B.3 ZIP Local File Header

ZIP 不是只有一個 header，而是多個檔案區塊 + central directory + end record。

Local File Header 開頭：

```text
50 4B 03 04
```

ASCII 是：

```text
PK\x03\x04
```


> 終端友好列表： Offset / Size / 意義

- **Offset:** `0x00`
  - **Size:** 4
  - **意義:** Local File Header Signature `50 4B 03 04`

- **Offset:** `0x04`
  - **Size:** 2
  - **意義:** Version needed to extract

- **Offset:** `0x06`
  - **Size:** 2
  - **意義:** General purpose bit flag

- **Offset:** `0x08`
  - **Size:** 2
  - **意義:** Compression method

- **Offset:** `0x0A`
  - **Size:** 2
  - **意義:** Last mod file time

- **Offset:** `0x0C`
  - **Size:** 2
  - **意義:** Last mod file date

- **Offset:** `0x0E`
  - **Size:** 4
  - **意義:** CRC-32

- **Offset:** `0x12`
  - **Size:** 4
  - **意義:** Compressed size

- **Offset:** `0x16`
  - **Size:** 4
  - **意義:** Uncompressed size

- **Offset:** `0x1A`
  - **Size:** 2
  - **意義:** File name length = N

- **Offset:** `0x1C`
  - **Size:** 2
  - **意義:** Extra field length = M

- **Offset:** `0x1E`
  - **Size:** N
  - **意義:** File name

- **Offset:** after name
  - **Size:** M
  - **意義:** Extra field


其他 ZIP signature：


> 終端友好列表： Signature / 意義

- **Signature:** `50 4B 03 04`
  - **意義:** local file header

- **Signature:** `50 4B 01 02`
  - **意義:** central directory file header

- **Signature:** `50 4B 05 06`
  - **意義:** end of central directory

- **Signature:** `50 4B 07 08`
  - **意義:** data descriptor


CTF 快速檢查：

```bash
file archive
unzip -l archive
binwalk archive
strings archive | head
xxd -g 1 -l 64 archive
```

如果 JPG/PNG/PDF 裡面 `binwalk` 看到 `Zip archive`，試：

```bash
binwalk -e file
unzip extracted.zip
```

---

### B.4 PE / EXE / DLL

Windows executable 常見開頭：

```text
4D 5A
```

ASCII：

```text
MZ
```


> 終端友好列表： Offset / Bytes / 意義

- **Offset:** `0x00`
  - **Bytes:** `4D 5A`
  - **意義:** DOS Magic `MZ`

- **Offset:** `0x3C`
  - **Bytes:** 4 bytes
  - **意義:** `e_lfanew`，PE header offset，little-endian

- **Offset:** `e_lfanew`
  - **Bytes:** `50 45 00 00`
  - **意義:** PE Signature `PE\0\0`

- **Offset:** `e_lfanew + 4`
  - **Bytes:** 2 bytes
  - **意義:** Machine Type

- **Offset:** `e_lfanew + 6`
  - **Bytes:** 2 bytes
  - **意義:** Number of Sections


Machine type 常見：


> 終端友好列表： Bytes / 意義

- **Bytes:** `4C 01`
  - **意義:** x86

- **Bytes:** `64 86`
  - **意義:** x64 / AMD64


快速讀 `e_lfanew`：

```bash
xxd -g 1 -s 0x3c -l 4 file.exe
```

注意 little-endian，例如：

```text
E0 00 00 00 = 0x000000E0
```

快速檢查：

```bash
file file.exe
strings file.exe | grep -i 'flag\|pico\|password'
objdump -x file.exe | head
```

---

### B.5 ELF Linux Binary

開頭：

```text
7F 45 4C 46
```

ASCII：

```text
\x7f ELF
```


> 終端友好列表： Offset / Bytes / 意義

- **Offset:** `0x00`
  - **Bytes:** `7F 45 4C 46`
  - **意義:** ELF magic

- **Offset:** `0x04`
  - **Bytes:** `01` / `02`
  - **意義:** 32-bit / 64-bit

- **Offset:** `0x05`
  - **Bytes:** `01` / `02`
  - **意義:** little-endian / big-endian

- **Offset:** `0x06`
  - **Bytes:** `01`
  - **意義:** ELF version


CTF 快速檢查：

```bash
file vuln
checksec --file=vuln
strings vuln | grep -i 'flag\|pico'
nm vuln | grep ' win\| main\| vuln'
objdump -d vuln | grep '<win>'
```

---

### B.6 PDF

開頭：

```text
25 50 44 46 2D
```

ASCII：

```text
%PDF-
```

常見結構：

```text
%PDF-1.7
1 0 obj
...
endobj
xref
trailer
startxref
%%EOF
```

PDF CTF 常見藏法：

```text
/Author(base64...)
/Title(...)
/Producer(...)
/EmbeddedFile
/JS
/IEND 後附加資料（如果其實混了圖像）
```

快速檢查：

```bash
file doc.pdf
strings doc.pdf | grep -i 'pico\|flag\|author\|title\|base64\|secret'
exiftool doc.pdf
binwalk doc.pdf
```

如果有 `pdftotext`：

```bash
pdftotext doc.pdf -
pdftotext doc.pdf out.txt
```

---

### B.7 其他常見 Magic Headers


> 終端友好列表： 格式 / Header / Magic / 命令

- **格式:** GIF
  - **Header / Magic:** `47 49 46 38 37 61` / `47 49 46 38 39 61`
  - **命令:** `file`, `strings`

- **格式:** BMP
  - **Header / Magic:** `42 4D`
  - **命令:** `file`, `xxd`

- **格式:** WAV
  - **Header / Magic:** `52 49 46 46 .... 57 41 56 45`
  - **命令:** `file`, `strings`, `xxd`

- **格式:** MP3 ID3
  - **Header / Magic:** `49 44 33`
  - **命令:** `file`, `strings`

- **格式:** Gzip
  - **Header / Magic:** `1F 8B 08`
  - **命令:** `gzip -d`, `file`

- **格式:** Bzip2
  - **Header / Magic:** `42 5A 68`
  - **命令:** `bzip2 -d`

- **格式:** XZ
  - **Header / Magic:** `FD 37 7A 58 5A 00`
  - **命令:** `xz -d`

- **格式:** 7z
  - **Header / Magic:** `37 7A BC AF 27 1C`
  - **命令:** `7z` 若可用，否則 `binwalk`

- **格式:** RAR
  - **Header / Magic:** `52 61 72 21 1A 07`
  - **命令:** `file`, `binwalk`

- **格式:** SQLite
  - **Header / Magic:** `53 51 4C 69 74 65 20 66 6F 72 6D 61 74 20 33 00`
  - **命令:** `sqlite3`

- **格式:** PCAP
  - **Header / Magic:** `D4 C3 B2 A1` / `A1 B2 C3 D4`
  - **命令:** `tshark`, `tcpdump`

- **格式:** PCAPNG
  - **Header / Magic:** `0A 0D 0D 0A`
  - **命令:** `tshark`


---

## C. URL Decode / Encode

### C.1 URL encoding 是什麼？

URL 裡有些字元不能直接出現，所以會變成 `%xx`：


> 終端友好列表： 原字元 / URL encoded

- **原字元:** `:`
  - **URL encoded:** `%3A`

- **原字元:** `/`
  - **URL encoded:** `%2F`

- **原字元:** `?`
  - **URL encoded:** `%3F`

- **原字元:** `=`
  - **URL encoded:** `%3D`

- **原字元:** `&`
  - **URL encoded:** `%26`

- **原字元:** 空格
  - **URL encoded:** `%20` 或 `+`

- **原字元:** 中文
  - **URL encoded:** UTF-8 bytes 再 `%xx`


例如：

```text
https%3A%2F%2Fexample.com%2Fsearch%3Fq%3Dpython%2Bcode
```

解碼後：

```text
https://example.com/search?q=python code
```

注意：`+` 只有在 query parameter / form encoding 裡通常代表空格。

---

### C.2 CLI URL decode：Python 最穩

```bash
python3 -c 'import urllib.parse,sys; print(urllib.parse.unquote_plus(sys.argv[1]))' 'https%3A%2F%2Fexample.com%2Fsearch%3Fq%3Dpython%2Bcode%26lang%3D%E4%B8%AD%E6%96%87'
```

只解 `%xx`，不把 `+` 當空格：

```bash
python3 -c 'import urllib.parse,sys; print(urllib.parse.unquote(sys.argv[1]))' 'a+b%20c'
```

把 `+` 也當空格：

```bash
python3 -c 'import urllib.parse,sys; print(urllib.parse.unquote_plus(sys.argv[1]))' 'a+b%20c'
```

---

### C.3 CLI URL encode

```bash
python3 -c 'import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))' 'https://example.com/search?q=python code&lang=中文'
```

如果是 query value：

```bash
python3 -c 'import urllib.parse,sys; print(urllib.parse.quote_plus(sys.argv[1]))' 'python code 中文'
```

`jq` 也可以 encode：

```bash
printf '%s' 'python code 中文' | jq -sRr @uri
```

---

### C.4 Manual URL decode 函數

```python
def manual_url_decode(encoded_str):
    # In query/form encoding, '+' usually means space.
    encoded_str = encoded_str.replace('+', ' ')

    decoded_bytes = bytearray()
    i = 0
    length = len(encoded_str)

    while i < length:
        if encoded_str[i] == '%':
            hex_value = encoded_str[i+1:i+3]
            decoded_bytes.append(int(hex_value, 16))
            i += 3
        else:
            decoded_bytes.append(ord(encoded_str[i]))
            i += 1

    return decoded_bytes.decode('utf-8')

encoded_url = 'https%3A%2F%2Fexample.com%2Fsearch%3Fq%3Dpython%2Bcode%26lang%3D%E4%B8%AD%E6%96%87'
print(manual_url_decode(encoded_url))
```

CTF 用途：

```text
Web log
HTTP request
cookie
redirect URL
payload encoded 多層
```

如果解一次還像 `%xx`，再解一次。

---

## D. Encoding / Decoding CLI 轉換大全

### D.0 判斷 encoding 的直覺


> 終端友好列表： 看到 / 第一反應

- **看到:** `aGVsbG8=`
  - **第一反應:** Base64

- **看到:** `68656c6c6f`
  - **第一反應:** Hex

- **看到:** `01001000 01101001`
  - **第一反應:** Binary ASCII

- **看到:** `&#112;&#105;...`
  - **第一反應:** HTML entity

- **看到:** `%70%69%63%6f`
  - **第一反應:** URL encoding

- **看到:** `cvpbPGS`
  - **第一反應:** ROT13 可能

- **看到:** `\x70\x69\x63\x6f`
  - **第一反應:** escaped hex bytes

- **看到:** `112 105 99 111`
  - **第一反應:** decimal ASCII

- **看到:** `160 151 143 157`
  - **第一反應:** octal ASCII 可能


---

### D.1 Base64

Decode：

```bash
echo 'aGVsbG8=' | base64 -d
```

Encode：

```bash
echo -n 'hello' | base64
```

移除換行再 decode：

```bash
cat data.txt | tr -d '\n ' | base64 -d > out
```

如果是 URL-safe base64：

```bash
python3 -c 'import base64,sys; s=sys.argv[1]; s+= "="*((4-len(s)%4)%4); print(base64.urlsafe_b64decode(s))' 'cGljb0NURg'
```

---

### D.2 Base32

如果系統有 `base32`：

```bash
echo 'NBSWY3DPEB3W64TMMQ======' | base32 -d
```

Python 穩定版：

```bash
python3 -c 'import base64,sys; print(base64.b32decode(sys.argv[1]).decode())' 'NBSWY3DPEB3W64TMMQ======'
```

---

### D.3 Hex string ↔ bytes

Hex → bytes/text：

```bash
echo -n '68656c6c6f' | xxd -r -p
```

bytes/file → hex：

```bash
xxd -p file
```

只看前 64 bytes：

```bash
xxd -g 1 -l 64 file
```

Hex dump：

```bash
hexdump -C file | head
```

---

### D.4 Decimal ↔ Hex

十進位轉十六進位：

```bash
printf '0x%x\n' 255
```

十六進位轉十進位：

```bash
printf '%d\n' 0xff
```

Bash arithmetic：

```bash
echo $((0xff))
```

地址計算：

```bash
printf '0x%x\n' $((0x614551e6f33d - 0x96))
```

---

### D.5 Binary bits → text / file

如果是 `0100100001101001`：

```bash
cat bits.txt | tr -d ' \n' | fold -w8 | while read b; do printf "\\$(printf '%03o' "$((2#$b))")"; done
```

Python 寫成檔案：

```python
with open('digits.bin', 'r') as f:
    bits = f.read().strip()

data = bytes(int(bits[i:i+8], 2) for i in range(0, len(bits), 8))

with open('recovered_file', 'wb') as f:
    f.write(data)
```

判斷結果：

```bash
file recovered_file
strings recovered_file | head
```

---

### D.6 ASCII decimal → text

```bash
printf '%s\n' '112 105 99 111' | awk '{for(i=1;i<=NF;i++) printf "%c", $i; print ""}'
```

Python：

```bash
python3 -c 'print("".join(chr(int(x)) for x in "112 105 99 111".split()))'
```

字元 → ASCII：

```bash
python3 -c 'print(ord("A"))'
```

ASCII → 字元：

```bash
python3 -c 'print(chr(65))'
```

---

### D.7 Octal bytes

Octal escape → text：

```bash
printf '\160\151\143\157\012'
```

十進位轉 octal：

```bash
printf '%o\n' 112
```

Octal 轉十進位：

```bash
printf '%d\n' 0160
```

---

### D.8 ROT13

```bash
echo 'cvpbPGS' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

常見特徵：

```text
picoCTF -> cvpbPGS
```

---

### D.9 URL encoding

Decode：

```bash
python3 -c 'import urllib.parse,sys; print(urllib.parse.unquote_plus(sys.argv[1]))' 'pico%43TF%7Btest%7D'
```

Encode：

```bash
python3 -c 'import urllib.parse,sys; print(urllib.parse.quote_plus(sys.argv[1]))' 'picoCTF{test}'
```

---

### D.10 HTML entities

```bash
python3 -c 'import html,sys; print(html.unescape(sys.argv[1]))' '&#112;&#105;&#99;&#111;CTF&#123;test&#125;'
```

---

### D.11 Escaped hex：`\x70\x69...`

```bash
printf '\x70\x69\x63\x6f\x43\x54\x46\x0a'
```

如果字串在檔案裡：

```bash
python3 -c 'import codecs,sys; print(codecs.decode(sys.argv[1], "unicode_escape"))' '\x70\x69\x63\x6f'
```

---

### D.12 JWT payload decode

JWT 長這樣：

```text
header.payload.signature
```

只看 payload：

```bash
TOKEN='xxx.yyy.zzz'
python3 -c 'import sys,base64,json; p=sys.argv[1].split(".")[1]; p += "="*((4-len(p)%4)%4); print(base64.urlsafe_b64decode(p).decode())' "$TOKEN"
```

---

### D.13 XOR single-byte decode

```python
ct = bytes.fromhex('3e272d21')
for key in range(256):
    pt = bytes([b ^ key for b in ct])
    if b'pico' in pt or all(32 <= x < 127 for x in pt):
        print(key, pt)
```

如果已知 key：

```python
ct = bytes.fromhex('3e272d210d1a')
key = 0x4e
print(bytes([b ^ key for b in ct]))
```

---

### D.14 UTF-8 bytes decode

如果看到：

```text
E4 B8 AD E6 96 87
```

```bash
echo -n 'e4b8ade69687' | xxd -r -p
```

Python：

```bash
python3 -c 'print(bytes.fromhex("e4b8ade69687").decode("utf-8"))'
```

---

### D.15 Encoding 多層解碼流程

遇到一大串可疑文字：

```text
先看形狀
↓
Base64? Hex? URL? Binary?
↓
Decode 一層
↓
file / strings 看結果
↓
如果仍然像 encoded，再 decode 下一層
```

實戰命令：

```bash
# base64 → file
cat data.txt | tr -d '\n ' | base64 -d > out
file out
strings out | head

# hex → file
echo -n '...' | xxd -r -p > out
file out
strings out | head

# binary bits → file
python3 -c 's=open("bits.txt").read().strip(); open("out","wb").write(bytes(int(s[i:i+8],2) for i in range(0,len(s),8)))'
file out
```

---

## E. 速查：拿到不同輸入時的第一反應


> 終端友好列表： 輸入 / 第一反應 / 命令

- **輸入:** `.jpg`
  - **第一反應:** metadata / stego
  - **命令:** `file`, `exiftool`, `strings`, `binwalk`

- **輸入:** `.png`
  - **第一反應:** chunks / LSB / appended data
  - **命令:** `pngcheck`, `strings`, `binwalk`

- **輸入:** `.pdf`
  - **第一反應:** metadata / objects
  - **命令:** `strings`, `exiftool`, `binwalk`, `pdftotext`

- **輸入:** `.pcap`
  - **第一反應:** protocol / stream
  - **命令:** `tshark -r`, `strings`, follow stream

- **輸入:** `010101...`
  - **第一反應:** bits to bytes
  - **命令:** `fold -w8`, Python bytes

- **輸入:** `%70%69...`
  - **第一反應:** URL decode
  - **命令:** `urllib.parse.unquote_plus`

- **輸入:** `aGV...==`
  - **第一反應:** Base64
  - **命令:** `base64 -d`

- **輸入:** `68656c...`
  - **第一反應:** Hex
  - **命令:** `xxd -r -p`

- **輸入:** `n,e,c`
  - **第一反應:** RSA
  - **命令:** `factorint`, `iroot`, `pow(c,d,n)`

- **輸入:** `g,p,A,b,enc`
  - **第一反應:** DH
  - **命令:** `shared=pow(A,b,p)`

- **輸入:** ELF
  - **第一反應:** pwn/re
  - **命令:** `file`, `checksec`, `strings`, `objdump`, `nm`, `gdb`


