## Golang log

Only use log.Fatal from main.main or int functions.

因为Fatal会导致defer 无法执行--