Responses to the [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/) for the [Handwriting Recognition API](https://github.com/WICG/handwriting-recognition).

### 1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

This API allows an origin to query which handwriting recognition models are available in the host environment. This is needed to determine language support for handwriting recognition, and decide if handwriting recognition should be performed with this API. For example, if the host environment only supports English handwriting, an origin targeting Chinese handwriting shouldn't use this API to recognize texts. 

This allows an origin to detect user’s installed languages, which may allow additional fingerprinting depending on whether this information is already available to sites (e.g. if browser limits model availability based on browser language settings).

This API also allows an origin to query or know certain characteristics about the underlying recognition algorithm (which is likely implemented as machine learning models). An origin can summarize these characteristics by inspecting recognition results or performing feature detection with the methods proposed in this API. The recognition result is the goal of this API, feature detection is for origins to determine if handwriting recognition should be used (e.g. does browser's handwriting recognizer meet the origin's need).

### 2. Is this specification exposing the minimum amount of information necessary to power the feature
Yes.

### 3. How does this specification deal with personal information or personally-identifiable information or information derived thereof
This API doesn't expose new information about the user.

### 4. How does this specification deal with sensitive information
No special treatment.

### 5. Does this specification introduce new state for an origin that persists across browsing sessions
No.

### 6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin
The origin can make deductions from the information returned in this API. For example, certain handwriting languages models are only available on certain operating systems.

### 7. Does this specification allow an origin access to sensors on a user’s device
No.

### 8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.
Information about the handwriting recognition models available on the host environment. This can be used to make deductions about the host environment. Similar deductions can be made from other existing Web APIs (e.g. User-Agent).

### 9. Does this specification enable new script execution/loading mechanisms
No.

### 10. Does this specification allow an origin to access other devices
No.

### 11. Does this specification allow an origin some measure of control over a user agent’s native UI
No.

### 12. What temporary identifiers might this this specification create or expose to the web
No.

### 13. How does this specification distinguish between behavior in first-party and third-party contexts
No. This API behaves in the same way when used in first-party and third-party contexts.

### 14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?
This API acts in the same way as the non-incognito mode.

### 15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?
There are no known security or privacy impacts of this feature.

This API does expose a new fingerprinting vector (see [fingerprinting section](https://github.com/WICG/handwriting-recognition/blob/main/explainer.md#fingerprinting) in the explainer]. But other fingerprinting techniques (without this API) already offer similar (or better) fingerprinting accuracy.

### 16. Does this specification allow downgrading default security characteristics
No.

### 17. What should this questionnaire have asked
No more questions.
