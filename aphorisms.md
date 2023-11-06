# One Liners Aphorisms
## nix-shell or virtualenv?
```nix
nix-shell -p 'python3.withPackages(pkgs: with pkgs;[ipython pytorch matplotlib itermplot])' --command 'MPLBACKEND="module://itermplot" ITERMPLOT=rv ipython'
```
![iterm plotting](images/Pasted%20image%2020231106151642.png)
