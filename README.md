CallLLM (Email Classification, multi-prompt)
Overview

This workflow iterates over a dictionary of prompts (key = request id, value = text), optionally pairs each prompt with an attachment, calls an LLM endpoint (Hugging Face by default, Local LLM if selected), parses the JSON response, and writes results into an output dictionary keyed by request id.

Inputs / Outputs

In Arguments

in_PromptPrefix : String
Example: Please determine the criticality of this email:

in_dicPromptsText : Dictionary(Of String, String)
Example:

New Dictionary(Of String, String) From {
  {"Email1","Tell me the current wather in San Jose Costa Rica."},
  {"Email2","This email is classified as a request."},
  {"Email3","This email is classified as feedback."}
}


in_dicPromptsAttachments : Dictionary(Of String, IO.FileInfo) (optional)
Attachment per request id (used if present).

in_LLMSelected : String
"HF" (default) or "Local".

in_EndPointURL : String
e.g. https://api-inference.huggingface.co/models/google/flan-t5-base

in_ContentType : String
e.g. application/json

in_Authorization : String
e.g. Bearer hf_******

in_PromptRequestBody : String
JSON payload sent to the endpoint (can be replaced at runtime per prompt).

in_PromptResultKeySource : String
Key to look for in the LLM response (e.g., "Response" or "generated_text").

in_ResponseHeaders : Dictionary(Of String,String) (optional)

Out Arguments

out_dicResponses : Dictionary(Of String, String)
Map of requestId → LLM response text.

Flowchart (mirrors the XAML)
Start
  │
  ▼
[Log] "Started prompt"
  │
  ▼
For Each (currentPromptText in in_dicPromptsText)           // Type: KeyValuePair(Of String,String)
  │
  ├─ Assign strRequestID   ← currentPromptText.Key
  ├─ Assign strFullPrompt  ← in_PromptPrefix + ": " + currentPromptText.Value
  │
  ├─ (Optional) Try to get attachment:
  │     fileIO ← in_dicPromptsAttachments(strRequestID)
  │   Catch: log "No Attachments found"; fileIO ← Nothing
  │
  └─ Flowchart: Choose LLM target
        ┌────────────────────────────────────────────────────┐
        │ Decision: in_LLMSelected = "Local" ?               │
        ├───────────────┬────────────────────────────────────┤
        │ True          │ False                              │
        ▼               ▼                                    │
    [HTTP] Local LLM    [HTTP] Hugging Face (default)        │
    Method=POST         Method=POST                           │
    URL=in_EndPointURL  URL=in_EndPointURL                    │
    Headers:            Headers:                              │
      - Authorization     - Authorization                     │
      - Content-Type      - Content-Type                      │
    Body=in_PromptRequestBody                                  │
    → jsResponse (String), intStatusCode (Int32)               │
        └────────────────────────────────────────────────────┘
  │
  ├─ Deserialize JSON (JObject) ← jsResponse  → jsobj
  ├─ [Log] "Status code: " + intStatusCode
  │
  ├─ If intStatusCode = 200 Then
  │     For Each (kvp in jsobj)    // kvp: KeyValuePair(Of String, JToken)
  │       If kvp.ToString.Contains(in_PromptResultKeySource) Then
  │         strResponse ← kvp.Value.ToString
  │         [Log] "AI Response: " + strResponse
  │         out_dicResponses(strRequestID) ← strResponse
  │       End If
  │     Next
  │   Else
  │     [Log] "Failed calling LLM … StatusCode …"
  │     Throw BusinessRuleException("Failed calling LLM …")
  │   End If
  │
  └─ Next prompt
  │
End

Notes & gotchas

Status handling: Non-200 responses log and throw a BusinessRuleException.

Key extraction: The workflow scans the top-level pairs of the JObject and matches any pair whose string contains in_PromptResultKeySource.

For Hugging Face text-gen models that return an array with generated_text, consider setting in_PromptResultKeySource = "generated_text" or adjusting the body/parse to suit your model’s format.

Attachments: The attachment fetch is wrapped in a Try/Catch and currently commented out in your XAML (under Comment Out). When needed, un-comment and include the file’s text/base64 in your prompt builder.

Endpoint/body: in_PromptRequestBody is passed as-is to the HTTP activity—swap it per prompt if required (e.g., inject strFullPrompt before the call).

Safety: Ensure out_dicResponses is initialized by the caller before invocation:

If out_dicResponses Is Nothing Then
    out_dicResponses = New Dictionary(Of String,String)
End If

Example defaults (exactly as in your XAML attributes)

in_PromptPrefix = Please determine the criticality of this email:

in_dicPromptsText = (three sample emails)

in_EndPointURL = https://api-inference.huggingface.co/models/google/flan-t5-base

in_ContentType = application/json

in_Authorization = Bearer hf_******

in_PromptRequestBody (sample translate prompt JSON)

in_PromptResultKeySource = Response

in_LLMSelected = HF