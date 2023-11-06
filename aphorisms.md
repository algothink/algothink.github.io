# nix-shell or virtualenv?
```nix
nix-shell -p 'python3.withPackages(pkgs: with pkgs;[ipython pytorch matplotlib itermplot])' --command 'MPLBACKEND="module://itermplot" ITERMPLOT=rv ipython'
```
![iterm plotting](https://github.com/algothink/algothink.github.io/blob/9b931367d960c0ca45dd0f793a653ab7a41e20a7/images/Pasted%20image%2020231106151642.png)https://github.com/algothink/algothink.github.io/blob/9b931367d960c0ca45dd0f793a653ab7a41e20a7/images/Pasted%20image%2020231106151642.png)
