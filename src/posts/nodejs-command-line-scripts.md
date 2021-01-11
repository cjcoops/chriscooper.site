---
title: Node.js Command Line Scripts
date: '2020-01-20'
tags:
  - nodejs
  - utils
  - command-line
---
As A JavaScript developer I write 99% of my code in JavaScript (or TypeScript). So when I want to write a command line script I'd much rather write it in JavaScript than in bash because:

- I don't need to continually Google syntax
- I can use the npm ecosystem to utilize 3rd party packages I'm already comfortable using

This article will cover the fundamentals when writing command line scripts with Node.js, most of which I learned from Kyle Sampson's [Digging Into Node.js](https://frontendmasters.com/courses/digging-into-node/) course on Frontend Masters.

## Initial Setup

Assuming you already have Node.js installed, create a file called `my-script.js`.

At the top of `my-script.js` add the following:

``` javascript
#!/usr/bin/env node

"use strict"
```

This will tell the system to use Node.js to run the script, otherwise it will be interpreted as a regular bash script. It will also indicate that the code should be executed in JavaScript's "strict mode".

This is not essential, but assuming you want to share your script with others you'll need to create a `package.json` for any npm packages you add. If you're project doesn't already have one run `npm init`.

You can then run your script in the command line with the following

``` text
./my-script.js
```

## Passing Arguments
