```
  netsh advfirewall firewall add rule name="Whisper" dir=in action=allow protocol=TCP
  localport=10300
  netsh advfirewall firewall add rule name="LibreTranslate" dir=in action=allow protocol=TCP
  localport=5050
```