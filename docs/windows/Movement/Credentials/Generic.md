## Search database file on Windows
```bash
Get-ChildItem -Recurse -File | Where-Object { $_.Name -like 'db*.php' }
```
## Dump database without interactive shell
```bash
C:\xampp\mysql\bin\mysql.exe -u certificate_webapp_user -pcert!f!c@teDBPWD -e "USE Certificate_WEBAPP_DB; SELECT * FROM users;" > C:\Users\xamppuser\Documents\mysql.txt
```