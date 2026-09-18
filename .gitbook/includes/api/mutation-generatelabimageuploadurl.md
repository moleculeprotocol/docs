Issues a single-use presigned PUT URL for a LabNft display image, restricted to the OCL admin. `contentType` must be one of `image/jpeg`, `image/png`, `image/webp`, `image/gif` or `image/svg+xml`, the generated key's extension is derived from it, and the client must upload with the matching `Content-Type` header. The uploaded object's type and size are validated again before the lab's `image` is updated, and other content types fail with `UNSUPPORTED_IMAGE_TYPE`.

Returns [`GenerateLabImageUploadUrlResult!`](/api-reference/types.md#generatelabimageuploadurlresult).

| Name | Type | Description |
| --- | --- | --- |
| `oclId` | `String!` | 32-byte OCL id of the lab, `0x` plus 64 hex, case-insensitive. |
| `contentType` | `String!` | MIME type of the image (case-insensitive): image/jpeg, image/png, image/webp, image/gif or image/svg+xml. |
