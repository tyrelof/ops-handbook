# Linux Text Processing (The Swiss Army Knife)

## Mental Model
Text flows through pipes `|`. You filter, transform, and format it line-by-line.

## Essential Tools

### Grep (Filter Lines)
```bash
grep "error" file.log
grep -r "TODO" .          # Recursive
grep -v "debug" access.log # Invert match (exclude)
```

### Awk (Columns)
Default delimiter is whitespace.

```bash
# Print 2nd column
echo "A B C" | awk '{print $2}'  # Output: B

# Print specific columns of specific rows
ps aux | grep "python" | awk '{print $2, $11}' # PID, Command
```

### Sed (Replace)
```bash
# Replace 'foo' with 'bar' (first occurrence per line)
sed 's/foo/bar/' file.txt

# Global replace
sed 's/foo/bar/g' file.txt
```

### Sort & Uniq
`uniq` only detects adjacent duplicates, so ALWAYS sort first.

```bash
# Count occurrences of unique lines
cat access.log | awk '{print $1}' | sort | uniq -c | sort -hr
```

### Xargs (Execute per line)
Convert stdin to arguments.

```bash
# Delete all .tmp files
find . -name "*.tmp" | xargs rm
```
