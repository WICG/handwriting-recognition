
# Handwriting Recognition Explainer

Authors:
- Jiewei Qian <qjw@google.com>
- Matt Giuca <mgiuca@google.com>
- Jon Napper <napper@google.com>
- Tom Buckley <tbuckley@google.com>

## Overview

Handwriting is a widely used input method, one key usage is to recognize the texts when users are drawing. This feature already exists on many operating systems (e.g. handwriting input methods). However, the web platform as of today doesn't have this capability, the developers need to integrate with third-party libraries (or cloud services), or to develop native apps.

We want to add handwriting recognition capability to the web platform, so developers can use the existing handwriting recognition features available on the operating system.

This document describes our proposal for a Web Platform API for performing on-line handwriting recognition from recorded real-time user inputs (e.g. touch, stylus).  The term “on-line” means the API recognizes the text as the users are drawing them.


## Problem Description

Conceptually, handwriting inputs are drawings. A drawing captures the information required to recreate a pen-tip movement for text recognition purposes. Take the handwritten “WEB” for example:

![Handwriting Concept](/images/handwriting-concept.svg)

*   A **drawing** consists of multiple ink strokes (e.g. the above letter E consists of three ink strokes).
*   An **ink stroke** represents one continuous pen-tip movement that happens across some time period (e.g. from one `touchstart` to its corresponding `touchend` event). The movement trajectory is represented by a series of ink points.
*   An **ink point** is an observation of the pen-tip in space and time. It records the timestamp and position of the pen-tip on the writing surface (e.g. a `touchmove` event).

