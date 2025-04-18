# building_a_javascript_bundler

## Building a Bundler

We start by having multiple files importing themselves in sequences

```bash
echo "console.log(require('./apple'));" > product/entry-point.js
echo "module.exports = 'apple ' + require('./banana') + ' ' + require('./kiwi');" > product/apple.js
echo "module.exports = 'banana ' + require('./kiwi');" > product/banana.js
echo "module.exports = 'kiwi ' + require('./melon') + ' ' + require('./tomato');" > product/kiwi.js
echo "module.exports = 'melon';" > product/melon.js
echo "module.exports = 'tomato';" > product/tomato.js
```

## Efficiently search for all files

using `JestHasteMap` looking at all files with extension `js` with the product relative path, we get
`// ['/path/to/product/apple.js', '/path/to/product/banana.js', …]`

We use yargs to add a possible entry point to run `node index.mjs --entry-point product/entry-point.js`

## Resolve the dependency graph

we use DependencyResolver from jest-resolve-dependencies,

`import { DependencyResolver } from 'jest-resolve-dependencies';`

```js
console.log(dependencyResolver.resolve(entryPoint));
// ['/path/to/apple.js']
```

Nice! With this solution we can now retrieve the full file paths of each module that our entry point depends on.

let's use a queue for the modules that need to be processed, and a `Set` to keep track of modules that have already been processed.

```js

const allFiles = new Set();
const queue = [entryPoint];
while (queue.length) {
  const module = queue.shift();
  // Ensure we process each module at most once
  // to guard for cycles.
  if (allFiles.has(module)) {
    continue;
  }

  allFiles.add(module);
  queue.push(...dependencyResolver.resolve(module));
}

console.log(chalk.bold(`❯ Found ${chalk.blue(allFiles.size)} files`));
console.log(Array.from(allFiles));
// ['/path/to/entry-point.js', '/path/to/apple.js', …]
```

### Serialize the bundle

```js
import fs from 'fs';
 
console.log(chalk.bold(`❯ Serializing bundle`));
const allCode = [];
await Promise.all(
  Array.from(allFiles).map(async (file) => {
    const code = await fs.promises.readFile(file, 'utf8');
    allCode.push(code);
  }),
);
console.log(allCode.join('\n'));
```
