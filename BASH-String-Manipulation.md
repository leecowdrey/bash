# BASH String Manipulation 

```bash
#${var#*SubStr}  # drops substring from start of string up to first occurrence of `SubStr`
#${var##*SubStr} # drops substring from start of string up to last occurrence of `SubStr`
#${var%SubStr*}  # drops substring from last occurrence of `SubStr` to end of string
#${var%%SubStr*} # drops substring from first occurrence of `SubStr` to end of string
```
