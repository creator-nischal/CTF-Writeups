# OverTheWire — Bandit (0 → 33) Write-up (No Passwords)
**Author:** Nischal Dhakal  
**Platform:** Linux (Ubuntu)  
**Rule:** I’m not publishing any game credentials/passwords. Run the commands yourself and you’ll get them.

---

## Quick connect
```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```
When it asks for a password, use the one for that level (I’m not writing it here).

---

# Level 0 → Level 1 (SSH basics)
### Goal
Log in via SSH.

### Steps
```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
ls
cat readme
```

### What I learned
- SSH with a custom port (`-p 2220`)
- `ls` + `cat` is enough to win early levels

---

# Level 1 → Level 2 (filename is `-`)
### Goal
Password is stored in a file literally named `-`.

### Steps
```bash
ls
# These both work:
cat ./-
# or:
cat -- -
```

### What I learned
- `-` can be treated like stdin by tools, so you must use `./-` or `--`.

---

# Level 2 → Level 3 (spaces in filename)
### Goal
Password is in a file with spaces in the name.

### Steps
```bash
ls
cat "spaces in this filename"
# or:
cat spaces\ in\ this\ filename
```

### What I learned
- Quotes vs escaping spaces

---

# Level 3 → Level 4 (hidden file)
### Goal
Password is in a hidden file inside `inhere`.

### Steps
```bash
ls
cd inhere
ls -la
cat .hidden
```

### What I learned
- `ls -la` shows dotfiles

---

# Level 4 → Level 5 (only human-readable file)
### Goal
Inside `inhere/` there are multiple weird files; only one is readable text.

### Steps
```bash
cd inhere
ls -la
file ./*
# Find the one that says ASCII text (or similar), then:
cat ./<that_file>
```

### What I learned
- `file` is a cheat code for “what even is this file?”

---

# Level 5 → Level 6 (find by properties)
### Goal
Password file is: **human-readable**, **1033 bytes**, **not executable**, under `inhere/`.

### Steps
```bash
cd inhere
find . -type f -size 1033c ! -executable -exec file {} \; | grep -i text
# Then cat the matching file path:
cat ./<matching_file>
```

### What I learned
- `find` filters like a sniper (size, type, executable flag)

---

# Level 6 → Level 7 (find anywhere on system)
### Goal
Password file is somewhere on the server with specific owner/group/size.

### Steps
```bash
# This level usually wants: owned by bandit7, group bandit6, size 33 bytes
find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
cat /path/you/found
```

### What I learned
- `2>/dev/null` keeps your screen clean from permission spam

---

# Level 7 → Level 8 (grep the needle)
### Goal
Password is in `data.txt` next to a keyword.

### Steps
```bash
grep -n "millionth" data.txt
```

### What I learned
- `grep -n` gives line numbers (nice for screenshots + evidence)

---

# Level 8 → Level 9 (unique line)
### Goal
Password is the only line that appears once.

### Steps
```bash
sort data.txt | uniq -u
```

### What I learned
- `uniq` needs sorted input

---

# Level 9 → Level 10 (strings in binary)
### Goal
Password is in `data.txt` among human-readable strings.

### Steps
```bash
strings data.txt | grep -n "="
```

### What I learned
- `strings` extracts readable stuff from “binary-looking” files

---

# Level 10 → Level 11 (base64)
### Goal
Password is base64 encoded.

### Steps
```bash
base64 -d data.txt
```

### What I learned
- base64 decode is one command

---

# Level 11 → Level 12 (ROT13)
### Goal
Password is encrypted with ROT13.

### Steps
```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

### What I learned
- `tr` can do substitution ciphers fast

---

# Level 12 → Level 13 (hexdump + multi-compression)
### Goal
`data.txt` is a hexdump of a file that’s been compressed multiple times.

### Steps (clean + reliable)
```bash
mkdir -p /tmp/nischal_bandit12
cd /tmp/nischal_bandit12
cp ~/data.txt .

# Convert hexdump back to binary
xxd -r data.txt > data.bin

# Now repeat this loop manually:
file data.bin

# If file says gzip:
mv data.bin data.gz
gzip -d data.gz
mv data data.bin

# If file says bzip2:
mv data.bin data.bz2
bzip2 -d data.bz2
mv data data.bin

