# Linux Navigation & File Management

## Mental Model
Everything is a file (including hardware and sockets). Files are organized in a single hierarchical tree starting at `/`.

## Finding Files
```bash
# Find files by name (case insensitive)
find . -iname "*config*"

# Find files modified in the last 24 hours
find . -mtime -1

# Find files larger than 100MB
find . -type f -size +100M

# Locate (fast, indexed search)
sudo updatedb
locate <filename>
```

## Permissions (chmod/chown)
Format: `rwx` (Read=4, Write=2, Execute=1)

```bash
# User can R/W/X, Group can R/X, Others can R/X
chmod 755 script.sh

# Make executable
chmod +x script.sh

# Change owner
chown user:group file
chown -R user:group directory/
```

## Sticky Bit & Special Permissions
- **Sticky Bit (`chmod +t /tmp`)**: Only owner can delete their files in shared dir.
- **SGID (`chmod g+s dir`)**: New files inherit group ownership of directory.
