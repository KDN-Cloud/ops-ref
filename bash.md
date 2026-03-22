# bash / shell cheat sheet

## navigation & shortcuts (readline)

```
Ctrl-a           move to start of line
Ctrl-e           move to end of line
Ctrl-f / Ctrl-b  move forward / back one char
Alt-f / Alt-b    move forward / back one word
Ctrl-w           delete word before cursor
Ctrl-u           delete from cursor to start of line
Ctrl-k           delete from cursor to end of line
Ctrl-y           paste (yank) deleted text
Ctrl-l           clear screen (same as clear)
Ctrl-r           reverse history search
Ctrl-g           cancel history search
Ctrl-c           interrupt / kill current process
Ctrl-z           suspend process (resume with fg)
Ctrl-d           send EOF / exit shell
!!               repeat last command
!$               last argument of previous command
!*               all arguments of previous command
!<n>             repeat command n from history
!<str>           repeat last command starting with str
```

## variables

```bash
VAR=value                  # assign (no spaces around =)
export VAR=value           # assign and export to child processes
echo $VAR                  # read variable
echo "${VAR}"              # safer: explicit variable boundary
unset VAR                  # remove variable
readonly VAR=value         # constant
env                        # list all environment variables
printenv VAR               # print one env var
```

### special variables

```bash
$0               script name
$1 .. $9        positional arguments
$@              all arguments (each quoted)
$*              all arguments (single string)
$#              number of arguments
$?              exit code of last command
$$              PID of current shell
$!              PID of last background process
$_              last argument of last command
```

## strings

```bash
echo "Hello, $NAME"           # double quotes expand variables
echo 'Hello, $NAME'           # single quotes are literal
echo "It's a \"quote\""       # escape double quotes
${#VAR}                        # length of variable
${VAR:-default}                # use default if VAR unset or empty
${VAR:=default}                # assign default if VAR unset
${VAR:+alt}                    # use alt if VAR is set
${VAR:?error}                  # error if VAR unset
${VAR#prefix}                  # remove shortest prefix
${VAR##prefix}                 # remove longest prefix
${VAR%suffix}                  # remove shortest suffix
${VAR%%suffix}                 # remove longest suffix
${VAR/old/new}                 # replace first match
${VAR//old/new}                # replace all matches
${VAR:2:5}                     # substring (start=2, len=5)
${VAR^^}                       # uppercase (bash 4+)
${VAR,,}                       # lowercase (bash 4+)
```

## arrays

```bash
arr=(a b c d)                  # declare array
echo ${arr[0]}                 # first element
echo ${arr[-1]}                # last element
echo ${arr[@]}                 # all elements
echo ${#arr[@]}                # length
arr+=(e f)                     # append
unset arr[2]                   # delete element
arr[2]=X                       # set element
for x in "${arr[@]}"; do ...   # iterate
```

## conditionals

```bash
if [ condition ]; then
  ...
elif [ condition ]; then
  ...
else
  ...
fi
```

### test operators

```bash
# strings
[ -z "$s" ]          # empty string
[ -n "$s" ]          # non-empty string
[ "$a" = "$b" ]      # equal
[ "$a" != "$b" ]     # not equal

# numbers
[ $a -eq $b ]        # equal
[ $a -ne $b ]        # not equal
[ $a -lt $b ]        # less than
[ $a -le $b ]        # less than or equal
[ $a -gt $b ]        # greater than
[ $a -ge $b ]        # greater than or equal

# files
[ -e file ]          # exists
[ -f file ]          # is regular file
[ -d file ]          # is directory
[ -r file ]          # readable
[ -w file ]          # writable
[ -x file ]          # executable
[ -s file ]          # non-empty
[ file1 -nt file2 ]  # file1 newer than file2
[ file1 -ot file2 ]  # file1 older than file2

# logical
[ a ] && [ b ]       # AND
[ a ] || [ b ]       # OR
! [ a ]              # NOT
[[ a && b ]]         # AND (double bracket, bash)
[[ a || b ]]         # OR (double bracket, bash)
```

> Prefer `[[ ]]` over `[ ]` in bash scripts — it handles spaces and patterns more safely.

## loops

```bash
# for loop
for i in 1 2 3; do
  echo $i
done

# C-style for
for ((i=0; i<5; i++)); do
  echo $i
done

# while
while [ condition ]; do
  ...
done

# until
until [ condition ]; do
  ...
done

# iterate files
for f in /path/*.log; do
  echo "$f"
done

# iterate lines of file
while IFS= read -r line; do
  echo "$line"
done < file.txt

# iterate command output
while IFS= read -r line; do
  echo "$line"
done < <(command)
```

## functions

```bash
my_func() {
  local arg1="$1"
  echo "arg: $arg1"
  return 0
}

my_func "hello"

# function with default
greet() {
  local name="${1:-world}"
  echo "Hello, $name"
}
```