# If file says tar archive:
mv data.bin data.tar
tar -xf data.tar
# Then file the extracted output, and continue until you reach ASCII text.
```

### What I learned
- The pattern is: `file` → rename → decompress → repeat
- Always work in `/tmp` so you don’t trash your home dir

---

# Level 13 → Level 14 (SSH private key login)
### Goal
Use a private key to SSH into the next level.

### Steps
```bash
ls
# You’ll find a key file (example name: sshkey.private)
chmod 600 sshkey.private
ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
```

### What I learned
- SSH keys must be protected (`chmod 600`) or SSH refuses

---

# Level 14 → Level 15 (send password to a local port)
### Goal
Submit current level password to a service on localhost (port 30000).

### Steps
```bash
cat /etc/bandit_pass/bandit14
# Then send it:
cat /etc/bandit_pass/bandit14 | nc localhost 30000
```

### What I learned
- `localhost` means “inside the bandit server”, not my laptop
- piping a password into `nc` is clean and fast

---

# Level 15 → Level 16 (SSL/TLS service)
### Goal
Submit password to localhost:30001 **using SSL/TLS**.

### Steps (best way)
```bash
cat /etc/bandit_pass/bandit15 | openssl s_client -connect localhost:30001 -quiet
```

### What I learned
- `openssl s_client` is basically “netcat, but encrypted TLS”
- `-quiet` removes a lot of noise so you see the important output

---

# Level 16 → Level 17 (scan ports + find TLS + grab key)
### Goal
One port in 31000–32000 uses SSL/TLS and returns the next credentials.

### Steps
```bash
nmap -p 31000-32000 localhost
nmap -sV -p 31000-32000 localhost
```
From `-sV`, focus on ports that show `ssl/*`.

Connect to the SSL one and send the current password:
```bash
cat /etc/bandit_pass/bandit16 | openssl s_client -connect localhost:<ssl_port> -quiet
```

If it returns an SSH private key (starts with `-----BEGIN ... PRIVATE KEY-----`):
```bash
nano bandit17.key
# paste the key, save
chmod 600 bandit17.key
ssh -i bandit17.key bandit17@bandit.labs.overthewire.org -p 2220
```

### What I learned
- `nmap -sV` tells you which port is speaking TLS
- Some services echo your input; only one gives real creds

---

# Level 17 → Level 18 (diff)
### Goal
Password is the only changed line between `passwords.old` and `passwords.new`.

### Steps
```bash
ls
diff passwords.old passwords.new
```

### What I learned
- `diff` is instant “what changed?”

---

# Level 18 → Level 19 (logout trap via .bashrc)
### Goal
`.bashrc` kicks you out on SSH login. Still retrieve `readme`.

### Steps (no drama)
Run a single command over SSH (non-interactive):
```bash
ssh bandit18@bandit.labs.overthewire.org -p 2220 "cat readme"
```

Or copy it out with scp:
```bash
scp -P 2220 bandit18@bandit.labs.overthewire.org:readme .
cat readme
```

### What I learned
- Running a remote command avoids the “interactive shell trap”

---

# Level 19 → Level 20 (setuid wrapper)
### Goal
Use `bandit20-do` to run commands as another user.

### Steps
```bash
ls
./bandit20-do
./bandit20-do whoami
./bandit20-do cat /etc/bandit_pass/bandit20
```

### What I learned
- setuid means the program runs with the owner’s privileges
- So `./bandit20-do cat ...` reads files you normally can’t

---

# Level 20 → Level 21 (suconnect + local listener)
### Goal
`suconnect` connects to localhost:<port>.  
It expects you to send the current password, then it returns the next one.

### The KEY detail
**The listener must run on the Bandit server (inside your SSH session),** because `suconnect` connects to `localhost` *there*, not your Ubuntu machine.

### Steps (two terminals OR tmux)
**Terminal A (listener):**
```bash
nc -l -p 5000
# wait here
```

**Terminal B (client):**
```bash
./suconnect 5000
```

Now go back to Terminal A and paste the current password + press Enter.  
The next password will appear (either in A or B depending on nc version).

### Flags cheat (nc/ncat)
- `-l` = listen mode  
- `-p <port>` = local port (common on traditional `nc`)  
- `-k` = keep listening for more connections (mainly `ncat`)  

Examples:
```bash
nc -l -p 5000          # one connection
ncat -l -p 5000 -k     # keep accepting new connections
```

### What I learned
- “localhost” mistakes waste the most time in Bandit

---

# Level 21 → Level 22 (cron)
### Goal
A cronjob runs regularly. Find what it runs and where it writes the password.

### Steps
```bash
cd /etc/cron.d
ls
cat cronjob_bandit22
cat /usr/bin/cronjob_bandit22.sh
```

The script will reveal the output file path (usually in `/tmp`).  
Then:
```bash
cat /tmp/<file_from_script>
```

### What I learned
- Cron = scheduled automation, and scripts often leak output to predictable places

---

# Level 22 → Level 23 (cron + hashing)
### Goal
Cron script creates a filename from a hash.

### Steps
```bash
cd /etc/cron.d
cat cronjob_bandit23
cat /usr/bin/cronjob_bandit23.sh
```

Recreate the same hash locally exactly how the script does it, then read the file:
```bash
echo -n "<exact_string_the_script_hashes>" | md5sum
cat /tmp/<hash_you_got>
```

### What I learned
- If a script “hides” a filename using hashing, you can just compute it too

---

# Level 23 → Level 24 (cron executes your scripts)
### Goal
A cron script runs all scripts in a folder, but only if owned by you.

### Steps
Read the cron script first:
```bash
cd /etc/cron.d
cat cronjob_bandit24
cat /usr/bin/cronjob_bandit24.sh
```

Then drop your payload script where the cron looks:
```bash
cd /var/spool/bandit24/foo

cat > nischal.sh << 'EOF'
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/bandit24_from_cron
chmod 644 /tmp/bandit24_from_cron
EOF

chmod 755 nischal.sh
```

Wait ~1 minute (cron runs every minute), then:
```bash
cat /tmp/bandit24_from_cron
```

### What I learned
- If a cronjob executes your file as another user, you can make it write secrets somewhere readable

---

# Level 24 → Level 25 (brute force 4-digit PIN)
### Goal
Daemon on port 30002 needs: `<bandit24_password> <4-digit-pin>`.

### Steps (one connection, no file needed)
```bash
pass="$(cat /etc/bandit_pass/bandit24)"

for pin in $(seq -w 0000 9999); do
  echo "$pass $pin"
done | nc localhost 30002
```

Watch output until it prints the next password.

### What I learned
- `seq -w` is perfect for 0000 → 9999
- Piping into a single `nc` connection avoids reconnecting 10,000 times

---

# Level 25 → Level 26 (weird shell, escape)
### Goal
bandit26 shell is not bash and kicks you out.

### Steps (classic escape)
1) Use the key you’re given in bandit25 to SSH:
```bash
ssh -i bandit26.sshkey bandit26@bandit.labs.overthewire.org -p 2220
```

2) If it instantly exits, force it to open the pager:
- **Make your terminal window very small** so the text cannot fit (this triggers paging).

3) Once the pager opens (usually `more`/`less`):
- Press `v` to open editor (vi)
- From `vi`, spawn a shell:
```vim
:set shell=/bin/bash
:shell
```

Now you’re in a real shell as bandit26:
```bash
cat /etc/bandit_pass/bandit26
```

### What I learned
- Pagers/editors are escape hatches if you can reach them

---

# Level 26 → Level 27 (setuid again)
### Goal
Use another setuid binary to read next password.

### Steps
```bash
ls
./bandit27-do
./bandit27-do whoami
./bandit27-do cat /etc/bandit_pass/bandit27
```

### What I learned
- Same trick as Level 19, different wrapper

---

# Level 27 → Level 28 (git clone via SSH)
### Goal
Clone a remote git repo from your own machine and read the password.

### Steps (on your Ubuntu)
```bash
mkdir -p ~/ctf/otw && cd ~/ctf/otw
git clone ssh://bandit27-git@bandit.labs.overthewire.org:2220/home/bandit27-git/repo
cd repo
ls
cat README.md
```

### What I learned
- `git clone` is the real command (not `git-clone`)
- SSH URL format: `ssh://user@host:port/path`

---

# Level 28 → Level 29 (git history: secret was removed)
### Goal
Password was in README before, but got “fixed”. So check commits.

### Steps
```bash
git log --oneline
git show <older_commit_hash>
# or step back:
git show HEAD~1
```

### What I learned
- “Fixed info leak” usually means “the secret is in the previous commit”

---

# Level 29 → Level 30 (search across all commits)
### Goal
Password exists somewhere in repo history/branches.

### Steps (fastest search)
```bash
git rev-list --all > /tmp/all_commits.txt
git grep -n "password" $(cat /tmp/all_commits.txt) | head
```

When you see a commit hash that contains the real password:
```bash
git show <that_commit_hash>
```

### What I learned
- `git grep` across commits is like “CTRL+F the entire repo history”

---

# Level 30 → Level 31 (tags)
### Goal
The secret is hidden in a tag.

### Steps
```bash
git tag
git show <tagname>
```

### What I learned
- Tags can point to commits with extra hidden content

---

# Level 31 → Level 32 (push a file)
### Goal
Create `key.txt` with exact content and push to remote.

### Steps
```bash
# Create the file exactly as required:
echo "May I come in?" > key.txt

# Stage + commit + push
git add key.txt
git commit -m "Add key.txt for level"
git push origin master
```

The remote will reply with the next password after a successful push.

### What I learned
- “Push a file” means the remote has a hook that checks your commit

---

# Level 32 → Level 33 (uppercase shell escape)
### Goal
Everything you type becomes uppercase, so normal commands break. Escape.

### Steps
Try:
```bash
$0
```

Then run:
```bash
cat /etc/bandit_pass/bandit33
```

### What I learned
- `$0` is the name/path of the current shell (so it can re-launch a usable shell)
- Variables and special characters can bypass “uppercase-only” traps

---

## Git commands I used (so I don’t forget later)
- `git log --oneline` → short commit list
- `git show <hash>` → show what changed in a commit
- `git show HEAD~1` → show previous commit
- `git branch -a` → show local + remote branches
- `git checkout <branch>` → switch branch (old way)
- `git switch <branch>` → switch branch (newer way)
- `git rev-list --all` → list every commit hash
- `git grep "word" <hashes...>` → search inside commits
- `git tag` → list tags
- `git show <tag>` → show the tagged commit/content
- `git add <file>` → stage
- `git commit -m "msg"` → commit
- `git push origin master` → push

---

## Credits
OverTheWire Bandit wargame (official): https://overthewire.org/wargames/bandit/

