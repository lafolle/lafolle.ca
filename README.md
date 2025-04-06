[![Netlify Status](https://api.netlify.com/api/v1/badges/03e9876b-bd1e-48e6-abd0-ed1fa6ca0d81/deploy-status)](https://app.netlify.com/sites/jovial-goldstine-012f10/deploys)

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
