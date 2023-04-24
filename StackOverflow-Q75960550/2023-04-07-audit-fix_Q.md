This is a long post, but the core question is :

### How can I *force* `package.json` to use version `2.1.1` of `nth-check` ?

The top-level package `react-scripts@5.0.1` depends *indirectly*
 on `nth-check`. \
Read the details below to get the background and more context.

### 1. In summary, actions that *failed* to fix the vulnerabilities :

* [`npm install npm@latest -g`][Link01]
* [`npm audit fix`][Link02]
* [`npm-check-updates --upgrade`][Link03] &nbsp;
 ( or [`npm-upgrade`][Link04] )
* [`npm audit fix --force`][Link02] ( *not recommended* )
* [`npm update`][Link05]
* manually inserting `"nth-check": "^2.1.1",` under `dependencies` in
 `package.json`

### 2. Background and more details

Today I created a React project, by running :
```Command-line
npx create-react-app ./
```

Before moving on, I ran [`npm install npm@latest -g`][Link01],
 and then `npm --version`, which responded `9.6.4`.

After simplifying `package.json` slightly, I ran `npm install` :
 <sup>1</sup>
```Command-line
$ npm install
…
6 high severity vulnerabilities
To address all issues (including breaking changes), run:
  npm audit fix --force
Run `npm audit` for details.
```

[![6 high severity vulnerabilities][1]][1]

[1]: https://i.imgur.com/9bNmH4d.png
"6 high severity vulnerabilities"

UPS! "6 high severity vulnerabilities" – that's not looking good. \
I tried `npm audit fix` (without the `--force` flag).
The response was :

> … \
\# npm audit report \
nth-check  <2.0.1 \
Severity: high \
Inefficient Regular Expression Complexity in nth-check -
 https://github.com/advisories/GHSA-rp65-9cf3-cjxr \
fix available via `npm audit fix --force` \
Will install react-scripts@2.1.3, which is a breaking change \
node_modules/svgo/node_modules/nth-check \
  css-select  <=3.1.0 \
  Depends on vulnerable versions of nth-check \
  node_modules/svgo/node_modules/css-select \
    svgo  1.0.0 - 1.3.2 \
    Depends on vulnerable versions of css-select \
    node_modules/svgo \
      @svgr/plugin-svgo  <=5.5.0 \
      Depends on vulnerable versions of svgo \
      node_modules/@svgr/plugin-svgo \
        @svgr/webpack  4.0.0 - 5.5.0 \
        Depends on vulnerable versions of @svgr/plugin-svgo \
        node_modules/@svgr/webpack \
          react-scripts  >=2.1.4 \
          Depends on vulnerable versions of @svgr/webpack \
          node_modules/react-scripts \
6 high severity vulnerabilities

[![npm audit report: nth-check  <2.0.1, Severity: high][2]][2]

[2]: https://i.imgur.com/ddFd9aH.png
"npm audit report: nth-check  <2.0.1, Severity: high"
<sup>^ click to enlarge</sup>

Here I had the idea that maybe the `"dependencies"` could need an
 update? \
Would that fix the *vulnerabilities*?

Using the [`npm-check-updates`][Link03] package,
 I ran `npm install --global npm-check-updates`, then
 `npm-check-updates`, and finally `npm-check-updates --upgrade` :
 <sup>2</sup>
```Command-line
$ npm-check-updates --upgrade
…
 @testing-library/react       ^13.4.0  →  ^14.0.0
 @testing-library/user-event  ^13.5.0  →  ^14.4.3
 web-vitals                    ^2.1.4  →   ^3.3.1
```

[![npm-check-updates --upgrade][3]][3]

[3]: https://i.imgur.com/00rzmr7.png
"npm-check-updates --upgrade"

I now ran `npm install` again, but was still faced with
 *6 high severity vulnerabilities*.

Therefore, I went through with the other advice given and ran
 [`npm audit fix --force`][Link02] *twice*.
 <sup>3</sup> \
Again, still seeing the message *6 high severity vulnerabilities*.

### 3. It's a bug in the `react-scripts` package <sup>4</sup>

This was a major disappointment to me.
After all, I'm using the *latest* versions of all top-level packages,
 and still get the *vulnerabilities* error.

So which package is the culprit? \
We already have enough clues to find this out.
The very first line of the *npm audit report* says *nth-check <2.0.1*,
 so the `nth-check` package is clearly a suspect. \
The page linked in the audit report,
https://github.com/advisories/GHSA-rp65-9cf3-cjxr, \
titled *Inefficient Regular Expression Complexity in nth-check*,
confirms this conjecture.

*Where in the dependency tree does `nth-check` occur?*

