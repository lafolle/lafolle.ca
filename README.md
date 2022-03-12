# Powers lafolle.ca

To run local server-
```
hugo server -D
```

To introduce a new theme add it as a submodule-
```
git submodule  add https://github.com/lafolle/archie themes/archie
```

To publish-
```
hugo -D
git add public
git commit -m'New blog'
git push
```
