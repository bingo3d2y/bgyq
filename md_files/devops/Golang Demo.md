## Golang Demo

### for and local var

```go
//error
for _,i range(len(names)){
    go func test(i){
        // do something
    }
}
// v1
for _,i range(len(names)){
    j := i
    go func test(j){
        // do something
    }
}
// v2
for _,i range(len(names)){
    go func test{
        // do something
    }(i)
}
```

end

