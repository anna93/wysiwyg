# nuxt

> Nuxt.js project

## Build Setup

``` bash
# install dependencies
$ npm install # Or yarn install

# serve with hot reload at localhost:3000
$ npm run dev

# build for production and launch server
$ npm run build
$ npm start

# generate static project
$ npm run generate
```

For detailed explanation on how things work, checkout the [Nuxt.js docs](https://github.com/nuxt/nuxt.js).

- h1, h2, h3 elements cannot have nested elements
- i, b, ul can be nested in each other only
- inside ul an ol, enter should create new li
- only i, b, ul should be nested inside li
- link insertion should generate a preview
- img on paste should generate a full width preview
- code can be pasted using github gist
