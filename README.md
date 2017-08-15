# Medium style WYSIWYG Editor

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

## Editor Rules
- h2, h3 elements cannot have nested elements (h1 not available)
- i, b, ul can be nested in each other only
- inside ul an ol, enter should create new li
- only i, b, ul should be nested inside li
- link insertion should generate a preview
- img on paste should generate a full width preview
- code can be pasted using github gist
