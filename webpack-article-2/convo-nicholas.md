# Convo With Nicholas

Showed him how one can use the


To snapshot memory:
```
ps ufaxww
```

This prints out a snapshot of the processes running on your machine. Parent-child relationships are also expressed.

Vagrant is similar enough to our CI environment that the amount of memory you see being used by Webpack in Vagrant should closely reflect how much it actually takes up.

UPDATE: Memory tracking has evolved: (Thanks Co-op Mike)
```
ps uax | grep webpack | awk '{s+=$6} END {print s}'
```
This will:
1. Use `ps uax` to grab processes and their memory
2. Use `grep` to filter down to only lines that contain "webpack"
3. Use awk to grab the 6th column of each row, which is the resident memory column, and sum them all

Effectively, this gives you the exact amount of memory taken up by Webpack at the momemnt this command is ran.

Some things to consider:

Is it possible to place soft limits on Node processes for the amount of memory they use? (Hint: `--max-old-space-size=2048` as flag to `node`)

Is it possible to limit `fork-ts-checker-webpack-plugin` with plugin options? (Hint: We already do)

Using Chrome dev tools to snapshot heap usage, it seems like, at the very most, Webpack only uses ~600mb in the heap. If this is true, why does memory usage in Webpack spike up to 1.2gb when monitored by `ps`? Surely the stack can't be that large, especially considering that JavaScript stacks really aren't very large at all.
