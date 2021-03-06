# Canvas High Dynamic Range

## Proposal Summary / TLDR

### Use Cases

There three main classes of HDR use cases that inform this proposal. The are:
* To draw content that can precisely color-match existing SDR content, while allowing to take advantage of HDR headroom.
  * E.g, adding HDR to an existing SDR application.
  * E.g, using SDR HTML UI along with an HDR application, with guaranteed color matching.
* To draw PQ encoded HDR images as they are intended, displaying the specified nits.
  * E.g, displaying PQ content.
  * E.g, working in physical luminance.
* To draw HLG encoded HDR images as the are intended, mapping the HLG signal space to the full display device luminance range.
  * E.g, displaying HLG content.
  * E.g, using a fixed signal range that precisely maps to the full display device luminance range.

### Constraints

There exist the following constraints.
* The precise maximum luminance of the output display is not known and not knowable.
  * The [CSS Media Queries Level 5 Specification](https://www.w3.org/TR/mediaqueries-5/#valdef-media-dynamic-range-high) allows the application to query the ``'dynamic-range'``. The resulting values are ``'standard'`` and ``'high'``.
  * The value changes over time.
  * The exact value is a fingerprinting vector.
* The number of nits of SDR content is also not known and not knowable.
  * On macOS, it is always 100.
  * On Windows, it depends on a user slider setting.
  * The exact value is a fingerprinting vector (again).

### Proposed Solution

Given these constraints, it is impossible to construct a single way to display HDR content that is correct for all use cases.
The solution that we propose is to:
* Introduce new color spaces and precisions that are useful for HDR.
* Introduce an HTMLCanvasElement method through which an HDR compositing mode may be specified.
  * We introduce three HDR compositing modes, matching the three main use cases.
  * Note that these are modes of compositing a canvas, and are independent of the canvas' color space.
* Clearly define mappings from PQ and HLG signals to the defined color spaces
  * Mappings are chosen to precisely complement the HDR compositing modes
  * Remaining free parameters in the HLG mapping are chosen to ensure a smooth fallback when HDR is disabled.
  * Remaining free parameters in PQ mapping are chosen to make math easier.
* If the application can overcome the fingerprinting limitations (e.g, by just asking the user), any desired behavior can be accomplished, using math.

## New color spaces and storage formats

The existing CanvasColorSpaceProposal has been narrowed down to supporting just ``'srgb'`` and ``'display-p3'``.
It no longer covers adding other color spaces, or changing buffer formats.

This new proposal expands the set of supported color spaces to include:
* ``'rec2020'``
* ``'srgb-linear'``
* ``'rec2020-linear'``

There exist well-defined invertible transforms between all of these spaces.
The value (1,1,1) is a fixed point in all of those transforms (it means the same thing in all of these spaces).

This proposal also includes adding 16-bit floating-point as a supported storage format for 2D Canvas.
The exact mechanism is not yet defined (it will likely be in CanvasRenderingContext2DSettings or elsewhere) 

Similarly, ImageData will also be updated to support 16-bit fixed-point and 32-bit floating-point.

## HDR Compositing Modes

In this section we go over the three HDR compositing modes that we propose to make available.
These are compositing modes, which means that their effects are visible to the user via the display device, but are not visible to the application itself.

These compositing modes are independent of color space.
To be emphatic about this: Given a fixed compositing mode, then the same content, represented in different color spaces, will always appear identical.

In this section, we will define how we will map PQ and HLG color spaces to our set of defined color spaces (we will describe the mapping from those spaces into ``'srgb-linear'``, in particular).

### Mode 1: SDR-Relative Luminance (for matching SDR colors)

The requirement of this mode is that all SDR colors inside the HDR Canvas match exactly their appearance in an SDR canvas.

In this mode, color values outside of the range of [0, 1] may be used to represent luminance beyond the SDR range.
The exact luminance of such a color value is expressed relative to the maximum SDR luminance, rather than in absolute nits.
For example, the color ``color(srgb-linear 2 2 2)`` is exactly twice as bright as ``color(srgb 1 1 1)``, but it not known how many nits that is.

#### Metadata and tone mapping

There is no limit on the maximum luminance that can be expressed by a pixel value in this mode.
If no additional metadata is provided, then all pixels will be clamped to the display device's maximum luminance.

To avoid aggressive clamping, metadata may be provided, 
This metadata consists of the scene's maximum luminance, as a multiple of the SDR luminance.
A combination of the browser, operating system, and display device are responsible for mapping the canvas' into the display's capabilities.

Note that this mapping has the very severe restriction that it must respect the promise that SDR content inside the HDR canvas must match SDR content outside of the canvas exactly.
In other words, the mapping must be the identity on all SDR values.

Also note that no mastering primaries or white point are specified.
This is because there is no way to incorporate that data into a mapping that remains the identity on all SDR values.

#### Relationship to BT.2100

When a canvas with the color space ``'rec2020-linear'`` is composited in this mode, it is exactly using the scene-referred signal referred to in Table 10 of the [BT.2100 specification](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf).

#### Remarks on operating system interaction

On Windows, this mode opts the canvas in to being affected the SDR slider.

### Mode 2: Physical Luminance (for displaying PQ content)

This mode specifies a mapping from canvas color spaces to specific nit values.

PQ encoded images specify a number of nits for each pixel, and this allows hat specification to be respected.
A PQ image drawn to a canvas that is composited in this mode will appear the same as that image when drawn by an ``'img'`` element on the page.

#### Assigning pixel values to nits specified by PQ signals

In this mode, we assign a number of nits to colors.
We have to do this in two places.
* For input PQ images drawn to canvases, we have to map nits to color values.
* For outputting canvases in this compositing mode, we have to map color values to nits.

We choose to assign a the same mapping for both of these, so that simply drawing a PQ image to a canvas that is composited in this mode will behave as expected.
The alternative would be to allow the user to parameterize the output mode (that is easy to add to an API), or the input mode (which would be harder, perhaps through ImageBitmap). This approach only adds foot-guns, in our opinion. And, if the foot-guns prove to be worth it, they can be added later.

The mapping we decide is that the color ``color(srgb 1 1 1)`` is to map to 100 nits.
The motivation is that it's reasonable and that it the makes math easier for anyone who desires a different value (and many applications will want many other values).

For example if one wants to draw a color that will appear as 203 nits, this can be done with ``color(srgb-linear 2.03 2.03 2.03)``.
Similar math may be done to determine values to write to the WebGL or WebGPU swap chain.

#### Metadata and tone mapping

There is no limit on the maximum luminance that can be expressed by a pixel value in this mode.
Note that even values greater than PQ's limit of 10,000 nits are possible (e.g, ``color(srgb-linear 100.01 100.01 100.01`` is representable, and maps to 10,001 nits).
If no additional metadata is provided, then, as in the previously described mode, out of range pixel values will be clamped.

To avoid aggressive luminance clamping, the usual complement of HDR10 metadata may be provided (min luminance, max luminance, primaries, white point, CLL, and ALL).
As in the previous mode, a combination of the browser, operating system, and display device are responsible for mapping the content into the display's capabilities.

Note that unlike the previous mode, SDR colors are not sacrosanct in this mode, and SDR colors may be aggressively transformed.

#### Remarks on operating system interaction

On Windows, this mode opts the canvas out of being affected by the SDR slider.

On macOS, ``color(srgb 1 1 1)`` is always treated as matching PQ's 100 nits.
The actual number of nits displayed may vary widely depending on ambient lighting and display device capabilities (reference modes to disable these adaptations are available in operating system settings).
Consequently, if no metadata is specified (and therefore tone mapping is disabled), then this mode is equivalent to the previous mode.

#### Relationship to BT.2100

When a canvas with color space ``'rec2020-linear'`` is composited in this mode, it is almost-but-not-quite using the the display-referred signal referred to in Table 10 of the [BT.2100 specification](https://www.itu.int/dms_pubrec/itu-r/rec/bt/R-REC-BT.2100-2-201807-I!!PDF-E.pdf). The only difference is that BT.2100 indicates that a pixel value of (1,1,1) be 1 nit, while in this mode a pixel value of (1,1,1) is 100 nits.

### Mode 3: Stretching a fixed domain to all luminance (HLG scene light)

This mode specifies a mechanism through which a specific range of pixel values will be mapped to the full luminance range of the display device.

HLG encoded images specify color values in a way that is to be mapped to the entire available luminance range of the display device using the HLG inverse-EOTF and OOTF.
This mode applies this OOTF mapping to the canvas color spaces.
An HLG image drawn to a canvas that is composited in this mode will appear the same as that image when drawn by an ``'img'`` element on the page.

#### Assigning nits to HLG signals

Before discussing what we propose to do with canvas elements, we will first describe the recommended mechanism for displaying an HLG image on a display that uses raw luminance values as its signal, and has a specified maximum luminance.
This is an abbreviated summary of [PQ to HLG Transcoding](https://www.bbc.co.uk/rd/sites/50335ff370b5c262af000004/assets/592eea8006d63e5e5200f90d/BBC_HDRTV_PQ_HLG_Transcode_v2.pdf)

Transforming from HLG signal to raw luminance has three steps.

The first step is to convert from the HLG signal into scene light. This is done by the inverse-EOTF function.
The function's domain is [0, 1], and its range is normalized to [0, 1] (with 0.5 mapping to 1/12).
This step is independent of the output display.

The second step is to apply the OOTF to transform scene light into display light.
This function is a gamma ramp, parameterized by the maximum luminance of the display, with a domain and range are [0, 1].

The third step is to scale that value in [0, 1] by the display's maximum luminance (in nits) to arrive at the final luminance.

#### Assigning pixel values when drawing HLG images

What values should be written to a canvas when an HLG encoded image is drawn?
The application can query these values by a number of mechanisms, and so they cannot take into account the display device's properties that are classified as fingerprinting vectors (e.g, the display's precise maximum brightness).

The two-step process for displaying HLG signals suggests a solution to this problem, namely, that values written to the canvas should be interpreted as being in scene light.

Our solution is that an HLG encoded signal will be converted to a linear color space by applying the normalized inverse-EOTF to the signal, resulting in light values in the range of [0, 1].

#### Compositing behavior.

In this compositing mode, the browser, operating system, and display device are then responsible for converting from scene light values to display light values, by applying the appropriately parameterized OOTF and final scaling.

In this mode, the color ``color(srgb 1 1 1)`` represents maximum luminance.
All higher luminance values are outside of the domain of the HLG OOTF, and are clamped.

#### Use with SDR displays

This mode is suitable for use with display devices that are not capable of HDR output.

#### Remarks on choice of scene light space

For displaying HLG content, there are three conventions for the domain of scene light space.
* [0, 1] for normalized space scene light space.
* [0, 12] for non-normalized scene light space.
* Perhaps [0, 3.77] if one wanted SDR white to coincide with a signal value of 0.75 before applying the OOTF.

We make the first choice.
The first choice allows SDR to avoid special handling (see below), unlike the second choice (which would risk clamping all signal values greater than 0.5).
The first choice has rounder and nicer numbers than the third choice.

### Mode 0: SDR

For completeness, there does exist one more HDR mode, the no-HDR HDR mode (which is the current default behavior).

In this mode, ``'srgb-linear'`` pixel values are clamped to [0, 1].

For HLG images, the behavior that falls out is for the OOTF to not be applied.
The HLG OOTF is the identity function if the display maximum luminance is 334 nits.
So, for HLG images, the behavior for non-HDR canvases will be to display them as though the target display were 334 nits.
The is the goal of HLG: To gracefully transition between SDR and HDR.

For PQ images, the behavior that falls out is for the image to be clamped beyond 100 nits.

## Proposed API outline

```
  partial interface HTMLCanvasElement {
    void configureHDR(HTMLCanvasCompositingMode mode, optional HTMLCanvasHDRMetadata metadata);
  }

  enum HTMLCanvasHDRCompositingMode {
    'disabled',
    'relative-luminance',
    'physical-luminance',
    'hybrid-log-gamma',
  };

  dictionary HTMLCanvasHDRMetadata {
    // Value specified in multiples of SDR white.
    // Used by 'relative-luminance' mode.
    float? maxRelativeLuminance;

    // Values specified in nits.
    // Used by 'physical-luminance' mode.
    float? maxPhysicalLuminance;
    float? minPhysicalLuminance;

    // Values specified in CIE1931.
    // Used by 'physical-luminance' mode.
    float? redPrimaryX;
    float? redPrimaryY;
    float? greenPrimaryX;
    float? greenPrimaryY;
    float? bluePrimaryX;
    float? bluePrimaryY;
    float? whitePointX;
    float? whitePointY;
  }
```

## Example Applications

### Example Set 1: Application With HTML UI on top of a Canvas

In this example set we consider an application in which an HTML UI is used with content in an HTML Canvas.

The beginning of these examples is the same.
We have a canvas, and a green HTML element on top of it.
```
<html>
<body>
<div style='position: relative; width: 500px; height: 500px;'>
  <canvas id='MyCanvas' width=500px height=500px
          style='position:absolute; width: 100%; height:100%;'></canvas>
  <div id='MyGreenUI' style='position:absolute; left:10px; top:10px;
                             width:100px; height:20px;
                             background-color:rgb(0, 255, 0);'>
    MyUIText
  </div>
</div>  
<script>
  var canvas = document.getElementById('MyCanvas');
  // ... examples diverge here ...
</script>
</body>
</html>
```

We can consider two different types of applications in this general context.

#### Example 1A: Application wants the HTML UI to color-match with the HDR Canvas

In this example, the application wants color matching with the HTML UI.
The most concrete example of this would be a situation where the application uses the HTML color picker element to select colors to use inside the canvas.

In this case, the application will want to use the default mode of ``'relative-luminance'``.
```
    canvas.configureHDR('relative-luminance');

    // This green will match the color in MyGreenUI.
    var context = canvas.getContext('2d', precision='float16');
    context.fillStyle = 'rgb(0, 255, 0)';
    context.fillRect(50, 50, 20, 20);
```

#### Example 1B: Same as 1A, but using WebGL

This is the same as the above example, but the application is using WebGL.
```
    canvas.configureHDR('relative-luminance');

    // This green will match the color in MyGreenUI.
    var gl = canvas.getContext('webgl2');
    gl.configureSwapChain({format:gl.RGBA16F, colorSpace:'srgb-linear'});
    gl.clearColor(0.0, 1.0, 0.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
```

#### Example 1C: Application wants the HTML UI to be SDR content relative to the HDR Canvas

In this example, the application does not want color matching with the HTML UI.
The application views the HTML UI as being regular SDR content that is sitting on top of an HDR canvas.
* The UI should be displayed however the underlying OS displays SDR content.
* The HDR canvas should be displayed however the underlying OS displays HDR content.

The application in this case will want ``'physical-luminance'``.
```
    canvas.configureHDR('physical-luminance'});

    // This green will LIKELY NOT match the color in MyGreenUI.
    var context = canvas.getContext('2d', precision='float16');
    context.fillStyle = 'rgb(0, 255, 0)';
    context.fillRect(50, 50, 20, 20);
```

Note the phrasing of "likely not".
On Windows, if the SDR slider is set "just so", then they may happen to match.
On macOS, if there is no tonemapping applied to the canvas, then they will happen to match.

### Example Set 2: Using HDR for effects inside a Canvas element

In this set of examples, we have an existing SDR application.
This application wishes to add HDR for UI effects or for special effects.

In these examples, the application will want ``'relative-luminance'``, because that mode is guaranteed not to be disruptive to the application as it already exists.

#### Example 2A: Brightening part of a UI to draw user's attention

In this example, there exists a helpful piece of UI that the application wants to draw the user's attention to.
That UI is being drawn by the canvas element.

To draw the user's attention, the application pulses a doubling of the brightness of that part of the UI code.
```
  <html>
  <body>
  <canvas id='MyCanvas' width=500px height=500px
          style='position:absolute; width: 100%; height:100%;'></canvas>
  <script>
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHDR('relative-luminance');
    var context = canvas.getContext('2d', precision='float16');

    function animateBrigtenedUI(timeInSections) {
      // This code will brighten the content up to a factor of 2.
      var brighteningFactor = 1 + Math.abs(Math.sin(timeInSeconds * 2*Math.PI));
      context.filter = 'brightness(' + brighteningFactor + ')';

      // And this code draws the regular SDR UI.
      context.fillStyle = 'white';
      context.fillRect(100, 100, 120, 20);
      context.fillStyle = 'black';
      context.fillText('I Am Some Helpful Text', 105, 115);
    }
  </script>
  </body>
  </html>
```

#### Example 2B: An effect in a WebGL application

Consider an existing WebGL application with a lens flare effect.
The lens flare is currently an 8-bit texture, but the application wants to have it actually be HDR, without changing any of the rest of application.

Here is an outline of the previously existing application code. An existing HTMLCanvasElement, ``canvas`` is assumed to exist.
```
    var gl = canvas.getContext('webgl2');

    var myLensFlareData = new Uint8ClampedArray(4 * 256 * 256);
    // Populate the data.

    const tex = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, tex);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA8, 256, 256, 0, gl.RGBA,
                  gl.UNSIGNED_BYTE, myLensFlareData);
    gl.useProgram(myLensFlareProgram);
    gl.drawArrays(primitiveType, offset, count);
  }
```

It is possible to enable just this one HDR texture, with minimal changes to the existing application.
The comments in this code show the changes that are needed.
```
    // Enable HDR for the canvas. Use the default mode of 'relative-luminance',
    // because that will match the existing application behavior.
    canvas.configureHDR('relative-luminance');

    var gl = canvas.getContext('webgl2');

    // Configure the swap chain to be floating-point. Leave the color space as
    // the default of 'srgb'. This will leave all colors in [0, 1] unchanged,
    // but allow for specifying colors outside of the range of [0, 1].
    gl.configureSwapChain({format:gl.RGBA16F});

    // This time our data is a Float32Array, and we'll be writing values in the
    // extended sRGB color space, including values outside of [0, 1].
    var myLensFlareDataInExtendedSRGB = new Float32Array(4 * width * height);
    myLensFlareDataInExtendedSRGB[...] = ...

    const tex = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, tex);

    // Change the texture we're sampling from to be RGBA16F.
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA16F, 256, 256, 0, gl.RGBA,
                  gl.FLOAT, myLensFlareDataInExtendedSRGB);

    // Draw using the same program as before. It will sample the texture into
    // floating-point variables, and write them to the gl_FragColor.
    gl.useProgram(myLensFlareProgram);
    gl.drawArrays(primitiveType, offset, count);
  }
```

### Example Set 3: Displaying a PQ image inside a Canvas element.

In this set of examples we assume the existence of a PQ-encoded image.
Because the image is encoded in PQ, which is physical luminance, we will want the canvas to be interpreted as being in physical luminance.

#### Example 3A: Displaying a PQ image in a 2D Canvas as intended

In this example, we use a 2D Canvas to display a PQ image.
The nits specified by PQ image will match the nits displayed on the screen, as much as is allowed by the underlying operating system and the physical display.

```
  <html>
  <body>
  <canvas id='MyCanvas' width=2048px height=858px
          style='position:absolute; width: 95%;'></canvas>
  <script>
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHDR('physical-luminance'});
    var context = canvas.getContext('2d', precision='float16');

    var image = new Image();
    image.onload = function() {
      context.drawImage(image, 0, 0, 2048, 858);
    }
    var url = 'https://storage.googleapis.com/dalecurtis/cosmos_1000_pq_hdr.avif';
    image.src = url;
  </script>
  </body>
  </html>
```

#### Example 3B: Displaying a PQ image in a 2D Canvas NOT as intended

In this example, we make a mistake, and use ``'relative-luminance'`` in the above exmple.
```
  canvas.configureHDR('relative-luminance'});
```
What happens?

On Windows, the result will be that the image will be affected by the SDR slider.
This means that it will likely be brighter than is intended by the content author.

On macOS, the result will likely be indistinguishable.

#### Example 3C: Displaying a PQ image with a subtitle.

This example builds on example 3A, but adds a subtitle text to the image.

First consider the following code, where the subtitle specifies its color as being sRGB white.
In this case, the subtitle will appear as 100 nits.
```
  image.onload = function() {
    context.drawImage(image, 0, 0, 2048, 858);
    context.font = "128px Arial";
    context.fillStyle = 'white';
    context.fillText('Hello, I am a subtitle!', 400, 800);
  }
```

#### Example 3D: Displaying a PQ image with a 300 nit subtitle using CSS Color Level 4

Suppose the application does not want 100 nit subtitles, but would prefer 300 nit subtitles.
The easiest way to specify this would be to use CSS Color Level 4 syntax, in the ``srgb-linear`` color space.
In that space, the color values may be interpreted as hundreds of nits (or hectonits).
In that way, 300 nits would be represented by the style ``'color(srgb-linear 3 3 3)'`` as follows.
```
    context.font = "128px Arial";
    context.fillStyle = 'color(srgb-linear 3 3 3)';
    context.fillText('Hello, I am a subtitle!', 400, 800);
```

#### Example 3E: Displaying a PQ image with an 80 nit subtitle using a brightness filter

Alternatively, the application could use a brightness filter to achieve a different nit level for SDR content.
In this example, the application selects 80 nits for its subtitle.
```
  image.onload = function() {
    context.drawImage(image, 0, 0, 2048, 858);
    context.filter = 'brightness(0.8)';
    context.font = "128px Arial";
    context.fillStyle = 'white';
    context.fillText('Hello, I am a subtitle!', 400, 800);
  }
```

### Example Set 4: Displaying an HLG image inside a Canvas element.

In this set of examples we assume the existence of a HLG-encoded image.
We expect this HLG-encoded image to be displayed in a way that takes advantage of the full luminance of the display device.

#### Example 4A: Displaying an HLG image in a 2D Canvas as intended

In this example, we use a 2D Canvas to display a HLG image.
We set the canvas to use the appropriate compositing mode.

```
  <html>
  <body>
  <canvas id='MyCanvas' width=2048px height=858px
          style='position:absolute; width: 95%;'></canvas>
  <script>
    var canvas = document.getElementById('MyCanvas');
    canvas.configureHDR('hybrid-log-gamma'});
    var context = canvas.getContext('2d', precision='float16');

    var image = new Image();
    image.onload = function() {
      context.drawImage(image, 0, 0, 2048, 858);
    }
    var url = 'https://storage.googleapis.com/dalecurtis/cosmos_hlg.avif';
    image.src = url;
  </script>
  </body>
  </html>
```

#### Example 4B: Displaying an HLG image in a 2D Canvas NOT as intended

In this example, we make a mistake, and use ``'relative-luminance'`` in the above exmple.
```
  canvas.configureHDR('relative-luminance'});
```
What happens?

The HLG image will be limited to the SDR range (and will be drawn as though on a 334-nit SDR device).

#### Example 4C: Displaying an HLG image with a subtitle.

This example builds on example 3A, but adds a subtitle text to the image.

First consider the following code, where the subtitle specifies its color as being sRGB white.
In this case, the subtitle will appear as maxmum brightness.
```
  image.onload = function() {
    context.drawImage(image, 0, 0, 2048, 858);
    context.font = "128px Arial";
    context.fillStyle = 'white';
    context.fillText('Hello, I am a subtitle!', 400, 800);
  }
```

It is a convention in HLG to use a signal value of 0.75 as diffuse wwhite.
To accomplish this, we would convert HLG's 0.75 to a CSS color value.
```
  context.fillStyle = 'color(srgb-linear 0.265  0.265 0.265)';
```
