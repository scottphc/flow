# Flow Documentation

## Developing locally

* `npm install` in this directory
* `npm start` in this directory
* Open a browser to the link provided in the console.

## Adding sections

The documentation site is split into several sections, 
each of which acts as its own microsite.

To add a section, add an entry to the `sections` definition in `gastby-config.js`:

```js
const sections = [
  // ...
  {
    patterns: ['cadence/**/*'],
    sidebar: {
      null : [
        'cadence/index',
        'cadence/glossary',
      ],
      Tutorial: [
        'cadence/tutorial/index',
        'cadence/tutorial/resources',
        'cadence/tutorial/contracts',
        // ...
      ],
    },
  },
  // ...
];
```

The `patterns` array holds the file patterns that describe the content 
to be included in the section. Any `.md` or `.mdx` files matched by these 
patterns will be included in the section.

For example, the `"cadence/**/*"` pattern will match the following files:

```
- cadence/
  - index.md
  - glossary.md
  - tutorial/
    - index.md
    - resources.mdx
    - contracts.mdx
```

Each file will be available at the URL that corresponds to its relative filepath:

- `cadence/index.md => https://docs.onflow.org/cadence` (`index` is omitted from the URL)
- `cadence/tutorial/resources.mdx => https://docs.onflow.org/cadence/tutorial/resources`

The `sidebar` object describes the structure of the section sidebar. 
Each page included in the section will display the same sidebar.

The sidebar navigation can optionally be split into separate categories, 
each of which is a separate list in the `sidebar` object.

For each category, the object key is used as the title for that category. 
The category with a `null` key will not display a title and can be used to 
display sidebar content at the root level.