## input / output

```bash
read VAR                   # read input into VAR
read -p "Prompt: " VAR     # with prompt
read -s VAR                # silent (passwords)
read -t 5 VAR              # timeout after 5 seconds
read -a arr                # read into array
```

## redirects

```bash
cmd > file              # stdout to file (overwrite)
cmd >> file             # stdout to file (append)
cmd 2> file             # stderr to file
cmd 2>&1                # redirect stderr to stdout
cmd > file 2>&1         # stdout and stderr to file
cmd &> file             # same (bash shorthand)
cmd < file              # stdin from file
cmd1 | cmd2             # pipe stdout of cmd1 to stdin of cmd2
cmd1 |& cmd2            # pipe stdout + stderr
cmd > /dev/null         # discard stdout
cmd > /dev/null 2>&1    # discard all output
```

### here-doc & here-string

```bash
cat << EOF
line one
line two
EOF

cat <<- EOF        # strip leading tabs
	line one
EOF

cmd <<< "string"   # here-string (single string as stdin)
```

## process management

```bash
cmd &              # run in background
jobs               # list background jobs
fg %1              # bring job 1 to foreground
bg %1              # resume job 1 in background
kill %1            # kill job 1
kill -9 <pid>      # force kill by PID
wait               # wait for all background jobs
nohup cmd &        # run immune to hangup
disown %1          # detach job from shell
ps aux             # list all processes
ps aux | grep name # find process by name
pgrep name         # find PID by name
pkill name         # kill by name
top / htop         # interactive process viewer
```

## subshells & command substitution

```bash
result=$(command)          # capture output
result=`command`           # older syntax (avoid)
(cd /tmp && ls)            # subshell — directory change doesn't persist
{ cd /tmp; ls; }           # current shell — changes persist (note: space + semicolon)
```

## arithmetic

```bash
echo $((2 + 3))            # arithmetic expansion
echo $((VAR * 2))
let "x = 5 + 3"
x=$(( x + 1 ))
(( x++ ))                  # increment (no $)
(( x > 5 )) && echo "yes"  # condition (exit code)
```

## string manipulation with tools

```bash
echo "hello world" | tr 'a-z' 'A-Z'     # uppercase
echo "  trim  " | xargs                  # trim whitespace
echo "a:b:c" | cut -d: -f2              # split on : get field 2
echo "hello" | sed 's/l/L/g'            # replace
echo "foo bar baz" | awk '{print $2}'   # print second word
grep -r "pattern" /path                  # recursive search
grep -i "pattern" file                   # case-insensitive
grep -v "pattern" file                   # invert match
grep -l "pattern" /path/*               # list matching files only
grep -n "pattern" file                   # show line numbers
```

## find

```bash
find . -name "*.log"                     # find by name
find . -type f -name "*.sh"             # files only
find . -type d -name "config"           # dirs only
find . -mtime -7                         # modified in last 7 days
find . -size +100M                       # larger than 100MB
find . -empty                            # empty files/dirs
find . -name "*.log" -delete            # find and delete
find . -name "*.sh" -exec chmod +x {} \; # find and exec
find . -mindepth 1 -maxdepth 1 -type d  # immediate subdirs only
```

## scripting best practices

```bash
#!/usr/bin/env bash
set -euo pipefail          # exit on error, unset vars, pipe failures
IFS=$'\n\t'               # safer word splitting

# always quote variables
cp "$source" "$dest"

# check command exists
command -v docker &>/dev/null || { echo "docker not found"; exit 1; }

# temp files
tmp=$(mktemp)
trap "rm -f $tmp" EXIT     # cleanup on exit

# logging
log() { echo "[$(date +%H:%M:%S)] $*"; }
err() { echo "[ERROR] $*" >&2; }
```

## useful one-liners

```bash
# count lines in file
wc -l file.txt

# sort and unique
sort file.txt | uniq -c | sort -rn

# watch a command every 2 seconds
watch -n 2 df -h

# tail with follow
tail -f /var/log/syslog

# show disk usage, sorted
du -sh /* | sort -rh | head -20

# show open ports
ss -tlnp

# check if port is open
nc -zv host 22

# repeat a command n times
for i in $(seq 1 5); do echo $i; done

# run command on ssh host
ssh user@host 'bash -s' < local_script.sh

# copy with progress
rsync -avh --progress src/ dest/

# generate random string
openssl rand -hex 16

# encode / decode base64
echo "hello" | base64
echo "aGVsbG8K" | base64 -d

# timestamp a file
cp file.txt "file.txt.$(date +%Y%m%d_%H%M%S)"

# show all aliases
alias

# source a file
source ~/.bashrc  # or: . ~/.bashrc
```
