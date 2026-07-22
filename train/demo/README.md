# README demo GIF

Re-records `.github/demo.gif` — the terminal demo embedded in the top-level README.

Needs [vhs](https://github.com/charmbracelet/vhs) + `tree` (`brew install vhs tree`)
and the sibling `seedkit-examples/` checkout (the tree shown is the real
`07-vps-sqlite-saas` output).

```sh
cd train/demo
vhs demo.tape
cp seedkit-demo.gif ../../.github/demo.gif
```

`demo.sh` is the staged REPL the tape drives: typed prompt, generation steps,
line-by-line file tree, closing boot line.
