--- 
date: 2025-04-26T11:42:35Z
title: 'Midpoint Rounding Makes Me Sad Circa 2017'
description: ""
slug: ''
authors: []
tags: ['javascript', 'C#']
categories: []
externalLink: ""
series: []
---


## Rounding a number

Rounding numbers is easy yes?

```text
1.2 -> 1
1.8 -> 2
```

What about 1.5? 2? yes? that makes sense? yes?

WRONG!

There are many [strategies](https://en.wikipedia.org/wiki/Rounding#Rounding_to_integer) for rounding the midpoint value of .5, tl;dr, there's a bias with the values when aggregated.

Assuming we always round up in a random distributions of numbers, we find out that values at the midpoint always increment upwards but are never removed, and thus we now are overestimating values.

```text
actual -> rounded value
1.0 -> 1.0
1.1 -> 1.0 
1.2 -> 1.0
1.3 -> 1.0
1.4 -> 1.0
1.5 -> 2.0 | each rounding of this value "increases" the total by 0.5, but there is no equivalent value in the number line to remove 0.5
1.6 -> 2.0 | each rounding of this value "increases" the total by 0.4, but that is balanced out by 2.4 rounding down and removing 0.4
1.7 -> 2.0 | each rounding of this value "increases" the total by 0.3, but that is balanced out by 2.3 rounding down and removing 0.3
1.8 -> 2.0 | each rounding of this value "increases" the total by 0.2, but that is balanced out by 2.2 rounding down and removing 0.2
1.9 -> 2.0 | each rounding of this value "increases" the total by 0.1, but that is balanced out by 2.1 rounding down and removing 0.1
2.0 -> 2.0 | no rounding
2.1 -> 2.0 | complements 1.9 rounding
2.2 -> 2.0 | complements 1.8 rounding
2.3 -> 2.0 | complements 1.7 rounding
2.4 -> 2.0 | complements 1.6 rounding
2.5 -> 3.0
2.6 -> 3.0
2.7 -> 3.0
2.8 -> 3.0
2.9 -> 3.0
```

So over a large enough sample, we start to see biases in the data and this distortion is noticeable.

## How did this come to my attention in the first place?

### Background

The frontend displays these pretty tables with lots of values. There is an export functionality for the data to be downloaded to a .csv for the user. These end users love their numbers, but they notice a discrepancy in one of the values being shown, on the front end one of the cells in the table says 12 but when exported it says 13. What gives?

### The backend

The backend is written in C#, and with the deep dive from [ellisthion](https://github.com/ellisthion) some time ago, it was identified that `Math.Round` has issues with small fractional values like 1.5 is represented as 1.4999999.... and the method doesn't care about preserving precision so the rounded value is 1. `double.ToString()` on the other hand, due to some shenanigans on how stringification of the double works, behaves like midpoint rounding of away from zero.

So it was decided that for display purposes, the team would use `double.ToString()` as it mostly represents does what you think is should do, there's no real need to pick a different midpoint rounding strategy, the values have already been aggregated and we just want them display (exported) with the rounded value.

### The frontend

For the frontend, we were using Angular 2, and was using the [number pipe](https://github.com/angular/angular/blob/ffe5c49c3ebb51d534a339e0d85a0aa7967923dc/modules/%40angular/common/src/pipes/number_pipe.ts#L52C10-L52C25) for rounding purposes. Which is a wrapper for the [Internationalization API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl)

I did a deep dive into the [documentation](https://tc39.es/ecma402/#intl-object), midpoint rounding is technically not defined, circa 2017, see below for update.

The best I could find is this statement, which doesn't specify what happens with the midpoint value

> [[MinimumFractionDigits]] and [[MaximumFractionDigits]] are non-negative integers indicating the minimum and maximum fraction digits to be used. Numbers will be rounded or padded with trailing zeroes if necessary. These properties are only used when [[RoundingType]] is fraction-digits, more-precision, or less-precision.

If we do some black box testing on a few browsers,  

Reference: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/NumberFormat#digit_options>

```javascript
let options = {
maximumFractionDigits: 2,
}

console.log(new Intl.NumberFormat('en', options).format(12.225))
console.log(new Intl.NumberFormat('en', options).format(12.235))
console.log(new Intl.NumberFormat('en', options).format(-12.225))
console.log(new Intl.NumberFormat('en', options).format(-12.235))
```

Chrome and firefox uses away from zero
IE (as it was still [alive](https://blogs.windows.com/windowsexperience/2022/06/15/internet-explorer-11-has-retired-and-is-officially-out-of-support-what-you-need-to-know/) then) uses to round to positive infinity
[karma-intl-shim](https://www.npmjs.com/package/karma-intl-shim), aka our shim for the unit testing framework, uses round to zero strategy

So great, the midpoint rounding strategy is dependent on browser implementation of the internationalization API, and thus cannot be tested.

## The solution?

A few potential strategy

- Synchronize / implement the same rounding strategy between server and client
  - Client must implement custom rounding strategy (due to Internationalization API limitations) and baking into the component libraries that require rounding
- Server does all the rounding, and it serves as the source of truth
- Contribute to the spec and wait for it to be rolled out in each browsers

## Post mortem, 8 years later

This was my experience from 2017, and this is no longer an issue, the internationalization API now contains a parameter to specify a midpoint rounding strategy.

There was discussion about the midpoint rounding parameter in 2021

<https://github.com/tc39/ecma402/blob/aaa71cee2909a957ea4ab904bab1a0466a17bfca/meetings/notes-2021-02-11.md>

And in 2022, most browsers have implemented the parameter.

<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat/NumberFormat#roundingmode>

<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat>

![mdn rounding mode documentation](../mdn-rounding-mode.png)

```javascript
let options = {
  maximumFractionDigits: 2,
  roundingMode: 'ceil'
}
```