To answer that question, I ran `npm ls nth-check` :
```Command-line
$ npm ls nth-check
…
`-- react-scripts@5.0.1
  +-- @svgr/webpack@5.5.0
  | `-- @svgr/plugin-svgo@5.5.0
  |   `-- svgo@1.3.2
  |     `-- css-select@2.1.0
  |       `-- nth-check@1.0.2
  `-- html-webpack-plugin@5.5.0
    `-- pretty-error@4.0.0
      `-- renderkid@3.0.0
        `-- css-select@4.3.0
          `-- nth-check@2.1.1
```

[![npm ls nth-check --> nth-check@1.0.2 (OLD!)][4]][4]

[4]: https://i.imgur.com/6u3qapn.png
"npm ls nth-check --> nth-check@1.0.2 (OLD!)"

As can be seen, the second occurrence has version 2.1.1, which is
 the latest version at the time of writing.
The culprit is the old version 1.0.2.

#### 3. a. `npm update` *does not* force the use of `nth-check@2.1.1`

The first paragraph of [this answer][Link07] to
 *How to update dependencies of dependencies using npm*
 claims that running `npm update` will always update **all packages**,
 not just the ones specified in the *root* of `package.json`.
I tried it, but it didn't work.

### 4. How can I force `npm install` to use `nth-check@2.1.1` ?

One way to force `npm install` to use `nth-check@2.1.1` in all
 branches is to edit `package-lock.json` to replace the occurrences
 of nth-check@1.0.2 with nth-check@2.1.1. \
I tried it, and it *does* get rid of the vulnerabilities.

But I'm not happy with a solution that requires to manually modify
 `package-lock.json`. \
That deviates from the normal workflow, which is to start with a
 `package.json` *only*, \
and then create `package-lock.json` *automatically* by running
 `npm install`.

In an attempt to achieve this, I added the line
 `"nth-check": "^2.1.1",`
 under the (top-level) `dependencies` section, but apparently, this
 has no effect on *indirect* dependencies.

So far, I tried half a dozen actions. They all failed. What next?

---

### 5. How to reproduce all my findings <sup>5</sup>

Putting the `package.json` in an empty folder and then running
 `npm install` is all it takes to have a
 [*minimal, complete and verifiable example*][Link08].

The slightly simplified `package.json` is :
```package.json
{
  "name": "soq-75960550",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "@testing-library/jest-dom": "^5.16.5",
    "@testing-library/react": "^13.4.0",
    "@testing-library/user-event": "^13.5.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "5.0.1",
    "web-vitals": "^2.1.4"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {},
  "browserslist": {}
}
```

## References

* [Auditing package dependencies for security vulnerabilities][Link01]
* [The `npm audit [fix]` command][Link02]
* [`npm-check-updates` – `npm` package for updating packages][Link03]
* [`npm-upgrade` – `npm` package for updating packages][Link04]
* [`npm update` – `npm` *command* for updating packages][Link05]
* [Inefficient Regular Expression Complexity in nth-check][Link06]
* [How to update dependencies of dependencies using npm][Link07]
* [How to create a Minimal, Reproducible Example][Link08]
* [Fake vulnerabilities – why is npm audit broken?][Link09]

[Link01]:
https://docs.npmjs.com/auditing-package-dependencies-for-security-vulnerabilities
[Link02]:
https://docs.npmjs.com/cli/v9/commands/npm-audit#examples
[Link03]:
https://www.npmjs.com/package/npm-check-updates
[Link04]:
https://www.npmjs.com/package/npm-upgrade/v/3.1.0
[Link05]:
https://docs.npmjs.com/cli/v9/commands/npm-update
[Link06]:
https://github.com/advisories/GHSA-9c47-m6qq-7p4h
[Link07]:
https://stackoverflow.com/a/69950457
[Link08]:
https://stackoverflow.com/help/minimal-reproducible-example
[Link09]:
https://overreacted.io/npm-audit-broken-by-design/#why-is-npm-audit-broken

---

<sup>

<sup>1</sup>
The `package.json` is included at the end of the question.

<sup>2</sup>
The [`npm-upgrade`](https://www.npmjs.com/package/npm-upgrade)
 package is an alternative that offers pretty much the same
 functionality.

<sup>3</sup>
On the first run, `npm audit fix --force` *downgraded* `react-scripts`
 to ^2.1.3. \
On the second run, it *upgraded* `react-scripts` (back) to ^5.0.1.
I don't recommend using `--force`.

<sup>4</sup>
The author of `react-scripts` [would probably call this "false alarm"
 rather than "bug"][Link09], but for me as a user of the package,
 it's a bug.

<sup>5</sup>
Of course, don't expect my findings to be reproducible once the bug in
 the `react-scripts` package has been fixed.

</sup>