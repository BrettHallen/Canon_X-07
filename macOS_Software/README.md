# [Marcus von Cube’s CASsette IO Utilities](https://mvcsys.de/doc/casioutil.html)
Compiled for macOS.<br>

## basX07
```
usage: basX07 <options> infile outfile
         -b create a binary file
         -w create a WAV file
         -t create plain text for download via serial or USB interface
            <delay> is an optional delay in seconds after each line
         -c set character translation mode:
           D DOS charset - code page 850
           W Windows charset - code page 1252 (default)
           U Unicode (UTF-8) charset
           This simplifies the use of national characters
         -s remove space characters to save memory
```

## listX07
```
usage: listX07 <options> infile > outfile
         -w reads a WAV file directly
         -w- inverts the phase in case of difficulty
         -b<skip> reads a binary file, <skip> is an optional offset
         -i ignores any errors on the input stream
         -e[C|X] use backslash escapes in strings
           C: character escapes like '\pi'
           X: generic hexadecimal escapes (default)
         -c[A|D|W] use character set translation in strings
           A: ASCII codes only (very simple)
           D: DOS code page 437
           U: Unicode (UTF-8)
           W: Windows code page 1252 (default)
         -c takes precedence over -e. Both can be mixed.
```

## waveX07
```
usage: waveX07 -b<skip> infile outfile
               <skip> is an optional offset into the file
```