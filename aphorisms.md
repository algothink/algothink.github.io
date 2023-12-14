# One Liners Aphorisms
## nix-shell or virtualenv?

With `nix-shell` you can skip installing programs on your machine.
I use this one to quickly spawn an ipython with matplotlib and plot
directly on the terminal (iTerm).

```nix
nix-shell -p 'python3.withPackages(pkgs: with pkgs;[ipython matplotlib itermplot])' \
          --command 'MPLBACKEND="module://itermplot" ITERMPLOT=rv ipython'
```
![iterm plotting](images/Pasted%20image%2020231106151642.png)

