---
title: npm vs yarn vs pnpm
catalog: true
date: 2022-04-17 20:53:08
subtitle: Difference between npm\yarn\pnpm
header-img:
tags: 
- npm
- yarn
- pnpm
- package manager
categories: package manager
---
For Frontend developer, `npm` and `yarn` is two of the most widely used package managers. In the past several years, a new package manager called `pnpm` appears and significantly improves the speed of package installation. How `pnpm` dramatically speeds up the installation? Why `npm` and `yarn` have limitations? What's the differece between `npm`, `yarn` and `pnpm`? You can get your answer after reading this article.
## What will happen when executed `npm/yarn install`
- After executing the command, a dependency tree will be built.
- packages will be resolved into a specific version according to the `package.json` file
- download the tarball of the dependency to the local offline image。
- Unzip the dependency from the offline image to the local cache
- Copy the dependency from cache to `node_modules` directory

What will the dependency tree be like after the installation?
### `npm` version <3
the dependency tree would be a nested structure. e.g.
```
node_modules
└─ foo
   ├─ index.js
   ├─ package.json
   └─ node_modules
      └─ bar
         ├─ index.js
         └─ package.json

```
Just imagine the pitfalls of the structure:
- the dependency level will be too deep and will cause excessively long file paths, especially on window systems.
- lots of duplicate packages are installed, which leads to a large file size. 
  
  For example, if `baz` and `foo` are in the same directory, and both of them depend on the same version of lodash, then lodash will be installed in both node_modules, i.e. duplicate installations.
- Module instances cannot be shared. 
  
  For example, `React` has some internal variables. It is not possible to import the same `React` instance in two different packages, so the internal variables cannot be shared, which may leads to some unpredictable bugs.
### `npm` version >=3.0.0
After npm version >=3.0.0, npm uses a flat structure. e.g.
```
node_modules
├─ foo
|  ├─ index.js
|  └─ package.json
└─ bar
   ├─ index.js
   └─ package.json

```
Compared to the previous nested structure, all dependencies are slapped into the `node_modules` directory, which avoids the endless, deep nested directory structure. When installing a new package, it will searching in the upper level `node_modules`, according to the `node require` mechanism. Once it finds the same version of the package, it will not be reinstalled, solving the problem of repeated installation of a large number of packages, and the dependency hierarchy is not too deep.

### Disadvantages of the flat structure
However, the flat structure is not invulnerable. Problems below still exist:
- Dependencies structure is uncertain. e.g.

if `foo` depends `base64-js@1.0.1` while `bar` depends `base64-js@1.0.2`, when `npm install`, the installation structure might be like this:
```
node_modules
├─ foo
├─ bar
   └─ base64-js@1.0.2
└─ base64-js@1.0.1
```
or like this:
```
node_modules
├─ foo
   └─ base64-js@1.0.1
├─ bar
└─ base64-js@1.0.2
```
the installation structure depends on `foo` and `bar`'s order in `package.json`. If `foo` is declared before, then it would be the preceding structure, otherwise would be the following one.
This is the reason why `package-lock.json` file borns.
- The flattening algorithm itself is highly complex and time-consuming.
- The project can still illegally access packages that have no declared dependencies.

## How `pnpm install`
Take installing `express` as an example:
```
pnpm init -y
pnpm install express
```
and `node_modules` folder structure would be like:
```
.pnpm
.modules.yaml
express
mime-types
```
We can see `express`'s `Symbolic link` here, where there is no `node_modules` directory in it So where is `express`'s location?

The answer is `.pnpm/express@4.17.1/node_modules/express`

The other packages seem to follow the same pattern of `<package-name>@version/node_modules/<package-name>`.
```
▾ node_modules
  ▾ .pnpm
    ▸ accepts@1.3.7
    ▸ array-flatten@1.1.1
    ...
    ▾ express@4.17.1
      ▾ node_modules
        ▸ accepts  -> ../accepts@1.3.7/node_modules/accepts
        ▸ array-flatten -> ../array-flatten@1.1.1/node_modules/array-flatten
        ...
        ▾ express
          ▸ lib
            History.md
            index.js
            LICENSE
            package.json
            Readme.md
```
`pnpm` is well designed to put the package itself and it's dependencies under the same `node_module`, which is fully compatible with native `Node`, and keeps the packages and dependencies well organized.