We want the handwriting API to enable web developers to fully utilize the capabilities available in common handwriting recognition libraries. The recognizer will need to:
* Accept a vector representation of a drawing (described in the above picture).
* Recognize texts as users are writing, in real-time (each recognition costs less than hundreds of milliseconds).
* Not rely on the Internet (a note taking website should still work in flight mode). Though the recognizer can make use of cloud services if available.
* Return the text that's most likely written as a string.
* Allow web developers to control or fine-tune the recognizer. For example, allow developers to specify the language (an English recognizer won't recognize Chinese characters).
* Offer an extensible way to add support for new features, in order to utilize the latest features available on the underlying libraries.
* Provide a way for developers to query feature support, so developers can decide if the recognizer should be used in their app.

To satisfy common use cases, the recognizer also need to:
Return a list of alternatives (candidates) of the text.
* Rank alternatives based on likelihood of being correct.
* Return segmentation result for each character (or words). So clients can know which strokes (and points) make up a character. One use case is in note taking apps, users select recognized texts, and delete all strokes.

Non-goals:
* Design an API to recognize texts in static images. That is optical character recognition, and is better aligned with Shape Detection API.
* Deliver consistent output across all platforms. This is very difficult, unless we implement a common (publicly owned) algorithm. Therefore, we allow the same drawing to yield different outputs. But we want to achieve a common output structure (e.g. what attribute A means, which attributes must be present).



## Existing APIs

Here are some handwriting recognition features offered on various platforms:

### [Microsoft Windows Ink (UWP)](https://docs.microsoft.com/en-us/uwp/api/windows.ui.input.inking)

Windows Ink provides support for collecting, showing, and managing drawings, it supports both shape and text. For text recognition, there classes are used:

*   `InkPoint` and `InkStroke` store all digital ink information.
*   `InkPoint` stores: position, timestamp, pressure, and tilting angles.
*   `InkStroke` stores a list of `InkPoint`s.
*   `InkAnalyzer` recognizes texts from `InkStroke`, supports all Windows languages.
*   `InkAnalysisInkWord` includes alternative texts (candidates), bounding boxes of the text, and the composing strokes.

The handwriting recognition usage looks like the following, the text recognition result is traversed based on words. Applications can use other helper classes to streamline stroke management (e.g. `InkPresenter`).

```C#
InkStroke stroke = (new InkStrokeBuilder()).CreateStroke([
    new InkPoint(Position(x, y), pressure, tiltX, tiltY, timestamp),
    ...   // more points
]);

// Collect some strokes.
InkStroke strokes = [stroke, ...];

InkAnalyzer analyzer = new InkAnalyzer();
analyzer.AddDataForStrokes([inkStroke, ...]);
await analyzer.AnalyzeAsync();

// The result can be traversed based on words.
IReadOnlyList<IInkAnalysisNode> drawings = inkAnalyzer.AnalysisRoot.FindNodes(InkAnalysisNodeKind.InkWord);
for each (IInkAnalysisNode node in drawings) {
    InkAnalysisInkWord word = (InkAnalysisInkWord) word;
    word.RecognizedText;      // string, the recognized text
    word.TextAlternatives;    // list of string, alternative texts
}
```


### [Apple PencilKit](https://developer.apple.com/documentation/pencilkit)

PencilKit provides support for creating and managing ink drawings, key classes are:

*   `PKStrokePoint`, `PKStroke` and `PKDrawing` store digital ink information.
*   `PKStrokePoint` stores position, timestamp, force (pressure), tilting angles, and altitude.
*   `PKStroke` can be constructed by PKStrokePoints. It interpolates the points to form a smooth stroke path.
*   `Ink` represents the styling of the stroke (i.e. width, color, type).

Apple hasn't disclosed a public API for online handwriting recognition. But iPadOS 14 has the capability to convert handwriting to text.


### [MyScript](https://developer.myscript.com/)

MyScript is a commercial cross-platform ink SDK. It runs on Windows, iOS, Android, and Web. It requires a subscription to use, and can perform on-device recognition or acts as a cloud service.

MyScript doesn't use the point abstraction. Instead, a stroke is represented as an object which stores separate lists for position, timestamp and pressure (all lists are of the same length).

In the simplest form, MyScript takes an input in the following form:

```JavaScript
const input = {
  width: 400,
  height: 200,
  strokeGroups: {
    strokes: {
        id: "strokeId",
        x: [100, 103, 107, 109, ...],    // x coordinate
        y: [ 50,  55,  59,  63, ...],    // y coordinate
        p: [0.5, 0.4, 0.7, 0.3, ...],    // pressure
        t: [  0,  66, 132, 198, ...]     // timestamp
    }
  },
  configuration: { /* how to perform recognition */ }
}
```

The recognition result is returned as a JSON in their MyScript JIIX format (JSON Interactive Ink eXchange), it looks like this:

```JavaScript
{
  type: "Text",
  "bounding-box": { ... },
  label: "Hello world!",    // Text as a string
  words: [ ... ],           // Information on individual words
  id: "MainBlock"
}
```

MyScript also provides helper classes and SDKs to manage stroke capture. Applications can create a DOM element as a drawing area, and let MyScript SDK handles drawing capture, rendering, and recognition.

## Proposed Usage Example

### Query Feature Support

Handwriting recognizers on different platforms have different features. Web applications can query their feature support and decide if the API is suitable for their use case.

```JavaScript
// The list of features to detect.
await navigator.queryHandwritingRecognizerSupport({
  'languages': ['en', 'zh-CN']  // A list of languages
  'alternatives': true          // Can be any value
  'unsupportedFeature': true    // Can be any value
})

// For each query:
//   - If the recognizer supports its feature, returns true.
//   - If the recognizer supports its feature, but the provided
//     parameters aren't supported, return false.
//     For example when proving a wrong language tag.
//   - If the recognizer doesn't support its feature, the feature
//     name is not included in the return value.
//
// => {
//   languages: true,  // The recognizer supports both en and zh-CN
//   alternatives: true,
//   // Unsupported features are not included in the return value
// }
```

To mitigate passive fingerprinting, `queryHandwritingRecognizerSupport` may throw an Error if the website issues too many queries (e.g. when trying to enumerate all supported languages). The browser may show a permission prompt and ask if user grants access to unrestricted handwriting recognition features (before throwing the Error).


### Perform Recognition

```JavaScript
// Model constraints determine the handwriting recognition model
// used to create the recognizer.
const modelConstraints = {
  languages: ['zh-CN', 'en'],  // Languages, in order of precedence
}

// Create a handwriting recognizer.
const recognizer = await navigator.createHandwritingRecognizer(modelConstraints)

// Optional hints to improve recognizer's performance on a drawing
const optionalHints = {
  recognitionType: 'text',   // The type of content to be recognized
  inputType: 'mouse',        // Alternatively, “touch” or “pen”
  textContext: 'Hello, ',    // The text before the first stroke
  alternatives: 3,           // How many alternative results to return
}

// Start a new drawing.
// Multiple drawings can be created from a single recognizer. They
// can have different hints.
const drawing = recognizer.startDrawing(optionalHints)

// Create a new stroke.
const stroke = new HandwritingStroke()

// Add a point.
const point = { x: 84, y: 34, t: 959 }

// The point dictionary is copied, and added to the stroke object.
stroke.addPoint(point)

// Modifying a point added to a stroke has no effect.
point.x = newX    // We don't want this.
stroke.getPoints()[0].x = newX    // We don't want this.

// The point's value remains the same despite of two above assignments.
stroke.getPoints()[0].x === 84    // => true

// We can say it's a copy of dict
// Add a stroke to the drawing.
drawing.addStroke(stroke)

// Add more points to the stroke.
stroke.addPoint({ x: 93, y: 54, t: 1013 })

// Get predictions of the partial drawing.
// This will take into account both points that were added to the stroke.
await drawing.getPrediction()

// The returne value is a list of prediction results, in decreasing
// order of confidence (of that result being correct).
//
// The list is guaranteed to have at least one result.
//
// Each result may contain extra fields, if the recognizer supports
// returning these information. For example, segmentation results.
//
// => [
//   { text: "predicted text", /* extra fields */ },
//   { text: "...", /* extra fields */  }
// ]


// Add a new stroke.
const stroke2 = new HandwritingStroke()
stroke2.addPoint({x: 160, y: 39, t: 1761})
drawing.addStroke(stroke2)

// Get all strokes. Return a list of previously added HandwritingStroke object
// references, in the same order as they were added.
drawing.getStrokes()
// => [stroke, stroke2]

// Delete a previous stroke.
drawing.removeStroke(stroke)

// Get a new prediction.
await drawing.getPrediction()

// Complete the drawing and free up resources. Subsequent calls on the drawing
// object will throw an error.
await drawing.finish()
```


## Proposed API

### Feature Detection

Handwriting recognition can be implemented in different ways. We expect different implementations to different sets of features (and hints).

The `queryHandwritingRecognizerSupport` method allows Web developers to query implementation-specific features, decide whether handwriting recognition is supported, and whether it is suitable for their use case.

This method takes the query array, where each array element is a feature name (query). This method returns a dictionary, whose keys are the provided feature names, and the values are some information about the feature.

Conventionally, feature name is the same as the key name used in method arguments or outputs. If a feature name is not supported, the value (for that key-value pair) is `null`.

For example, these feature names are supported in this proposal:

* `graphemeSet`
* `alternatives`
* `textContext`
* `languages`
* `recognitionTypes`
* `inputTypes`
* `segmentationResult`

### Coordinates

This API follows the usual Web coordinate system. A coordinate is represented by its `x` and `y` attribute.  They represent the horizontal and vertical distance from the top-left corner. The top-left corner coordinate is `(x=0, y=0)`.

### Time

Time (`t` attribute of an ink point) is measured as a number of milliseconds elapsed since some reference time point (e.g. `Date.now()`).

### The ink and drawing model

An **ink point** is represented by a JavaScript object that has _at least_ three attributes:

*   `x`: number, the horizontal component of its coordinate.
*   `y`: number, the vertical component of its coordinate.
*   `t`: number, a timestamp, the number of milliseconds elapsed since the starting time (e.g. when the first ink point of the drawing is captured).

An ink point can have extra attributes. For example, pen-tip pressure and angles. These attributes are optional. If they are available. recognizers may use them to improve accuracy.

The recognizer will function with only the required attributes.

An **ink stroke** is represented by a JavaScript `HandwritingStroke` object, created by the client using the constructor. The ink points in a stroke _should_ have ascending `t` attributes.

Points are added by calling `addPoint` method, which deep copies the provided point dictionary.

A **drawing** is represented by a JavaScript `HandwritingDrawing` object, created by calling recognizer's startDrawing method. The ink strokes _should_ be in an ascending order based on their start time (the `t` attribute of their first point).

*   Strokes are added by calling `addStroke` method, which takes a `HandwritingStroke` object, and stores a reference to it.
*   Strokes can be deleted by calling `removeStroke` method, which takes a previously added HandwritingStroke object.

### Model constraints

Model constraints are used to determine and initialize the underlying handwriting recognition algorithm. They describes a set of constraints that the created recognizer must satisfy.

Model constraints can be empty. In this case, the browser is free to choose a default (e.g. based on `navigator.languages`).

`createHandwritingRecognizer` throws an `Error` if:
* The provided constraints can't be satisfied (e.g. the browser has no model for the chosen language)
* There isn't enough resource to initialize a recognizer (e.g. out of memory)

We propose the following model option:

* `languages`: A list of languages that the recognizer should attempt to recognize. They are identified by IETF BCP 47 language tags (e.g. `en`, `zh-CN`, `zh-Hans`). See [Language Handling](#language-handling) for determining fallbacks if the provided tag is not supported.


### Recognition hints

The recognizer _may_ accept hints to improve accuracy for each drawing.

Clients can optionally provide hints (or some combinations) when creating a `HandwritingRecognizer` object. Providing unsupported hints has no effect.

We propose the following hint attributes:

* `graphemeSet`: A list of strings, each string represents a grapheme (a user-perceived unit of the orthography) that is most likely to be written. Note, this is a hint, it doesn't guarantee that the recognizer only returns the characters specified here. Clients need to process the result if they want to filter-out unwanted characters.
* `recognitionType`: A string, the type of content to be recognized. The recognizer may use these to better rank the recognition results. It supports:
    * `email`: an email address
    * `number`: a decimal number
    * `text`: free form text in typical writing prose. It hints the input drawing represents real words. For example, a sentence in everyday speech.
    * `per-character`: treat the handwriting to be made up by individual, unrelated characters, and do not attempt to refine the ranking. For example, serial numbers, license keys.
* `inputType`: A string, identifying how are the strokes captured:
    * `touch`: Input was made with touchscreen
    * `pen`: Input was made with stylus pen
    * `mouse`: Input was made with mouse
* `textContext`: A string, the text that comes before the handwriting. This can be texts that were previously recognized, or were given as the writing context (e.g. "Write your name here:"). This is the linguistic context to help disambiguate the handwriting (e.g. “Hello <span style="text-decoration:underline;">world</span>” vs. “Hello <span style="text-decoration:underline;">word</span>”).
* `alternatives`: A number, the maximum number of alternative predictions.

Hints won't guarantee the result will satisfy these constraints. For example, proving the characters hint won't guarantee the prediction result will only contain these characters.

### The prediction result

A **prediction result** is a JavaScript object. It _must_ contain the text attribute:

*   `text`: string, the texts drawn in the digital ink

A prediction result _may_ contain additional attributes (if the implementation supports their respective features):

*   `segmentationResult`: A list of JavaScript objects, explained in the below [Segmentation Result](#segmentation-result) section.


`getPrediction()` method returns a list of prediction results, in decreasing order of confidence (e.g. the first result is the best prediction).

*   This list must have at least one prediction result.
*   If the recognizer supports returning alternatives, the list may contains multiple results.

For example, an implementation with segmentation support returns

```JavaScript
[
  { text: "best prediction", segmentationResult: [...] },
  { text: "2nd best prediction", segmentationResult: [...] },
  { text: "3rd best prediction", segmentationResult: [...] },
]
```


### Segmentation result
Segmentation result is a mapping from recognized graphemes (user-perceived characters) to their composing strokes and points. This provides per-character editing functionality (e.g. delete this handwriting character in a note taking app).

The segmentation result is a partition of the drawing: every point is attributed to exactly one grapheme.

In JavaScript, segmentation result is represented as a list. Each list item describes a grapheme, its position in the predicted text, and its composing segments. Each segment is represented as a stroke object and its begin-end point indices.

For example, the handwriting "int" produces a segmentation result like this:

<img src="images/segmentation-concept.svg" width="400" alt="Segmentation Concept">

```JavaScript
[
  {
    // The string representation of this grapheme.
    grapheme: "i",

    // The position of this grapheme in the predicted text.
    // predictionResult.text.slice(beginIndex, endIndex) === grapheme
    //
    // If the grapheme spans multiple Unicode code points,
    // `endIndex - beginIndex` is greater than 1.
    beginIndex: 0,
    endIndex: 1,

    // Drawing segments that make up the grapheme.
    segments: [
      {
        strokeIndex: 0,
        beginPointIndex: 0,
        endPointIndex: 30,
      },
      {
        strokeIndex: 1,
        beginPointIndex: 0,
        endPointIndex: 5,
      }
    ],
  },
  {
    grapheme: "n",
    segments: [...],
  },
  ...
]
```

The indices in segmentation results are based on its `getPrediction()` call. If the application modifies the drawing before `getPrediction()` returns, the indices may differ from what's currently in the drawing (e.g. `getStrokes()`).

Whitespaces are not included in the segmentation result, even if they are part of the predicted text.

In most languages, a grapheme (a user-perceived unit of orthography) corresponds to a Unicode grapheme cluster. For example, in Latin alphabet, grapheme "a" corresponds to grapheme cluster `U+0061`; "ä" corresponds to grapheme cluster `U+0061 U+0308`. In some complex scripts, some graphemes are composed of multiple grapheme clusters. For example, in Balinese, ᬓ᭄ᬱᭀ is made up of two Unicode grapheme clusters: `U+1B13` and `U+1B44 U+1B31 U+1B40`.


## Design Questions
### Why not use Web Assembly?
Web Assembly would not allow the use of more advanced proprietary handwriting libraries (e.g. those available on the operating system). Web developers also need to manage distribution (and update) of such libraries (might take several megabytes for each update).

Web API can do the same task more efficiently (better models, zero distribution cost, faster computation). This topic was previously discussed in Shape Detection API and Text-to-Speech API.


### Why not use Shape Detection?
Handwriting (in this proposal) includes temporal information (how the shape is drawn, pixel by pixel). We believe this additional temporal information distinguishes handwriting recognition from shape detection.

If we take out the temporal information, the task becomes optical character recognition (given a photo of written characters). This is a different task, and indeed fits within the scope of shape detection.

### Grapheme vs. Unicode code points
Grapheme is the minimal unit used in writing. It represents visual shape. On the other hand, Unicode code points are a computer's internal representation. It represents meaning. The two concepts aren't fully correlated.

Unicode combining marks are represented as a single code point. They are used to modify other characters, but not by themselves. This creates a problem when we need to distinguish between shape and meaning. For example, letter a (U+0061) and grave accent combining mark (U+0300) combines to à. Letter न (U+0928) and combining mark  ि (U+093F) combines to letter नि.

Handwriting recognition concerns with shape (input) and meaning (output). It's important to distinguish between those two. For example, when requesting to recognize only certain characters, graphemes should be used.

### Ranking vs. Score
It's very common to use a score for assessing alternative texts. This is commonly implemented in machine learning algorithms. However, it is not a good idea for the Web.

We expect different browser vendors to offer varying recognizer implementations, this will inevitably lead to the score being incomparable.

Because the score is an implementation detail of machine learning models, the meaning of score changes if the model changes. Therefore, scores are not comparable unless everyone uses the same model.

Thus, we choose to use ranking instead of score. This gives some indication on which alternative is better. This avoids the scenario where web developers misunderstand the score's implication and try to compare scores across different libraries, or filtering results based on it.


## Considerations

### Fingerprinting

The fingerprint vector comes from two parts: feature detection and recognizer implementation.

The amount of information (entropy) exposed depends on user agent's implementation. We believe there isn't a one-size-fits-all solution, and recommend the user agents decide whether privacy protections (e.g. permission prompts) are necessary for their users.

**Feature detection** could expose information about:
* User's language (or installed handwriting recognition models). This is also available in `navigator.languages`.
* The recognizer implementation being used, by summarizing the set of supported features. This might lead to some conclusions about the operating system and its version.

This can be mitigated by [privacy budget](https://github.com/bslassey/privacy-budget). The user agent can choose to throw errors (or return less accurate informations), if the number of queries to `queryHandwritingRecognizerSupport` is excessive (e.g. querying dozens of languages in one browsing session).

**Recognizer implementation** might expose information about the operating system, the device, or the user's habit. This largely depends on the recognizer technology being used.

Below are some types of recognizer implementations, and their associated risks:

* No recognizer support: No fingerprint risk.

* Cloud-based recognizer: Little to no risk. The website might be able to detect the cloud service being used (by comparing the prediction result against the result obtained by calling the cloud service directly). No extra information is revealed about the user or their device.

* Stateless models (most common): Output of such models is entirely dependent on the input drawing and the model itself.

  * Models that can only updated with the browser (e.g. entirely implemented within the browser): The website can learn about the browser's version, which can otherwise be detected by other means.

  * Models that can be updated outside of browser (e.g. implemented by calling an operating system API), depending on the scenario, the website can learn:

    * OS version, if the models are updated along with every OS update.
    * OS version range, if the models are updated with some but not all OS updates.
    * A specific update patch revision, if the models are updated out-of-band or on-demand.

  * Note, if the models utilizes hardware accelerators (e.g. GPU). The result might reveal information about particular hardwares.

* Stateful / Online learning models (worst hypothetical case): These models can learn based on the previous usages. For example, an OS recognizer that adjusts its output based on user's IME habits. These models can reveal large amount of information about the user, and poses a huge risk. \
However, we aren't aware of any recognizer implementations that falls within this type. But we recommend using privacy protection for these models, or use a fresh / clean state for each session.

**Cost of fingerprinting**: the fingerprinting solution need to craft and curate a set of handwriting drawings (adversarial samples) to exploit differences across models. The cost of generating these samples may be high, but it's safe to assume a motivated party can obtain such samples.


### Language Handling

For querying for supported languages, the implementation should only return the language tags that have dedicated (or fine-tuned) models. For example, if the implementation only has a generic English language model, it should only include "en" in supportedLanguages, even if this model works for its language variants (e.g. en-US).

Web developers may provide subtags (e.g. region and script). The implementation should interpret them, and choose fallbacks if necessary. In general:

* If the provided language tag doesn't match any recognizer, remove the last subtag until there is a match. For example, `"zh-Hans-CN"` -> `"zh-Hans"` -> `"zh"`.
* If the browser can't match any recognizer (after the above fallbacks), `createHandwritingRecognizer` method throws an Error.

If language model options aren't provided, this implementation should try to pick a model based on `navigator.languages` or user's input methods. If this fails to match any recognizer, `createHandwritingRecognizer` method throws an Error.

### Model Constraints vs. Model Identifier
In the current design, `createHandwritingRecognizer` takes model constraints, and let the browser to determine the exact recognition models being used.

This offload some work for Web developers. Developers don't have to write logics to pick a specific model (e.g. parse language tags, decide fallbacks, etc.) from a list of supported models.

At early design stages, we are unsure if requiring web applications to explicitly pick a model is a good idea (or ergonomic for web developers). We'd need developer feedbacks to better decide this.

The current design (of using model constraints) has room for a future addition `modelIdentifier` field. This would work similarly to Web Speech Synthesis API, where the web application explicitly chooses a voice.

* `queryHandwritingRecognizerSupport` would have a `supportedModels` query. It returns a JavaScript object describing the characteristics of the available models.
    ````JavaScript
    {
      identifier: 'zh-Hani',
      languages: ['zh'],
      offlineService: true,
      ... /* Attributes may vary based on platform */
    }
    ````
* `modelIdentifier` is mutually exclusive with other model constraints. If it's used with any other constraint, `createHandwritingRecognizer` will throw an Error.

### Interoperability

This API aims to find the "greatest common divisor" of major platforms. The input to handwriting recognizer function should be easily translated to the input required by the underlying handwriting recognition API (available on the operating system).

Browsers can implement their own handwriting recognition algorithm, if there are no native ones, or they decides to provide cross-platform consistency.


### Incremental recognition

The implementation may keep states about the texts already recognized and perform incremental recognition. The implementation may use heuristics to determine which strokes require recognition (e.g. the strokes close to the newly added ones since last recognition).

## Alternative API Design

Alternatively, the API could take in a complete drawing, recognizes the text, and returns the result. This is simpler to use, the disadvantage being:

* It can't support incremental recognition. Each time the recognizer is called, the complete drawing is processed, even if part of the drawing may already be processed previously.
* It causes more overhead if the recognizer is repeatedly called on incremental drawings (i.e. when new strokes are added to an existing drawing). The information in the existing drawing will be processed again, even if they were processed in previous recognition requests.

```JavaScript
// Create a handwriting recognizer.

const recognizer = await navigator.createHandwritingRecognizer({
  languages: ['zh-CN', 'en'],
  // Optionally, provide more hints.
})

// Collect a drawing (list of strokes).

const drawing = [
  handwritingStroke,
  ...
]

// Optionally, include some hints.

const optionalHints = {
  inputType: 'mouse',   // Alternatively, “touch” or “pen”
  textContext: 'Hello, ',   // The text before the first stroke
  alternatives: 5,
}

// Recognize the text.
const result = recognizer.recognize(drawing, optionalHints)
// => The result list:
//   [
//     { text: "best prediction" },
//     { text: "2nd best prediction" },
//     ...
//   ]
```
