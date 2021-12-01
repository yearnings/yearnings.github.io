# Drawing images where they don't belong with console.log()
Abusing the browser console for multidimensional easter eggs.

I'm working on a giant robot that paints. More on that eventually. I'm also working on the p5.js software that drives said robot, since that's not a thing that you just can get off the shelf (yet!).

This involves things like loading image data and doing various transforms in sequence. In the course of all that, it's useful to see both the data itself, and its visual representation.

One could cook up some fancy p5 lib to log text alongside graphics to the canvas, but we already have a fine place to do this, for the sufficiently sadistic. That place is `console.log()`;

Let's get weird.

## `number[] => Color[][]`

### What's in an image?
Blah blah _"array of numerical pixel data"_ you probably know this. We'll go straight to talking how _p5js specifically_ exposes [pixel arrays](https://p5js.org/reference/#/p5/pixels): as a single flat `number[]` representing each pixel's `r, g, b, a` values.

So if `channelCount = 4 // (r, g, b ,a)` and `totalPixelCount = img.width * img.height`, then `img.pixels.length == totalPixelCount * channelCount`;

A 3x5 px image would have `3*5*4` = `60` numerical values (ranged 0 - 255) in its pixel array.

These values oriented row-first, so a 2x2 image's values are ordered

```ts
0 1
2 3
```

and not

````ts
0 2
1 3
````

### Extracting color channels
Step one is to combine these channel values into actual colors.

To preserve the original pixel values, we
* clone the array into something we can safely mutate
* iterate over it
* extract the first 4 values (corresponding to rgba channels)
* convert them to a proper `:Color`
* repeat

```ts
const mutablePixels = [...img.pixels];

const mutablePixelColors = [...Array(img.pixels.length)].map(() => {
  const [r,g,b,a] => mutablePixels.splice(0,4);
  return color(r,g,b,a);
});
```

That's a bit dense. Here it is annotated and explicitly typed:

```ts
// Clone the pixel array by spreading its values into a new array;
// Note: JS consts are immuatable in that they can't be reassigned,
// but you can freely mutate anything _inside_ the array object
// because though the _contents_ may change, the _underlying object
// reference_ is never reassigned, so JS considers it the same item.
const mutablePixels:number[] = [...img.pixels];

// A container for the result of our operation
// As the name foreshadows, we'll mutate this too, later.
const mutablePixelColors:p5.Color[] = 
  // Create a new empty array with length img.pixels.length
  [...Array(img.pixels.length)]
  // Map over this array, with empty args () because we need
  // neither the value (which is empty) nor the index (because
  // we're always operating on the front of the mutablePixels array)
  .map(() => {
    // .splice(), starting at zero (the front of the array) and
    // grabbing 4 items (one for each channel). This both removes
    // them from the mutablePixels array and returns them, where
    // they can be destructured into four consts named r,g,b,a
    const [r,g,b,a] => mutablePixels.splice(0,4);
    // Use these consts as the args for the channels of a new color
    return color(r,g,b,a);
  });

// mutablePixelColors is now a flat array of image colors
```

### Breaking it up
We now have an array of colors that's 1/4th as long as the original, but it's still just a flat/one-dimensional line.

let's further separate it into each row.

```ts
  const pixelRows = [...Array(img.height)].map(() => {
    return mutablePixelColors.splice(0, img.width);
  });
```

Lightly annotated:
```ts
  // Another empty array of the needed length
  const pixelRows:p5.Color[][] = [...Array(img.height)]
    // Still not needing empty array data or indices
    .map(() => {
      // from the start of the color array, grab as
      // many colors are required for an image row
      return mutablePixelColors.splice(0, img.width);
  });

  // pixelRows is now an array of horizontal slices, each
  // containing every color needed to display a full row.

  // Note: we spliced all the values out of mutablePixelColors,
  // which is now empty.
```

## Color[][] => console.log
What started as a 1d array of numbers is now a 2d array of colors, which is a lot more image-like.  
  
With this, we can start to have some fun.  
  
