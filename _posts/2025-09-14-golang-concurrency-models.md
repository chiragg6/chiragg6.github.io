## Concurrency
Due to a plugin called `jekyll-titles-from-headings` which is supported by GitHub Pages by default. The above header (in the markdown file) will be automatically used as the pages title.

If the file does not start with a header, then the post title will be derived from the filename.

This is a sample blog post. You can talk about all sorts of fun things here.

---

### Concurrency

#### Some T-SQL Code

```
func rangeGen(start, stop int) <-chan int {
    out := make(chan int)
    go func() {
        for i := start; i < stop; i++ {
            out <- i
        }
        close(out)
    }()
    return out
}
```

#### Some PowerShell Code

```powershell
Write-Host "This is a concurrency Code block";

# There are many other languages you can use, but the style has to be loaded first

ForEach ($thing in $things) {
    Write-Output "It highlights it using the GitHub style"
}
```
