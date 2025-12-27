# Mandelbrot-Set-P5JS-Image-Generator

Code and commandlines used to create large mandelbrot set picture and result image. Using P5.js and ImgMagick.
- Original tutorial (starter code): [YouTube/J Alex Carney](https://www.youtube.com/watch?v=ixFCsST2pF4&pp=ygUYaG93IHRvIGNyZWF0ZSBtYW5kZWxicm90)

## Generation

Semi-automatically generate 4x4 separate images in [editor.p5.js](https://editor.p5js.org/).
- Limitation: 8bit PNG is the bes format it offers, but it is good enough
- Limitation: since the result image is huge, we can only do this by creating smaller tiles of it
- You can try a different color scheme in the worker's code

```js
// Divide the entire image into a ROWS x COLS grid, and render a separate tile each time (specified by ROW, COL, 0-based)
const MAX_ITER = 4500;
const ESCAPE_RADIUS = 4;
// Utilizing parallelism to complete render faster, setting additional 2 workers since some rendering slices are much faster
const WORKER_COUNT = navigator.hardwareConcurrency + 2 || 4;
let workers = [];
let completedWorkers = 0;
let startTime;

// Limitation: 14000x8000 is about the maximum resolution allowed
const FULL_WIDTH = 56000; // Entirety width after assembly
const FULL_HEIGHT = 32000; // Entirety height after assembly
const ASPECT_RATIO = FULL_WIDTH / FULL_HEIGHT;

// Render 1 tile per run
const ROWS = 4; // Splitted image row
const COLS = 4; // Splitted image column
const ROW = 3; // Current tile row index (Manual control, range: 0..ROWS-1)
const COL = 3; // Current tile col index (Manual control, range: 0..COLS-1)

// Coordinates
let cenX = -1;
let cenY = 0;
let zoom = 1; // zoom: the y-axis range is [-1, 1] (multiplied by zoom for scaling).

let tileX, tileY, tileWidth, tileHeight;

function setup() {
  // Calculate the tile's pixel size (the last column/row could be different in size)
  tileWidth = Math.floor(FULL_WIDTH / COLS);
  tileHeight = Math.floor(FULL_HEIGHT / ROWS);
  // Allocate remaining pixels to last column/row to ensure the total pixels after splicing match FULL_WIDHT/HEIGHT
  const extraW = FULL_WIDTH - tileWidth * COLS;
  const extraH = FULL_HEIGHT - tileHeight * ROWS;

  // Pixel coordinates of the tile's top-left corner (within entire image)
  tileX = COL * tileWidth;
  tileY = ROW * tileHeight;

  // Add the remaining pixels for last column/row
  if (COL === COLS - 1) tileWidth += extraW;
  if (ROW === ROWS - 1) tileHeight += extraH;

  // Make a canvas containing only this tile 
  createCanvas(tileWidth, tileHeight);
  pixelDensity(1);
  colorMode(HSB, 360, 100, 100);

  startTime = millis();
  console.log(`Rendering tile row=${ROW}, col=${COL}, tileSize=${tileWidth}x${tileHeight}, offset=${tileX},${tileY}`);

  renderTileWithWorkers();
}

function renderTileWithWorkers() {
  // Initialize
  workers.forEach(w => w.terminate());
  workers = [];
  completedWorkers = 0;

  // Divided tile into Y sections for parallelism
  const tileStripHeight = Math.ceil(tileHeight / WORKER_COUNT);

  for (let i = 0; i < WORKER_COUNT; i++) {
    const worker = new Worker(createWorkerScript());
    workers.push(worker);

    const startYInTile = i * tileStripHeight;
    const endYInTile = Math.min(startYInTile + tileStripHeight, tileHeight);

    // Skip work allocation when there's none
    if (startYInTile >= endYInTile) {
      worker.terminate();
      completedWorkers++;
      continue;
    }

    worker.postMessage({
      tileWidth: tileWidth,
      tileHeight: tileHeight,
      startYInTile: startYInTile,
      endYInTile: endYInTile,
      // Global attributes for result alignments
      fullWidth: FULL_WIDTH,
      fullHeight: FULL_HEIGHT,
      tileOffsetX: tileX, // tile's x offset
      tileOffsetY: tileY, // tile's y offset
      maxIter: MAX_ITER,
      cenX: cenX,
      cenY: cenY,
      zoom: zoom,
      aspectRatio: ASPECT_RATIO
    });

    worker.onmessage = function(e) {
      const data = e.data;
      const pixels = new Uint8ClampedArray(data.pixels);
      const rowCount = data.endYInTile - data.startYInTile;

      const imageData = new ImageData(pixels, tileWidth, rowCount);
      // Copy tile to primary canvas
      drawingContext.putImageData(imageData, 0, data.startYInTile);

      completedWorkers++;
      console.log(`Dection ${completedWorkers}/${WORKER_COUNT} completed (tile rows ${data.startYInTile}-${data.endYInTile})`);

      // Save PNG and cleanup
      if (completedWorkers === WORKER_COUNT) {
        console.log(`Render completed! Duration: ${(millis() - startTime) / 1000}s`);
        // Generate filename and start download (prompt)
        saveCanvas(`mandelbrot_r${ROW}_c${COL}`, 'png');

        workers.forEach(w => w.terminate());
        workers = [];
      }

      worker.terminate();
    };

    worker.onerror = function(e) {
      console.error(e);
    };
  }
}

// Worker Script
function createWorkerScript() {
  const workerCode = `
    onmessage = function(e) {
      const {
        tileWidth, tileHeight,
        startYInTile, endYInTile,
        fullWidth, fullHeight,
        tileOffsetX, tileOffsetY,
        maxIter, cenX, cenY, zoom, aspectRatio
      } = e.data;

      // pixels: tileWidth * (endYInTile - startYInTile) * 4
      const rows = endYInTile - startYInTile;
      const pixels = new Uint8ClampedArray(tileWidth * rows * 4);

      // Multiplane mapping by the scale of the entire image to ensure that all tiles are aligned
      const xScale = (zoom * 2 * aspectRatio) / fullWidth;  // map per global pixel
      const yScale = (zoom * 2) / fullHeight;

      for (let y = startYInTile; y < endYInTile; y++) {
        const rowOffset = (y - startYInTile) * tileWidth * 4;
        // Global Y = tileOffsetY + y
        const globalY = tileOffsetY + y;
        const cy = (globalY - fullHeight / 2) * yScale + cenY;

        for (let x = 0; x < tileWidth; x++) {
          const pixelIndex = rowOffset + x * 4;
          // Global X = tileOffsetX + x
          const globalX = tileOffsetX + x;
          const cx = (globalX - fullWidth / 2) * xScale + cenX;

          // Mandelbrot iteration
          let zx = 0, zy = 0;
          let zx2 = 0, zy2 = 0;
          let iter = 0;

          while (zx2 + zy2 < 4 && iter < maxIter) {
            zy = 2 * zx * zy + cy;
            zx = zx2 - zy2 + cx;
            zx2 = zx * zx;
            zy2 = zy * zy;
            iter++;
          }

          if (iter === maxIter) {
            pixels[pixelIndex] = 0;
            pixels[pixelIndex + 1] = 0;
            pixels[pixelIndex + 2] = 0;
            pixels[pixelIndex + 3] = 255;
          }

          else {
            const zn = Math.sqrt(zx2 + zy2);
            // Avoid log(0) edge cases
            const safeZn = Math.max(zn, 1e-10);
            const nu = Math.log(Math.log(safeZn) / Math.LN2) / Math.LN2;
            const smoothIter = iter + 1 - nu;

            let hue = (smoothIter / maxIter * 1080) % 360;
            let saturation = 125; // hsbToRgb clamps it to [0,100] interally
            let brightness = smoothIter < maxIter * 0.1
              ? Math.min(100, smoothIter / maxIter * 1000)
              : Math.min(100, 30 + smoothIter / maxIter * 70);

            if (iter > maxIter * 0.9) {
                brightness *= (maxIter - iter) / (maxIter * 0.1);
            }

            const rgb = hsbToRgb(hue, saturation, brightness);
            pixels[pixelIndex] = rgb.r;
            pixels[pixelIndex + 1] = rgb.g;
            pixels[pixelIndex + 2] = rgb.b;
            pixels[pixelIndex + 3] = 255;
          }
/*
          else { // Generate with full HUE range, clearer but looks less asthetic
            const zn = Math.sqrt(zx2 + zy2);
            const safeZn = Math.max(zn, 1e-10);
            const nu = Math.log(Math.log(safeZn) / Math.LN2) / Math.LN2;
            const smoothIter = iter + 1 - nu;

            // HSB Color scheme
            let hue = (Math.sqrt(smoothIter) * 1080) % 360;
            let saturation = 125;
            let brightness = smoothIter < maxIter * 0.07
              ? Math.min(100, smoothIter / maxIter * 1000)
              : Math.min(100, 30 + smoothIter / maxIter * 70);

            // Prune extreme brightness on deep iteration pixels
            if (iter > maxIter * 0.9) {
                brightness *= (maxIter - iter) / (maxIter * 0.1);
            }

            const rgb = hsbToRgb(hue, saturation, brightness);
            pixels[pixelIndex] = rgb.r;
            pixels[pixelIndex + 1] = rgb.g;
            pixels[pixelIndex + 2] = rgb.b;
            pixels[pixelIndex + 3] = 255;
          }
*/
        }
      }

      // Return the ArrayBuffer
      postMessage({pixels: pixels.buffer, startYInTile, endYInTile}, [pixels.buffer]);
    };

    // HSB (0..360, 0..100, 0..100) to RGB (0..255)
    function hsbToRgb(h, s, v) {
      h = (h % 360 + 360) % 360;
      s = Math.max(0, Math.min(100, s)) / 100;
      v = Math.max(0, Math.min(100, v)) / 100;

      const i = Math.floor(h / 60);
      const f = h / 60 - i;
      const p = v * (1 - s);
      const q = v * (1 - s * f);
      const t = v * (1 - s * (1 - f));

      let r, g, b;
      switch (i % 6) {
        case 0: [r, g, b] = [v, t, p]; break;
        case 1: [r, g, b] = [q, v, p]; break;
        case 2: [r, g, b] = [p, v, t]; break;
        case 3: [r, g, b] = [p, q, v]; break;
        case 4: [r, g, b] = [t, p, v]; break;
        case 5: [r, g, b] = [v, p, q]; break;
      }

      return {
        r: Math.round(r * 255),
        g: Math.round(g * 255),
        b: Math.round(b * 255)
      };
    }
  `;

  return URL.createObjectURL(new Blob([workerCode], { type: 'application/javascript' }));
}
```

## Combination

Using Imgmagick to combine generated 4x4 images into the final result, encode to TIFF with maximum compression:
- Don't try to do this manually, its slow and complicated
- Advanced image editors like PhotoShop, have mandatory PPI resampling, thereby destroying all the details
- Change image paths to your generated image folder/directory
- Change line break symbol if you are not using CMD (shell, bash, etc.)

```batch
magick ^
  ( mandelbrot_r0_c0.png mandelbrot_r0_c1.png mandelbrot_r0_c2.png mandelbrot_r0_c3.png +append ) ^
  ( mandelbrot_r1_c0.png mandelbrot_r1_c1.png mandelbrot_r1_c2.png mandelbrot_r1_c3.png +append ) ^
  ( mandelbrot_r2_c0.png mandelbrot_r2_c1.png mandelbrot_r2_c2.png mandelbrot_r2_c3.png +append ) ^
  ( mandelbrot_r3_c0.png mandelbrot_r3_c1.png mandelbrot_r3_c2.png mandelbrot_r3_c3.png +append ) ^
  -append ^
  -depth 8 ^
  -type TrueColor ^
  -define tiff:predictor=2 ^
  -define tiff:big=true ^
  -define tiff:tile-geometry=256x256 ^
  -define tiff:zip-quality=9 ^
  -define tiff:rows-per-strip=1 ^
  -compress Zip ^
  -quality 100 ^
  mandelbrot_combined.tif
```