Jo historical code replace hua tha, use delete nahi kiya; alag legacy directories mein safely preserve kiya hai.
Registry-based form routing add ki, jisse har form type ko uske correct processing flow mein bheja jaata hai. First Assessment ke related sections ko groups mein process kiya jaata hai.
First Assessment processing ko optimize karke sirf do parallel AI model calls mein complete kiya.
Limited workers, processing timeouts, rate admission aur request IDs add kiye, jisse server overload aur mixed responses control hote hain.
Safer model clients aur provider fallback add kiya, jisse primary AI model fail hone par configured fallback use ho sake.
Numeric values, blank fields, zero, RPE aur exercise load ke liye validation add ki.
General, left aur right measurement values ki unnecessary duplication ko prevent kiya.
Current measurement ko galti se clinical goal banne se prevent kiya.
Structured clarification suggestions add kiye, jo transcript ke actual evidence ke against verify hote hain.
AI response late aane par clinician ke manual changes overwrite na hon, uske liye protection add ki.
Reset functionality fix ki, jisse prefetched form values, renderer state aur voice state properly clear hote hain.
Manual aur automatic GraphQL saves ko ek common save flow mein unify kiya.
Same appointment ke saves ko proper order mein process karna aur successful save confirmation check karna add kiya.
Strict GraphQL field normalization aur alias mapping add ki, jaise name → testName, details → comments aur description → conclusion.
Existing saved report ko reload karna aur Report.records se form ko prefill karna add kiya.
Frontend code se active hardcoded API/WebSocket key fallbacks remove kiye.
Frontend ke liye Docker aur Nginx packaging, SPA routing aur health checks add kiye.
Main corrected flows ke liye regression tests add kiye, taaki future changes se fixed issues dobara na aayein.
