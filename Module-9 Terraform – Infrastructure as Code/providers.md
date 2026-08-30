
![omkarsharma2821](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/0r316n0o9lzl4gwadcsw.png)

### Importance of version constraints and Version Locking in Terraforma and providers versions

| Operator | Example     | Allows                     | Meaning                   |
| -------- | ----------- | -------------------------- | ------------------------- |
| `=`      | `= 6.62.0`  | `6.62.0` only              | Exact version             |
| `!=`     | `!= 6.62.0` | Everything except `6.62.0` | Exclude a version         |
| `>`      | `> 6.62.0`  | `6.62.1`, `6.63.0`, etc.   | Greater than              |
| `>=`     | `>= 6.62.0` | `6.62.0` and newer         | Greater than or equal     |
| `<`      | `< 6.62.0`  | Versions below `6.62.0`    | Less than                 |
| `<=`     | `<= 6.62.0` | `6.62.0` and older         | Less than or equal        |
| `~>`     | `~> 6.62.0` | `6.62.x`                   | Allow patch updates       |
| `~>`     | `~> 6.62`   | `6.x`                      | Allow minor/patch updates |