### Styling the console
From the [MDN docs for console](https://developer.mozilla.org/en-US/docs/Web/API/console#outputting_text_to_the_console):

// ---

You can use the `%c` directive to apply a CSS style to the console output:

```
console.log("This is %cMy stylish message", "color: yellow; font-style: italic; background-color: blue;padding: 2px");
```
  
The text before the directive will not be affected, but the text after the directive will be styled using the CSS declarations in the parameter.  

[link to styled console image]()

_You may use %c multiple times_

// ---

It's that last bit that's really exciting...

But first let's do it once.

### Print a swatch
```ts
const logColor = (c: p5.Color) => console.log(`%c -`, `background:${c}; color:${c}`);
```

Annotated: 
```ts
// A function that takes a color
const logColor = (c: p5.Color) => 
  // and console.logs it, passing two args:
  console.log(
    // The string to style.
    // This stars with the %c flag so the style applies
    // to the whole message, which is only a dash. This
    // content dictates the size of our styled message,
    // so the longer this string, the longer your
    // printed color swatch will appear.
    `%c -`,
    // Both the background and the text should use the
    // same color, which makes the text invisible.
    `background:${c}; color:${c}`
  );
```

Try it out:
```ts
const testColor = color(255,0,255);
logColor(testColor);
console.log('BAM!');
```

[Image preview (needs to be made)]()

How 'bout that!

This one trick on its own is wildly useful: I used it often for logging the result of automated A11Y testing on [Spectrum's design library](https://spectrum.adobe.com/). Try it anywhere you want to visually preview colors during development without touching the DOM.

## You weren't supposed to do that

Before we bring it home, a note about how `console.log` wants this content: as a single string to be styled, followed by separate css strings for each style referenced in the first string.
  
A string with 3 styles should log as 4 args: 
```ts
  console.log(
    '%c one %c two %c three',
    'style-1-css',
    'style-2-css',
    'style-3-css'
  );
```

```ts
const logColorRow = (cArr: p5.Color[]) => {
  const textString = cArr.map(() => '%c -').join('');
  cost styleStrings = cArr.map(c => `background: ${c}; color:${c};`);
  console.log(textString, ...styleStrings);
}
```

Annotated:
```ts
// A function that takes an array of colors 
// representing every pixel in an image row
const logColorRow = (cArr: p5.Color[]) => {
  // Repeat a string that applies a style and has 
  // a single dash (and a preceding space) as content
  // eg when cArr.length is 3, textString is '%c -%c -%c -'
  const textString:string = cArr.map(() => '%c -').join('');
  // An array of CSS style strings, one for each %c in textString
  const styleStrings:string[] = cArr.map(c => `background: ${c}; color:${c};`);
  // the content to be styled + destructuring the styleStrings[] as the rest of the args
  console.log(textString, ...styleStrings);
}
```

In use:
```ts
pixelRows.forEach(row => logColorRow);
```
[Image preview (needs to be made)]()

Excellent. If you highlight this console text, you'll see what those dashes are doing:

[Image preview (needs to be made)]()

## We need to go deeper
When I said _multidimensional_ easter eggs, that wasn't a cheeky joke about 2D arrays.

Let's make an easter egg for the people who highlight the easter egg.

Instead of dashes, let's encode a message. A [very important message](https://knowyourmeme.com/memes/navy-seal-copypasta).

```ts
// This leaves the original space between words at 
// the end of lines. This matters when formatting.
const veryImportantMessage = `
  What the fuck did you just fucking say about me, 
  you little bitch? I'll have you know I graduated 
  top of my class in the Navy Seals, and I've been 
  involved in numerous secret raids on Al-Quaeda, 
  and I have over 300 confirmed kills. I am trained 
  in gorilla warfare and I'm the top sniper in the 
  entire US armed forces. You are nothing to me but 
  just another target. I will wipe you the fuck out 
  with precision the likes of which has never been 
  seen before on this Earth, mark my fucking words. 
  You think you can get away with saying that shit 
  to me over the Internet? Think again, fucker. 
  As we speak I am contacting my secret network of 
  spies across the USA and your IP is being traced 
  right now so you better prepare for the storm, 
  maggot. The storm that wipes out the pathetic 
  little thing you call your life. You're fucking 
  dead, kid. I can be anywhere, anytime, and I can 
  kill you in over seven hundred ways, and that's 
  just with my bare hands. Not only am I extensively 
  trained in unarmed combat, but I have access to 
  the entire arsenal of the United States Marine 
  Corps and I will use it to its full extent to wipe 
  your miserable ass off the face of the continent, 
  you little shit. If only you could have known what 
  unholy retribution your little "clever" comment 
  was about to bring down upon you, maybe you would 
  have held your fucking tongue. But you couldn't, 
  you didn't, and now you're paying the price, you 
  goddamn idiot. I will shit fury all over you and 
  you will drown in it. You're fucking dead, kiddo.
`;

// Multiline string defs with backticks are nice,
// but now all of those tabs and line breaks are
// hardcoded in there and need to be removed.
const formatted = veryImportantMessage
  // delete newlines.
  .replace(/(\r\n|\n|\r)/gm, "")
  // replace multiple instaces of space (from the 
  // two line tabs) with a single space.
  .replace(/(\s+)/g, " ")
  // nix any lingering start and end spaces
  .trim();
```

Now, with our manifesto ready, we can refactor the row printer to use it.

Annotated where changed:
```ts
// we now expect a second arg rowIndex for the row we're on
const logColorRow = (cArr: p5.Color[], rowIndex: number) => {
  // Takes the index of the current color in the colorArray
  const text = (cIndex: number) => {
    // Return two chars per color swatch
    const count = 2;    
    // Warp the cursor forward over text used by previous rows
    const rowOffset = rowIndex * cArr.length * count;
    // and again for as many letters have been used by this row.
    const finalOffset = rowOffset + (index*count);
    // get characters left in the text, falling back to spaces thereafter
    const chars = veryImportantMessage.substr(finalOffset, count) || ' '.repeat(count);
    // return with the style marker
    return `%c${chars}`
  }

  // Map needs args now because we need the color index,
  // and we have to declare a variable for the color itself
  // along the way. The _ name is a convention for unused args
  // 
  // This index is passed directly to text()
  const textString = cArr.map((_,cIndex) => text(cIndex)).join('');
  //Added font size,line height, and monospace so pixels stay a consistent size
  cost styleStrings = cArr.map(c => `background: ${c}; color:${c}; font-size: 20px; line-height:16px font-family: monospace;`);
  console.log(textString, ...styleStrings);
}
```
