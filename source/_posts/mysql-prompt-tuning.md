---
title:  mysql prompt tuning
date: 2010-10-11
---
```
mysql> prompt (\u@\h) [\d]>  
PROMPT set to '(\u@\h) [\d]>'  
(debian-sys-maint@localhost) [cms]>
```

persistent in bashrc:
```   
export MYSQL_PS1='(\u@\h) [\d]> '
```
 