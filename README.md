# Workflows In Linked Data - Visualiser (WILD-V)

This Web Application is build using [Solid](https://solidproject.org), [n3](https://github.com/rdfjs/N3.js/), and [d3](https://github.com/d3/d3).
Currently, the following is supported:

- [x] Create workflows with tree-based structure using the [WILD](http://people.aifb.kit.edu/co1683/2017/wild/vocab) ontology, presented at the 17th International Semantic Web Conference ([ISWC](http://iswc2018.semanticweb.org)), 2018, by Käfer and Harth (see their [supplementary website](http://people.aifb.kit.edu/co1683/2018/iswc-wild/)).
- [x] Create those workflows using a graphical interface using [d3](https://github.com/d3/d3).
- [x] Display RDF lists and blank nodes nicely in the text area (which [n3](https://github.com/rdfjs/N3.js/) doesn't do out-of-the-box).
- [x] HTTP GET workflows described with WILD from the web, e.g. this workflow [here](http://people.aifb.kit.edu/co1683/2018/iswc-wild/workflows/fig1.ttl) from the supplementary website. 
- [x] Authentication using your [Solid](https://solidproject.org) WebId, e.g. to view the same workflow on my Pod [here](https://uvdsl.inrupt.net/public/).
- [x] Save workflows to your [Solid](https://solidproject.org) Pod and delete them.
- [ ] Add additional data using the graphical interface, e.g. conditions. (Which you can already do in the text area.)
<!-- - [ ] Anchor workflow documentation on an Ethereum Blockchain (tbd - that's what has been done in version 1). -->

![screenshot](https://github.com/uvdsl/wild-v/blob/main/img/screenshot.PNG?raw=true)

* __Remember:__ Some servers do not like solid identities and respond with CORS issues => Error: Failed to fetch.
* __Remember:__ Tree-based structure => no loops in activities can be visualised. Semantically they are possible, I think :)

## Build and run using Docker
```
docker build -t wild-v:latest .
docker run -d -p 8080:80 --name WILD-V wild-v:latest
```

## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

