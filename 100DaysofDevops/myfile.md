Here’s a compact `sed` cheat sheet for common lab tasks.

## Replace text
```bash
sed 's/old/new/'
sed 's/old/new/g'
sed -i 's/old/new/g' file
```

## Replace exact line
```bash
sed -i 's/^old$/new/' file
```

## Replace config values
```bash
sed -i 's/^Listen 80$/Listen 6400/' /etc/httpd/conf/httpd.conf
```

## Use a different delimiter
Useful when the text contains `/`:
```bash
sed -i 's|/var/www/html|/srv/www|g' file
```

## Print only matching lines
```bash
sed -n '/pattern/p' file
```

## Delete matching lines
```bash
sed -i '/pattern/d' file
```

## Replace multiple strings
```bash
sed -i -e 's/foo/bar/g' -e 's/old/new/g' file
```

## Insert text before or after a line
```bash
sed -i '/pattern/i\inserted line' file
sed -i '/pattern/a\inserted line' file
```

## Example for your Apache task
```bash
sed -i 's/^Listen 80$/Listen 6400/' /etc/httpd/conf/httpd.conf
```

If you want, I can also give you a **`sed` + `awk` + `grep` quick reference** for interview or lab use.