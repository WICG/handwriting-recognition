Web IDL for Handwriting Recognition API
===

## Entry point
```webidl
[SecureContext]
partial interface Navigator {
  [CallWith=ScriptState, RaisesException]
  Promise<HandwritingRecognizer>
      createHandwritingRecognizer(HandwritingModelConstraints constraints);

  // V2 feature query.
  [CallWith=ScriptState, RaisesException]
  Promise<HandwritingRecognizerQueryResult?>
      queryHandwritingRecognizer(HandwritingModelConstraints constraints);
};

dictionary HandwritingModelConstraints {
  required sequence<DOMString> languages;
};

dictionaty HandwritingRecognizerQueryResult {
  bool textAlternatives;
  bool textSegmentation;
  HandwritingHintsQueryResult hints;
};

dictionaty HandwritingHintsQueryResult {
  sequence<DOMString> recognitionType;
  sequence<DOMString> inputType;
  bool textContext;
  bool alternatives;
};
```

## Recognizer
```webidl
[SecureContext]
interface HandwritingRecognizer {
  [CallWith=ScriptState, RaisesException]
  HandwritingDrawing startDrawing(optional HandwritingHints hints = {});

  [RaisesException] void finish();
};

dictionary HandwritingHints {
  sequence<DOMString> graphemeSet = [];
  DOMString recognitionType = "text";
  DOMString inputType = "mouse";
  DOMString textContext = "";
  unsigned long alternatives = 3;
};
```

## Point and Stroke
```webidl
[SecureContext, Exposed=Window]
interface HandwritingStroke {
  constructor();
  void addPoint(HandwritingPoint point);
  sequence<HandwritingPoint> getPoints();
  void clear();
};

dictionary HandwritingPoint {
  double x;
  double y;

  // Timestamp in milliseconds since the start of the current drawing.
  DOMTimeStamp t;
};
```

## Drawing
```webidl
[SecureContext]
interface HandwritingDrawing {
  void addStroke(HandwritingStroke stroke);
  void removeStroke(HandwritingStroke stroke);
  void clear();
  sequence<HandwritingStroke> getStrokes();

  [CallWith=ScriptState]
  Promise<sequence<HandwritingPrediction>> getPrediction();
};

dictionary HandwritingPrediction {
  required DOMString text;
  sequence<HandwritingSegment> segmentationResult;
};

dictionary HandwritingSegment {
  required DOMString grapheme;
  required unsigned long beginIndex;
  required unsigned long endIndex;
  required sequence<HandwritingDrawingSegment> drawingSegments;
};

dictionary HandwritingDrawingSegment {
  required unsigned long strokeIndex;
  required unsigned long beginPointIndex;
  required unsigned long endPointIndex;
};
```

## Previous API
### Feature Query
```webidl
// V1 feature query, replaced by queryHandwritingRecognitionLanguage.
[CallWith=ScriptState, RaisesException]
partial interface Navigator {
  Promise<HandwritingFeatureQueryResult>
      queryHandwritingRecognizerSupport(HandwritingFeatureQuery query);
}

dictionary HandwritingFeatureQuery {
  sequence<DOMString> languages;
  any alternatives;
  any segmentationResult;
};

dictionary HandwritingFeatureQueryResult {
  boolean languages;
  boolean alternatives;
  boolean segmentationResult;
};
```