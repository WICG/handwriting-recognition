Web IDL for Handwriting Recognition API
===

## Entry point
```webidl
[SecureContext]
partial interface Navigator {
  [CallWith=ScriptState, RaisesException]
  Promise<HandwritingRecognizer>
      createHandwritingRecognizer(HandwritingModelConstraint constraint);

  [CallWith=ScriptState, RaisesException]
  Promise<HandwritingFeatureQueryResult>
      queryHandwritingRecognizerSupport(HandwritingFeatureQuery query);
};

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

dictionary HandwritingModelConstraint {
  required sequence<DOMString> languages;
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

  // The amount of time since the start of the current drawing.
  DOMHighResTimestamp t;
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