Instead of a dazzling array of dependencies under `node_modules` directory, the dependencies are now largely consistent with those declared in `package.json`. Even though `pnpm` will internally have some packages that will set up dependency lifting and will be lifted into the root `node_modules`, overall the root `node_modules` are much clearer and more standardized than before.

Besides, `pnpm`'s approach to dependency management also cleverly circumvents the problem of illegal access to dependencies, i.e. as long as a package does not declare a dependency in `package.json`, it won't be accessible in the project.

### Illegal access of `npm/yarn install`
What is the illegal access of `npm/yarn install` ? Let's take an example.

If package `A` depends on `B` while `B` depends on `C`, then even if `A` does not declare the dependency of `C`, due to the existence of dependency lifting, `C` is installed into the `node_modules` of `A`. Then I can directly use `C` in `A` and it can be run without problem, and it can be run normally after going live. 

Think about the pitfalls of this:
- First, the version of `B` may change at any time. 
  
  If the previous dependency is `C@1.0.1`, and now a new version of `B` releases with a dependence of `C@2.0.1`. After executing `npm/yarn install` in `A`, `C@2.0.1` would be installed, **and `A` is still using the old API of `C`, it may report an error directly**.
- Second, if `B` doesn't depend on `C` after the update, then when installing dependencies, `C` won't be installed in `node_modules`, and `A` will also report an error when referencing `C`'s code.
- Third, in a `monorepo` project, if `A` depends on `X`, `B` depends on `X`, and there is a `C` that does not depend on `X` but uses `X` in its code, `npm/yarn` will put `X` in `node_modules` in the root directory due to dependency lifting, so that `C` can run locally because it can be loaded according to node's package loading mechanism. But imagine that if `C` is sent out as a separate package and users install it separately, `X` will not be found and the code that references `X` will report an error when it is executed.

These are all potential bugs of dependency promotion. If it's a toolkit for many developers, it would be a serious hazard.

Also, `npm` has tried to solve this problem by specifying the `--global-style` parameter to disable variable lifting, but this is equivalent to going back to the days of nested dependencies, and the disadvantages mentioned earlier will still be exposed.

`npm`/`yarn` find it hard to solve the dependency lifting problem, but the community still has a specific solution. For more details, please click [here](https://github.com/dependency-check-team/dependency-check)

However, **`pnpm` uses a unique dependency management approach that not only solves the security problem of dependency lifting, but also greatly optimizes performance in time and space**.

## Advantages of `pnpm`
### Fast speed
In most scenarios, **`pnpm` is significantly faster than `npm`/`yarn`**, even when comparing to using `yarn PnP mode`

### Efficient use of disk space
`pnpm` internally uses a **content-based addressing** file system to store all files on disk:
- It doesn't install the same package over and over again. 
  
  With `npm/yarn`, if 100 projects all rely on `lodash`, then `lodash` is likely to be installed 100 times, and there are 100 places on disk where this code is written. 
  
  However, when using `pnpm`, it will only be installed once, and only one place will be written to disk, and then it will be used again directly using **hardlink** .

- Even if installing a different version of the package, **`pnpm` will reuse the code of the previous version to a great extent**. For example, if `lodash` has 100 files, and one more file is added after the update, the disk will not be rewritten with 101 files, but keep the hardlink of the original 100 files and only write the new file.

### Supporting `monorepo`
Nowadays, frontend projects are becoming more and more complex, and more projects are using `monorepo`. With the help of `monorepo`, we can use a single git repository to manage multiple subprojects, and all subprojects are stored in the packages directory of the root directory. 

Another big difference between `pnpm` and `npm/yarn` is the support for `monorepo`, which is reflected in the functionality of each subcommand. For example, execute `pnpm add A -r` in the root directory, and all packages will be added to the `A`'s dependency, and `--filter` is also supported to filter the package.

### High security
As mentioned above, previously, when using `npm/yarn`, due to the flat structure of `node_module`, if `A` depends on `B` and `B` depends on `C`, then `A` can use `C` directly, but the problem is that `A` does not declare `C` as a dependency. Therefore, this kind of illegal access will happen. However `pnpm` creates a dependency management method to solve this problem and ensure security.